# Capítulo 8: Armazenamento e Persistência

*"Dados que não sobrevivem a um reboot não são dados — são uma sugestão."*

---

## Quando Tudo é Efêmero, Onde Ficam Seus Dados?

Em um servidor Linux, o armazenamento parece permanente. Você particiona um disco com `fdisk`, formata com `mkfs.ext4`, monta em `/data`, e adiciona uma entrada no `/etc/fstab` para que ele volte após um reboot. Seus arquivos estão ali no metal. Você pode fazer `ls /data` e vê-los. Pode fazer `du -sh /data` e saber exatamente quanto espaço está usando. A relação entre seus dados e o disco é direta, física e óbvia.

Kubernetes quebra essa relação — deliberadamente.

Containers são efêmeros por design. Quando um pod morre (e pods morrem o tempo todo — crashes, reagendamentos, rolling updates), seu sistema de arquivos desaparece junto. Cada arquivo que o container escreveu, cada entrada de log, cada cálculo temporário — tudo perdido. É como se cada processo no seu servidor Linux rodasse em um mount tmpfs que fosse apagado em cada reboot.

Para aplicações stateless (um frontend web, um API gateway), isso é aceitável. O container inicia, serve requisições e morre sem remorso. Mas no momento em que você tem um banco de dados, um serviço de upload de arquivos, ou qualquer coisa que precisa lembrar algo entre reinícios, você precisa de persistência. Você precisa de armazenamento que sobreviva ao pod.

Este capítulo é sobre como o Kubernetes resolve esse problema — e como os conceitos se mapeiam de volta para a pilha de armazenamento Linux que você já conhece.

---

## O Problema da Efemeridade

Vamos tornar isso concreto. Quando você executa um container, ele recebe uma camada gravável por cima das camadas somente-leitura da imagem (esse é o sistema de arquivos overlay do Capítulo 2). Qualquer arquivo que o container cria vive nessa camada gravável.

O problema: essa camada gravável está atrelada ao ciclo de vida do container. Delete o pod, e a camada é coletada pelo garbage collector. Reagende o pod para um node diferente, e ele começa do zero com uma camada gravável limpa.

Isso é como executar cada serviço no seu servidor Linux com seu diretório de trabalho em um mount `tmpfs`:

```bash
mount -t tmpfs tmpfs /var/lib/myapp
systemctl start myapp
# myapp grava dados em /var/lib/myapp...
systemctl stop myapp
# Dados perdidos. Para sempre.
```

Ninguém faria isso em um servidor Linux (pelo menos não para dados que importam). Mas no Kubernetes, esse é o comportamento padrão. Você precisa explicitamente optar pela persistência.

---

## Tipos de Volume

O Kubernetes fornece vários tipos de volume, cada um resolvendo um problema de armazenamento diferente. Vamos percorrê-los do mais simples ao mais poderoso.

### emptyDir — Armazenamento Temporário Compartilhado

Um volume `emptyDir` é criado quando um pod é atribuído a um node e existe enquanto o pod rodar naquele node. Quando o pod é removido, os dados no `emptyDir` são deletados permanentemente.

Pense nele como um diretório `/tmp` compartilhado entre todos os containers no mesmo pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-storage
spec:
  containers:
  - name: writer
    image: busybox:1.36
    command: ["sh", "-c", "echo 'hello from writer' > /shared/message.txt && sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  - name: reader
    image: busybox:1.36
    command: ["sh", "-c", "sleep 5 && cat /shared/message.txt && sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  volumes:
  - name: shared-data
    emptyDir: {}
```

Dois containers, um volume. O writer coloca um arquivo em `/shared`, e o reader pode vê-lo. Isso é exatamente como dois processos em uma máquina Linux compartilhando `/tmp`.

Você também pode usar memória ao invés de disco como backing de um `emptyDir`:

```yaml
volumes:
- name: cache
  emptyDir:
    medium: Memory
    sizeLimit: 64Mi
```

Isso cria um mount tmpfs — rápido, mas conta contra o limite de memória do container. Útil para espaço temporário, caches, ou sockets compartilhados entre containers sidecar.

### hostPath — Acesso Direto ao Filesystem do Host

Um volume `hostPath` monta um arquivo ou diretório do filesystem do node host dentro de um pod. É o equivalente Kubernetes de um bind mount:

```yaml
volumes:
- name: host-data
  hostPath:
    path: /var/log/host
    type: Directory
```

Em um servidor Linux, isso é como:

```bash
mount --bind /var/log/host /container/logs
```

**Aviso:** `hostPath` é perigoso em produção. Seu pod agora está acoplado ao filesystem de um node específico. Se o pod for reagendado para um node diferente, os dados não estarão lá. Também cria riscos de segurança — um container com acesso ao filesystem do host pode causar muito estrago. Use-o para desenvolvimento e DaemonSets específicos de nível de sistema (como coletores de log), mas nunca para dados de aplicação em produção.

### PersistentVolume (PV) — Armazenamento no Nível do Cluster

Um PersistentVolume é um pedaço de armazenamento no cluster que foi provisionado por um administrador ou provisionado dinamicamente por um StorageClass. Pense nele como uma partição de disco que o admin do cluster preparou e disponibilizou:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
```

Isso é como o admin executando `fdisk`, `mkfs`, e adicionando a entrada no `/etc/fstab` — preparando armazenamento para uso, mas ainda não montando em nenhuma aplicação específica.

Um PV existe independentemente de qualquer pod. É um recurso no nível do cluster (não namespaced), assim como um disco físico existe independentemente de qualquer processo que possa usá-lo.

### PersistentVolumeClaim (PVC) — Solicitando Armazenamento

Um PersistentVolumeClaim é como um pod solicita armazenamento. É como um usuário dizendo "preciso de 1GB de armazenamento leitura-escrita" sem se importar com qual disco específico o fornece:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

O Kubernetes olha os PVs disponíveis, encontra um que satisfaz a solicitação (capacidade suficiente, modo de acesso compatível), e os vincula. Uma vez vinculado, o PVC é exclusivamente atrelado àquele PV.

Um pod então referencia o PVC na sua spec de volume:

```yaml
volumes:
- name: data
  persistentVolumeClaim:
    claimName: my-pvc
```

A separação é intencional. O admin gerencia PVs (o "qual armazenamento existe"). O desenvolvedor gerencia PVCs (o "qual armazenamento eu preciso"). O pod apenas diz "me dê o armazenamento que o PVC prometeu." Isso desacopla infraestrutura de preocupações da aplicação.

---

## StorageClasses — Provisionamento Dinâmico

No modelo PV/PVC acima, alguém precisa criar PVs manualmente antes que pods possam solicitá-los. Isso é aceitável para clusters pequenos, mas não escala. Imagine um admin criando manualmente partições de disco toda vez que um desenvolvedor precisa de armazenamento — isso é um gargalo.

StorageClasses resolvem isso habilitando provisionamento dinâmico. Ao invés de pré-criar PVs, você define um StorageClass que descreve *como* criar armazenamento sob demanda:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

Agora quando um PVC referencia este StorageClass, o provisioner automaticamente cria o armazenamento de backing (neste caso, um Azure Premium SSD), cria um PV para ele, e o vincula ao PVC. Sem intervenção do admin.

Isso é como LVM thin provisioning no Linux — você define um pool e uma política, e volumes lógicos são criados sob demanda conforme aplicações os solicitam. Ou pense nisso como a diferença entre o admin particionar discos manualmente versus configurar LVM para alocar automaticamente de um volume group.

A maioria das distribuições Kubernetes vem com um StorageClass padrão. Kind usa `standard` baseado no Rancher local-path-provisioner. Provedores de nuvem possuem os seus (Azure: `managed-csi`, AWS: `gp3`, GKE: `standard-rwo`). Você pode ver o que está disponível com:

```bash
kubectl get storageclass
```

O StorageClass padrão (marcado com `(default)`) é usado quando um PVC não especifica um.

---

## Modos de Acesso

PersistentVolumes suportam diferentes modos de acesso que controlam quantos nodes podem montar o volume simultaneamente:

| Modo de Acesso | Abreviação | Descrição |
|----------------|-----------|-----------|
| ReadWriteOnce | RWO | Montado como leitura-escrita por um único node |
| ReadOnlyMany | ROX | Montado como somente-leitura por muitos nodes |
| ReadWriteMany | RWX | Montado como leitura-escrita por muitos nodes |
| ReadWriteOncePod | RWOP | Montado como leitura-escrita por um único pod (K8s 1.27+) |

Pense nestes como opções de montagem NFS no Linux:

- RWO é como montar uma partição ext4 — apenas uma máquina pode montá-la como leitura-escrita.
- ROX é como `mount -o ro` em NFS — muitas máquinas podem ler, nenhuma pode escrever.
- RWX é como um mount NFS completo de leitura-escrita — muitas máquinas podem ler e escrever.
- RWOP é o mais restritivo — mesmo que múltiplos pods rodem no mesmo node, apenas um pode montá-lo.

Nem todos os backends de armazenamento suportam todos os modos de acesso. Discos locais tipicamente só suportam RWO. Sistemas de arquivos de rede como NFS ou Azure Files suportam RWX. Verifique a documentação do seu driver CSI para saber o que é suportado.

---

## Políticas de Recuperação (Reclaim Policies)

Quando um PVC é deletado, o que acontece com o PV subjacente e seus dados? A política de recuperação decide:

| Política | Comportamento | Analogia Linux |
|----------|--------------|----------------|
| Retain | PV e dados são preservados. Admin deve limpar manualmente. | `umount /data` — a partição e seus arquivos ainda existem |
| Delete | PV e o armazenamento de backing são ambos deletados. | `umount /data && wipefs -a /dev/sdb1` — tudo se foi |

A política `Retain` é mais segura — você não vai perder dados acidentalmente. Mas significa que alguém precisa reclamar manualmente o PV após o PVC ser deletado. A política `Delete` é conveniente para workloads efêmeros mas perigosa se você deletar um PVC por engano.

---

## CSI — Container Storage Interface

CSI é o sistema de plugins que permite ao Kubernetes se comunicar com qualquer backend de armazenamento. É como drivers de dispositivo do Linux, mas para armazenamento. Ao invés do Kubernetes conhecer todos os possíveis sistemas de armazenamento (Azure Disks, AWS EBS, NFS, Ceph, etc.), ele define uma interface padrão que provedores de armazenamento implementam.

Drivers CSI comuns:

| Backend de Armazenamento | Driver CSI |
|--------------------------|------------|
| Azure Disk | disk.csi.azure.com |
| Azure Files | file.csi.azure.com |
| AWS EBS | ebs.csi.aws.com |
| GCE Persistent Disk | pd.csi.storage.gke.io |
| NFS | nfs.csi.k8s.io |
| Ceph | rbd.csi.ceph.com |

Em um servidor Linux, você instala um módulo do kernel para suportar um novo sistema de arquivos (`modprobe nfs`). No Kubernetes, você instala um driver CSI para suportar um novo backend de armazenamento. O princípio é o mesmo — uma arquitetura baseada em plugins que separa o sistema central da implementação de armazenamento.

---

## Volume Snapshots

Volume snapshots são cópias point-in-time dos dados de um PersistentVolume. Se você já usou snapshots LVM no Linux (`lvcreate --snapshot`), o conceito é idêntico.

No Kubernetes, snapshots são gerenciados através de três objetos:

- **VolumeSnapshot** — a solicitação para tirar um snapshot (como executar `lvcreate --snapshot`)
- **VolumeSnapshotContent** — os dados reais do snapshot (como o volume lógico de snapshot LVM)
- **VolumeSnapshotClass** — a política de como snapshots são tirados (como as configurações de snapshot LVM)

Snapshots requerem um driver CSI que os suporte — nem todos suportam. Eles são implementados como CRDs (Custom Resource Definitions), não embutidos na API core do Kubernetes. Esta é uma funcionalidade mais avançada que mencionamos aqui para completude; seu cluster Kind não terá suporte a snapshots pronto para uso.

---

## Comparação Linux ↔ Kubernetes

| Conceito Linux | Equivalente K8s | Notas |
|----------------|-----------------|-------|
| `/etc/fstab` | spec do PersistentVolume | Define armazenamento disponível |
| `mount /dev/sdb1 /data` | volumeMount do PVC | Anexa armazenamento a um pod |
| Volume lógico LVM | PersistentVolume | Armazenamento block/file abstraído |
| `mount -t nfs` | PV com provisioner NFS | Armazenamento conectado via rede |
| tmpfs | emptyDir (medium: Memory) | Armazenamento efêmero em memória |
| bind mount | hostPath | Acesso direto ao filesystem do host |
| LVM thin provisioning | StorageClass + provisionamento dinâmico | Alocação sob demanda |
| `lvcreate --snapshot` | VolumeSnapshot | Cópia point-in-time |
| Opções do `/etc/fstab` (ro, rw) | Modos de Acesso (ROX, RWO, RWX) | Permissões de leitura/escrita |
| `fdisk` + `mkfs` + `mount` | Provisionamento estático de PV | Admin prepara armazenamento manualmente |
| LVM auto-extend | Provisionamento dinâmico via StorageClass | Criação automática de armazenamento sob demanda |

---

> ### 🔀 Onde a Analogia com Linux Quebra
>
> - **Armazenamento é desacoplado de computação.** No Linux, armazenamento é diretamente conectado à máquina — seu SSD está fisicamente dentro (ou conectado via cabo) ao servidor. No Kubernetes, armazenamento é um recurso do cluster que pode ser anexado a qualquer pod em qualquer node, graças a drivers CSI e armazenamento conectado via rede. Você não pensa sobre "qual servidor tem o disco" — você pensa sobre "qual pod precisa dos dados."
>
> - **PVCs têm seu próprio ciclo de vida.** Um PVC não é apenas uma solicitação de mount que desaparece quando você desmonta. É um objeto Kubernetes persistente que existe independentemente de qualquer pod. Deletar um pod não deleta seu PVC (a menos que cascading explícito seja configurado). Isso significa que dados sobrevivem a reinícios de pods, reagendamentos, e até deleção deliberada de pods.
>
> - **Provisionamento dinâmico elimina o workflow do admin.** No Linux, o workflow de armazenamento é `fdisk` → `mkfs` → `mount` → atualizar `/etc/fstab`. Um humano executa cada passo. Com StorageClasses, você pula tudo isso — o provisioner cria o armazenamento de backing, formata-o, e o disponibiliza automaticamente. O desenvolvedor escreve um PVC, e o armazenamento aparece. Isso é fundamentalmente diferente da experiência tradicional de sysadmin.

---

## Laboratório Diagnóstico: Armazenamento Hands-On

### Pré-requisitos

Certifique-se de que você tem um cluster Kind rodando. Se não, crie um:

```bash
kind create cluster --name storage-lab
```

### Lab 1: emptyDir — Armazenamento Temporário Compartilhado

Dois containers em um pod compartilhando um volume `emptyDir`. O writer cria conteúdo, e um servidor web o serve.

Salve como `emptydir-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
  - name: content-writer
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      echo '<html><body><h1>Hello from the sidecar!</h1></body></html>' > /usr/share/nginx/html/index.html
      echo "File written at $(date)" >> /usr/share/nginx/html/index.html
      sleep 3600
    volumeMounts:
    - name: shared-content
      mountPath: /usr/share/nginx/html
  - name: web-server
    image: nginx:1.27
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-content
      mountPath: /usr/share/nginx/html
  volumes:
  - name: shared-content
    emptyDir: {}
```

Aplique e verifique:

```bash
# Criar o pod
kubectl apply -f emptydir-pod.yaml

# Aguardar até estar rodando
kubectl wait --for=condition=Ready pod/emptydir-demo --timeout=60s

# Verificar que o writer criou o arquivo
kubectl exec emptydir-demo -c content-writer -- cat /usr/share/nginx/html/index.html

# Verificar que o servidor web pode servi-lo
kubectl exec emptydir-demo -c web-server -- curl -s http://localhost:80

# Limpar
kubectl delete pod emptydir-demo
```

**O que acabou de acontecer:** Dois containers no mesmo pod compartilharam um sistema de arquivos. O sidecar busybox escreveu um arquivo HTML, e o container nginx o serviu. Nenhum container sabia sobre o outro — eles apenas concordaram em um caminho de montagem. Este é o padrão sidecar em ação.

### Lab 2: PersistentVolume e PersistentVolumeClaim — Provisionamento Manual

Crie um PV baseado em `hostPath` (adequado para testes locais com Kind), depois reivindique-o com um PVC.

Salve como `pv-demo.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-pv
spec:
  capacity:
    storage: 256Mi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: manual-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 256Mi
  storageClassName: ""
```

Note o `storageClassName: ""` — isso diz ao Kubernetes "não use provisionamento dinâmico; vincule a um PV existente."

```bash
# Criar o PV e PVC
kubectl apply -f pv-demo.yaml

# Verificar que o PVC está vinculado ao PV
kubectl get pv
kubectl get pvc
```

Você deve ver o status do PVC como `Bound` e o PV mostrando `manual-pvc` como sua solicitação.

Agora implante um pod que grava dados no volume. Salve como `pv-writer.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pv-writer
spec:
  containers:
  - name: writer
    image: busybox:1.36
    command: ["sh", "-c", "echo 'Data written at '$(date) > /data/message.txt && sleep 3600"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: manual-pvc
```

```bash
# Implantar o pod writer
kubectl apply -f pv-writer.yaml
kubectl wait --for=condition=Ready pod/pv-writer --timeout=60s

# Verificar que os dados foram gravados
kubectl exec pv-writer -- cat /data/message.txt

# Deletar o pod (mas NÃO o PVC)
kubectl delete pod pv-writer

# Implantar um novo pod que lê do mesmo PVC
# Salve como pv-reader.yaml:
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pv-reader
spec:
  containers:
  - name: reader
    image: busybox:1.36
    command: ["sh", "-c", "cat /data/message.txt && sleep 3600"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: manual-pvc
EOF

kubectl wait --for=condition=Ready pod/pv-reader --timeout=60s

# Os dados sobreviveram à deleção do pod!
kubectl exec pv-reader -- cat /data/message.txt

# Limpar
kubectl delete pod pv-reader
kubectl delete pvc manual-pvc
kubectl delete pv manual-pv
```

**O que acabou de acontecer:** Você criou armazenamento (PV), o solicitou (PVC), gravou dados, destruiu o pod, criou um novo pod, e os dados ainda estavam lá. O PVC manteve o PV vinculado e os dados intactos mesmo que o pod original tenha sido destruído.

### Lab 3: StorageClass — Provisionamento Dinâmico

Kind vem com um StorageClass padrão chamado `standard` baseado no Rancher local-path-provisioner. Vamos usá-lo.

```bash
# Inspecionar o StorageClass padrão
kubectl get storageclass
```

Você deve ver algo como:

```
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      AGE
standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   5m
```

Agora crie um PVC sem especificar um PV — o StorageClass vai provisionar um automaticamente. Salve como `dynamic-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 128Mi
```

```bash
# Criar o PVC
kubectl apply -f dynamic-pvc.yaml

# Verificar status do PVC — ele estará Pending até que um pod o use
# (porque o volumeBindingMode é WaitForFirstConsumer)
kubectl get pvc dynamic-pvc

# Implantar um pod que usa o PVC
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo 'Dynamic storage works!' > /data/proof.txt && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: dynamic-pvc
EOF

kubectl wait --for=condition=Ready pod/dynamic-demo --timeout=90s

# Agora verifique novamente — o PVC deve estar Bound
kubectl get pvc dynamic-pvc

# E um PV foi automaticamente criado!
kubectl get pv

# Verificar os dados
kubectl exec dynamic-demo -- cat /data/proof.txt

# Limpar
kubectl delete pod dynamic-demo
kubectl delete pvc dynamic-pvc
```

**O que acabou de acontecer:** Você criou um PVC sem criar um PV antes. Quando o pod foi agendado, o StorageClass `standard` automaticamente provisionou um PV e o vinculou ao seu PVC. Sem intervenção do admin. Isso é provisionamento dinâmico — o padrão usado em clusters de produção em todo lugar.

### Lab 4: StatefulSet com Armazenamento Por Réplica

StatefulSets são como o Kubernetes lida com aplicações stateful (bancos de dados, filas de mensagens). Cada réplica recebe seu próprio armazenamento persistente que a segue através de reinícios.

Salve como `statefulset-demo.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: stateful-svc
spec:
  clusterIP: None
  selector:
    app: stateful-demo
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: stateful-demo
spec:
  serviceName: stateful-svc
  replicas: 3
  selector:
    matchLabels:
      app: stateful-demo
  template:
    metadata:
      labels:
        app: stateful-demo
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          echo "I am $(hostname), writing my identity" > /data/identity.txt
          date >> /data/identity.txt
          sleep 3600
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 64Mi
```

```bash
# Implantar o StatefulSet
kubectl apply -f statefulset-demo.yaml

# Aguardar todos os pods estarem prontos
kubectl rollout status statefulset/stateful-demo --timeout=120s

# Cada pod tem sua própria identidade e seu próprio PVC
kubectl get pods -l app=stateful-demo
kubectl get pvc

# Verificar os dados únicos de cada pod
kubectl exec stateful-demo-0 -- cat /data/identity.txt
kubectl exec stateful-demo-1 -- cat /data/identity.txt
kubectl exec stateful-demo-2 -- cat /data/identity.txt

# Deletar um pod — o StatefulSet o recria com o MESMO nome e o MESMO PVC
kubectl delete pod stateful-demo-1
kubectl wait --for=condition=Ready pod/stateful-demo-1 --timeout=60s

# Os dados de antes da deleção ainda estão lá!
kubectl exec stateful-demo-1 -- cat /data/identity.txt

# Limpar
kubectl delete statefulset stateful-demo
kubectl delete svc stateful-svc
kubectl delete pvc data-stateful-demo-0 data-stateful-demo-1 data-stateful-demo-2
```

**O que acabou de acontecer:** O StatefulSet criou três pods (stateful-demo-0, -1, -2), cada um com seu próprio PVC (data-stateful-demo-0, -1, -2). Quando você deletou o pod -1, o StatefulSet o recriou com o mesmo nome e o reanexou ao mesmo PVC. Os dados sobreviveram. É assim que bancos de dados rodam no Kubernetes — cada réplica tem sua própria identidade e armazenamento persistente.

### Limpeza

```bash
# Deletar o cluster Kind quando terminar
kind delete cluster --name storage-lab

# Remover os arquivos YAML
rm -f emptydir-pod.yaml pv-demo.yaml pv-writer.yaml dynamic-pvc.yaml statefulset-demo.yaml
```

---

## Principais Conclusões

1. **Armazenamento de container é efêmero por padrão.** Qualquer coisa escrita dentro do sistema de arquivos de um container é perdida quando o pod morre. Se você precisa que dados sobrevivam, deve usar volumes.

2. **emptyDir é para armazenamento compartilhado temporário, com escopo de pod.** É perfeito para padrões sidecar onde containers no mesmo pod precisam compartilhar arquivos, mas os dados não precisam sobreviver ao pod.

3. **PersistentVolumes e PersistentVolumeClaims separam gerenciamento de armazenamento do consumo.** Admins provisionam PVs; desenvolvedores solicitam armazenamento via PVCs. Essa separação de responsabilidades é um princípio de design central do Kubernetes.

4. **StorageClasses habilitam provisionamento dinâmico.** Em produção, você quase nunca cria PVs manualmente. Você define um StorageClass, e o provisioner cria armazenamento sob demanda quando PVCs são submetidos.

5. **Modos de Acesso controlam acesso concorrente.** RWO para escrita em um único node, ROX para leitura em múltiplos nodes, RWX para escrita em múltiplos nodes. Seu backend de armazenamento determina quais modos estão disponíveis.

6. **StatefulSets dão a cada réplica seu próprio armazenamento persistente.** Através de `volumeClaimTemplates`, cada pod em um StatefulSet recebe um PVC dedicado que o segue através de reinícios e reagendamentos.

7. **Políticas de recuperação determinam o que acontece com dados após PVCs serem deletados.** Use `Retain` para dados que você não pode perder, `Delete` para armazenamento efêmero que deve ser limpo automaticamente.

---

## Leitura Complementar

- [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Dynamic Volume Provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
- [CSI Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#csi)
- [Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Rancher Local Path Provisioner (used by Kind)](https://github.com/rancher/local-path-provisioner)

---

**Anterior:** [Capítulo 7 — Rede](07-networking.md)
**Próximo:** [Capítulo 9 — Configuração e Secrets](09-configuration-and-secrets.md)
