# Capítulo 14: Operações em Produção

*"Implantar em produção não é a linha de chegada — é a linha de partida."*

---

## De Manutenção de Servidor Único para Ciclo de Vida do Cluster

Em um servidor Linux, operações Day-2 são familiares: você executa `apt upgrade`, agenda backups com `cron` de `/etc` e bancos de dados, coloca máquinas em modo de manutenção para troca de hardware e restaura de backup quando as coisas dão errado. Você já fez isso centenas de vezes.

No Kubernetes, as mesmas responsabilidades existem — mas são distribuídas entre múltiplos nós, múltiplos componentes do control plane e um armazenamento de estado compartilhado (etcd) que mantém a verdade de todo o seu cluster. Fazer upgrade não é "executar `apt upgrade` e reiniciar." É uma dança coreografada: faça upgrade do control plane, depois drene cada worker um por um, faça upgrade, libere-o e passe para o próximo. Pule um passo e você terá version skew. Apresse e terá downtime.

Este capítulo cobre as operações Day-2 essenciais que mantêm um cluster Kubernetes saudável: upgrades, backups, gerenciamento de nós, recuperação de desastres e o padrão Operator que automatiza tarefas operacionais complexas.

---

## Upgrades de Cluster

### O Ciclo de Vida do Upgrade

O Kubernetes segue uma sequência estrita de upgrade:

```
1. Upgrade do control plane (API server, controller-manager, scheduler, etcd)
2. Upgrade dos workers um de cada vez (drain → upgrade kubelet/kubectl → uncordon)
3. Verificar saúde do cluster após cada passo
```

Você não pode pular versões minor em um upgrade com kubeadm (por exemplo, 1.34 → 1.35, não 1.33 → 1.35). Use uma versão minor com suporte e seu patch mais recente com suporte, não a versão `.0` fixada para os labs descartáveis deste livro.

### O Fluxo de Upgrade com kubeadm

Este é um **modelo para kubeadm autogerenciado em Debian/Ubuntu com systemd**, não uma receita de upgrade do Kind nem um procedimento para serviços gerenciados. Ele pressupõe um repositório APT `pkgs.k8s.io` existente e uma chave de assinatura confiável, acesso root nos nós e um kubeconfig de administrador. Siga o [guia do kubeadm específico da versão](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) e as [instruções do repositório de pacotes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/). Adapte a versão minor se o destino com suporte do seu upgrade for outro.

**Passo 1 — Prepare-se e faça backups antes de alterar pacotes:**

- Leia as notas da versão de destino, APIs removidas, version skew, requisitos de SO/runtime e matrizes de compatibilidade de CNI/CSI/add-ons. Avance apenas uma versão minor por vez; use patches atuais com suporte e valide o caminho em staging.
- Tire e verifique um snapshot do etcd fora do nó, faça backup seguro de `/etc/kubernetes` e das chaves de criptografia e faça backups de dados consistentes com a aplicação. Ensaie a recuperação; os diretórios locais de backup do kubeadm não são um plano de recuperação de desastres.
- Confirme a saúde dos nós/control plane, quórum do etcd, CNI/DNS, prontidão dos workloads, capacidade livre suficiente e PDBs que permitam a operação. Revise dados locais/`emptyDir` e janelas de manutenção. Um cluster com control plane único terá indisponibilidade da API.

**Passo 2 — Em cada nó, quando chegar sua vez, selecione o repositório APT da versão minor de destino e a versão exata dos pacotes:**

```bash
# Example target minor; replace it for a different supported one-minor upgrade.
# Inspect the existing file and preserve its signed-by path/options.
sudo cat /etc/apt/sources.list.d/kubernetes.list
sudoedit /etc/apt/sources.list.d/kubernetes.list
# In that file, change ONLY the minor in the existing repository URL to:
# https://pkgs.k8s.io/core:/stable:/v1.35/deb/
sudo apt-get update
apt-cache madison kubeadm
apt-cache madison kubelet
apt-cache madison kubectl

# Fill these from the listings: latest supported patch, with exact package suffix.
TARGET_VERSION='v1.35.<PATCH>'
PACKAGE_VERSION='1.35.<PATCH>-<PACKAGE_REVISION>'
```

Os valores entre sinais de menor/maior são placeholders, **não escolhas de versão executáveis**. Confirme que a mesma versão selecionada está disponível para os três pacotes. Defina essas variáveis novamente no shell de cada nó; variáveis de shell não são transferidas entre máquinas. Se o repositório/chave estiver ausente ou tiver outro nome, configure-o pelas instruções oficiais, em vez de sobrescrever arquivos às cegas.

**Passo 3 — Faça upgrade do primeiro nó do control plane:**

```bash
# On the first control plane node:
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm="$PACKAGE_VERSION"
sudo apt-mark hold kubeadm
kubeadm version
sudo kubeadm upgrade plan "$TARGET_VERSION"
# Read the plan and resolve warnings before continuing.
sudo kubeadm upgrade apply "$TARGET_VERSION"
```

Siga a sequência de upgrade suportada pelo provedor de CNI e atualize o CNI conforme necessário; o kubeadm **não** faz isso por você. Verifique a saúde da rede/DNS. Depois conclua o **Passo 5 neste nó do control plane**, incluindo drain antes do upgrade do kubelet, antes de prosseguir.

**Passo 4 — Faça upgrade de cada control plane adicional e depois de cada worker, um nó por vez:**

Execute o Passo 2 nesse nó e depois:

```bash
# On the node currently being upgraded (additional control plane OR worker):
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm="$PACKAGE_VERSION"
sudo apt-mark hold kubeadm
kubeadm version
sudo kubeadm upgrade node
```

Use `upgrade apply` apenas no primeiro control plane; `upgrade node` atualiza a configuração dos demais nós e, nos control planes adicionais, seus manifestos estáticos. Conclua o Passo 5 em **cada** nó antes de seguir. Termine todos os control planes antes de iniciar os workers; não execute esse ciclo simultaneamente em vários membros do etcd.

**Passo 5 — Drene e atualize kubelet/kubectl em TODOS os nós, incluindo todos os control planes:**

```bash
# From the administrator's kubectl machine, set the actual node name:
NODE='<node-being-upgraded>'
kubectl drain "$NODE" --ignore-daemonsets --timeout=10m
# Only add --delete-emptydir-data after accepting loss of that node's emptyDir data.
# Resolve PDB/capacity/readiness blockers; do not bypass eviction protections.

# On that same node (with PACKAGE_VERSION set as in Step 2):
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet="$PACKAGE_VERSION" kubectl="$PACKAGE_VERSION"
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
sudo systemctl --no-pager status kubelet

# From the administrator's kubectl machine:
kubectl wait --for=condition=Ready "node/$NODE" --timeout=180s
kubectl get nodes -o wide
kubectl uncordon "$NODE"
```

Antes de iniciar o próximo nó, verifique a versão de kubelet reportada, a saúde da API/etcd quando aplicável, CNI/DNS, prontidão da aplicação e tráfego ponta a ponta. `drain` espera o término dos Pods removidos, **não** a prontidão dos substitutos; verifique explicitamente o rollout de cada Deployment/StatefulSet afetado, capacidade e saúde dos PDBs. Depois do último nó, verifique as versões de todos os componentes e os pacotes retidos, e procure regressões no monitoramento. Pare e investigue falhas em vez de continuar pela frota.

### Política de Version Skew

Componentes do Kubernetes têm regras estritas de compatibilidade:

| Componente | Skew Permitido do API Server |
|-----------|------------------------------|
| kubelet | Até 3 versões minor atrás |
| kube-proxy | Até 3 versões minor atrás |
| controller-manager, scheduler | Até 1 versão minor atrás |
| kubectl | ±1 versão minor |

Para um API server na v1.35, a regra geral de skew permite kubelets v1.35, v1.34, v1.33 ou v1.32 — nunca v1.36. Esse é um limite de compatibilidade, não uma recomendação de manter versões sem suporte. Em HA, o API server mais antigo limita os clientes compartilhados durante um rollout; kubeadm e provedores podem impor regras mais restritas. Consulte a [política completa de version skew](https://kubernetes.io/releases/version-skew-policy/).

### Upgrades em Kubernetes Gerenciado

Em serviços gerenciados (AKS, EKS, GKE), o provedor cuida dos upgrades do control plane. Você gerencia os upgrades do node pool:

| Provedor | Control Plane | Node Pool |
|----------|--------------|-----------|
| AKS | `az aks upgrade --kubernetes-version` | `az aks nodepool upgrade` |
| EKS | `aws eks update-cluster-version` | Atualizar AMI do node group |
| GKE | Automático ou `gcloud container clusters upgrade` | `gcloud container clusters upgrade --node-pool` |

---

## Backup e Restore do etcd

### Por Que o etcd É Crítico

O etcd é um armazenamento distribuído de chave-valor que mantém **todo** o estado do cluster: Deployments, Services, ConfigMaps, Secrets, regras RBAC, recursos customizados — tudo. Se o etcd for perdido sem um backup, seu cluster se foi. Os workloads continuam rodando (o kubelet continua gerenciando containers), mas você perde toda a capacidade de gerenciamento.

Pense no etcd como o diretório `/etc` do seu cluster inteiro, mais a tabela de processos, mais a tabela de rotas, tudo em um banco de dados.

### Backup com etcdctl

Os exemplos abaixo pressupõem um control plane kubeadm com etcd local protegido por TLS e os caminhos padrão de certificados. Execute no **host Linux do control plane**, não na sua estação; adapte endpoints e credenciais de cliente à implantação real. Use `etcdctl` para snapshots online e `etcdutl` para status/restore offline. Confira a imagem/versão do etcd em execução e as versões das duas ferramentas antes de usá-las. Para recuperação, use ferramentas e um servidor etcd compatíveis com a versão **major.minor** do snapshot (de preferência o mesmo patch); não combine restauração com um upgrade minor não planejado do etcd.

Crie um diretório de backup protegido fora do diretório de dados em uso e então tire um snapshot:

```bash
sudo install -d -m 0700 /backup
sudo etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

As flags de certificado são obrigatórias porque o etcd usa mutual TLS para toda comunicação.

### Verificar o Backup

```bash
sudo etcdutl snapshot status /backup/etcd-snapshot.db --write-out=table
```

Isso informa hash, revisão, quantidade de chaves e tamanho; é uma inspeção útil, **não uma prova de recuperabilidade ou de consistência dos dados da aplicação**. O restore verifica o hash de integridade de um snapshot criado por `snapshot save`; ensaie também uma restauração isolada. Criptografe e copie o snapshot para fora do nó/cluster, com retenção e controle de acesso. Faça backup separado de PKI, configuração/chaves de criptografia do API server e outras configurações do host. O etcd não contém o conteúdo dos PersistentVolumes.

### Exemplo de Restore de Membro Único — Somente Cluster Autogerenciado Descartável

**Limite de aplicação:** este modelo é para exatamente **um control plane e um membro etcd local**, com Pods estáticos do kubeadm, identidade/IP/PKI do nó inalterados e etcd **3.6.x**. Não é um restore de HA, um procedimento de serviço gerenciado nem um comando para executar no lab de backup Kind abaixo. Exige acesso a console/root independente do API server, snapshot verificado, `etcdutl`/`etcdctl` compatíveis disponíveis nesse host e janela de manutenção. Para outras versões/topologias, use a documentação de recuperação da versão correspondente.

Registre `--name`, URLs de peers e clientes, certificados, `--data-dir`, `volumeMounts` e `hostPath` do manifesto etcd existente. Um restore cria um **novo cluster lógico**; não use nomes de membros padrão/URLs de peers em localhost com um manifesto que espera outros valores.

Restaurar uma revisão antiga pode deixar os caches de informers/watch do Kubernetes inconsistentes. O [guia oficial de recuperação do etcd 3.6](https://etcd.io/docs/v3.6/op-guide/recovery/#restoring-with-revision-bump) recomenda **tanto** `--bump-revision` **quanto** `--mark-compacted` para Kubernetes: o incremento deve superar as revisões gravadas desde o snapshot, e a marcação de compactação invalida watches para que os clientes refaçam a listagem. O incremento de um bilhão do exemplo é ilustrativo, não um limite universal; calcule a margem com base na idade do snapshot e na taxa de gravação e confirme que o `etcdutl snapshot restore --help` compatível oferece essas flags.

1. Pare os escritores da API e os clientes do control plane, depois o etcd. Mantenha o kubelet em execução para que observe a remoção dos manifestos. Guarde os arquivos **fora** do diretório observado; até arquivos de backup dentro dele podem ser processados.

   ```bash
   # On the single control plane host; use a fresh protected holding directory.
   sudo install -d -m 0700 /etc/kubernetes/restore-hold
   sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /etc/kubernetes/restore-hold/
   sudo mv /etc/kubernetes/manifests/kube-controller-manager.yaml /etc/kubernetes/restore-hold/
   sudo mv /etc/kubernetes/manifests/kube-scheduler.yaml /etc/kubernetes/restore-hold/
   # Inspect the local runtime; wait until those containers have stopped.
   sudo crictl ps
   sudo mv /etc/kubernetes/manifests/etcd.yaml /etc/kubernetes/restore-hold/
   # Wait until etcd has also stopped before restoring.
   sudo crictl ps
   ```

   Não prossiga enquanto qualquer processo antigo de etcd/API server estiver em execução. Preserve o diretório de dados original para investigação; confirme que `/var/lib/etcd-restored` ainda não existe.

2. Restaure em um **novo diretório no host**, com identidade explícita de membro único. Substitua os placeholders pelos valores registrados do manifesto e escolha o incremento de revisão antes de executar:

   ```bash
   ETCD_NAME='<existing --name>'
   PEER_URL='<existing https peer URL on port 2380>'
   BUMP=1000000000
   sudo etcdutl snapshot restore /backup/etcd-snapshot.db \
     --data-dir=/var/lib/etcd-restored \
     --name="$ETCD_NAME" \
     --initial-cluster="${ETCD_NAME}=${PEER_URL}" \
     --initial-advertise-peer-urls="$PEER_URL" \
     --initial-cluster-token=single-member-recovery \
     --bump-revision="$BUMP" \
     --mark-compacted
   ```

   Não ignore a verificação de integridade. Garanta que o diretório restaurado tenha proprietário, permissões e, quando aplicável, rótulos SELinux exigidos pelo container etcd.

3. Edite o arquivo **retido** `/etc/kubernetes/restore-hold/etcd.yaml`. No layout padrão, altere apenas o **hostPath** do volume `etcd-data`, mantendo o ponto de montagem no container e seu data-dir:

   ```yaml
   # Relevant fields only — do not replace the entire static Pod with this fragment.
   spec:
     containers:
     - name: etcd
       command:
       - etcd
       - --data-dir=/var/lib/etcd
       # Keep all other command flags, TLS settings, probes, and mounts.
       volumeMounts:
       - name: etcd-data
         mountPath: /var/lib/etcd
     volumes:
     - name: etcd-data
       hostPath:
         path: /var/lib/etcd-restored
         type: DirectoryOrCreate
   ```

   Aqui, `/var/lib/etcd-restored` fica no **host**, montado em `/var/lib/etcd` **dentro** do container. Alterar apenas `--data-dir` não cria a montagem do host. Se o layout original for diferente, mapeie explicitamente os três caminhos. Garanta que nome/peers/bootstrap correspondam à composição restaurada de membro único; preserve SANs TLS válidos e endpoints de cliente (ou atualize a configuração do API server se os endpoints mudarem).

4. Devolva **apenas** o manifesto etcd e aguarde a saúde:

   ```bash
   sudo mv /etc/kubernetes/restore-hold/etcd.yaml /etc/kubernetes/manifests/
   sudo etcdctl --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
     --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
     endpoint health
   ```

   Repita as verificações de saúde enquanto o Pod estático inicia; inspecione logs do runtime em caso de falha. Verifique também status/revisão do endpoint e a lista de membros usando as mesmas flags TLS. Somente após confirmar uma composição saudável de **membro único**, devolva os demais manifestos e reinicie o kubelet conforme o [guia de restore do Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#restoring-an-etcd-cluster):

   ```bash
   sudo mv /etc/kubernetes/restore-hold/kube-apiserver.yaml /etc/kubernetes/manifests/
   sudo mv /etc/kubernetes/restore-hold/kube-controller-manager.yaml /etc/kubernetes/manifests/
   sudo mv /etc/kubernetes/restore-hold/kube-scheduler.yaml /etc/kubernetes/manifests/
   sudo systemctl restart kubelet
   # Once the API server is available, using an administrator kubeconfig:
   kubectl get --raw='/readyz?verbose'
   kubectl get nodes
   ```

   Verifique também os controllers, a leitura de Secrets criptografados e a recuperação da aplicação. Reinicie/reconcilie outros consumidores de watches conforme necessário; incremento de revisão/compactação e reinícios de clientes corrigem caches desatualizados, não dados perdidos de PV.

### Recuperação de HA Exige Outro Runbook

**Não repita o restore de membro único de forma independente em cada control plane.** Se o quórum sobreviver, use o [procedimento oficial de substituição de membros](https://etcd.io/docs/v3.6/op-guide/runtime-configuration/) para substituir os membros com falha um por vez, em vez de reverter todo o banco.

Se o quórum estiver irrecuperavelmente perdido, isole o cluster antigo, pare **todos** os API servers e coordene um novo cluster etcd lógico a partir do **mesmo snapshot**. Restaure cada novo membro com seu próprio nome, diretório de dados e URL de peer, mas com uma **lista completa de membros `--initial-cluster` compartilhada**, token e política de incremento de revisão/compactação comuns. Configure TLS de peers/clientes e endpoints do API server de forma correspondente, verifique o quórum e todos os membros e então reinicie API servers e consumidores de watches. Etcd externo exige seu próprio procedimento de serviço e armazenamento gerenciado pelo operador. Siga o [guia de recuperação de desastres do etcd](https://etcd.io/docs/v3.6/op-guide/recovery/) da versão correspondente e o guia de restore do Kubernetes; este capítulo intencionalmente não apresenta um restore genérico de HA para copiar e colar.

### Agendando Backups Regulares

Em produção, automatize backups do etcd:

- **CronJob no cluster:** Montar certificados do etcd e executar etcdctl em um agendamento
- **Cron externo:** O crontab de um nó do control plane executa o backup e copia para armazenamento remoto
- **Velero:** Faz backup de objetos Kubernetes selecionados **pela API**, com proteção de dados de volumes configurada separadamente; ele **não** tira snapshots do banco etcd

### Kubernetes Gerenciado: etcd É Invisível

No AKS, EKS e GKE, o provedor gerencia o etcd; o acesso direto aos membros e este procedimento de restore geralmente não estão disponíveis. Confirme as garantias de recuperação do control plane e as opções de restore oferecidas ao cliente. Replicação/backups gerenciados pelo provedor não substituem seu próprio plano de recuperação de recursos, volumes, estado externo e aplicações.

---

## Operações de Nó

### Cordon — Marcar como Não-Agendável

```bash
kubectl cordon <node-name>
```

Fazer cordon marca um nó como não-agendável. Pods existentes continuam rodando, mas o scheduler não colocará novos pods lá. É o "modo de manutenção" do Kubernetes.

```bash
# Check node status — look for SchedulingDisabled
kubectl get nodes
# NAME                      STATUS                     ROLES
# worker-1                  Ready,SchedulingDisabled   <none>
```

### Drain — Remover Pods Graciosamente

```bash
kubectl drain <node-name> --ignore-daemonsets --timeout=10m
```

Fazer drain faz três coisas:
1. Faz cordon do nó (marca como não-agendável)
2. Remove pods elegíveis pela API de eviction, respeitando PodDisruptionBudgets
3. Aguarda esses pods **terminarem**, não os substitutos serem agendados ou ficarem Ready

As flags:
- `--ignore-daemonsets` — Deixa os pods de DaemonSet no lugar; drain não os remove
- `--delete-emptydir-data` — Aceitação opcional e explícita da exclusão de dados de `emptyDir`; efêmero **não** significa que é seguro perder
- `--timeout=10m` — Limita esta tentativa; um timeout é uma falha no drain, não autorização para fazer manutenção

Antes de drenar, inspecione capacidade restante, afinidade/taints, topologia de PVCs, prontidão das réplicas, PDBs e dados locais. Pods mirror/estáticos permanecem; Pods sem controlador podem bloquear o drain e precisam de um plano deliberado de recuperação. Não adicione rotineiramente `--force` ou `--disable-eviction` para contornar bloqueios. Depois que o drain terminar com sucesso, verifique explicitamente rollouts afetados, backends prontos dos Services e tráfego da aplicação antes de interromper outro nó; os substitutos ainda podem estar Pending. Consulte o [contrato oficial do drain](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_drain/).

### Uncordon — Retornar ao Serviço

```bash
kubectl uncordon <node-name>
```

O nó se torna agendável novamente. Pods existentes em outros nós não migrarão automaticamente de volta — mas novos pods agora podem ser colocados aqui.

### O Padrão de Substituição de Nó

```bash
# 1. Drain the old node
kubectl drain old-node --ignore-daemonsets --timeout=10m
# Review/accept emptyDir loss separately if the drain reports it.
# Verify replacements and application readiness before continuing.

# 2. Terminate the old node (cloud provider specific)
# aws ec2 terminate-instances --instance-ids ...
# az vm delete ...

# 3. Add a new node (join token from kubeadm)
kubeadm token create --print-join-command
# On new node:
kubeadm join <control-plane>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

# 4. Verify
kubectl get nodes
```

---

## Estratégia de Recuperação de Desastres

### O Que Fazer Backup

Um plano completo de recuperação de desastres cobre:

| O Quê | Por Quê | Como |
|------|-----|-----|
| **etcd** | Todo o estado do cluster (Deployments, Services, RBAC) | `etcdctl snapshot save` |
| **Persistent Volumes** | Dados da aplicação (bancos de dados, uploads) | CSI snapshots, Velero |
| **Estado externo** | Registros DNS, certificados TLS, configurações IAM | Infrastructure-as-Code (Terraform) |
| **Manifestos YAML** | Fonte da verdade para todos os recursos | Repositório Git (GitOps) |
| **Secrets e chaves de criptografia** | Credenciais e capacidade de descriptografar objetos restaurados | Armazenamentos de secrets externos; configuração/chaves de criptografia protegidas separadamente |

### RPO e RTO

| Termo | Significado | Exemplo |
|------|---------|---------|
| **RPO** (Recovery Point Objective) | Quanto de dados você pode perder? | "1 hora" = backups de hora em hora aceitáveis |
| **RTO** (Recovery Time Objective) | Quão rápido deve recuperar? | "15 minutos" = recuperação automatizada necessária |

Sua frequência de backup (RPO) e automação de recuperação (RTO) determinam seu investimento em recuperação de desastres.

### Estratégias Multi-Cluster

| Estratégia | Descrição | Complexidade | Caso de Uso |
|----------|-------------|-----------|----------|
| **Ativo/Passivo** | Cluster primário serve tráfego; standby recebe backups | Média | Sensível a custo, RTO moderado |
| **Ativo/Ativo** | Múltiplos clusters servem tráfego simultaneamente | Alta | Requisitos de zero-downtime |
| **Backup/Restore** | Cluster único com backups off-site | Baixa | Dev/staging, RTO longo aceitável |

### Velero para Backup de Cluster

O Velero faz backup de objetos de recursos Kubernetes consultando o **API server**, não tirando snapshots do etcd. A proteção de volumes depende de plugins compatíveis de provedor/armazenamento de objetos, suporte a snapshots CSI e configuração de `VolumeSnapshotClass`, ou configuração suportada de backup de sistema de arquivos/data mover. Não é automaticamente uma cópia portável de todos os PVs.

Antes de usar estes comandos ilustrativos, instale uma versão do Velero com suporte à sua versão do Kubernetes, o plugin correspondente de provedor/armazenamento de objetos, credenciais/IAM e o local de armazenamento de backups. Configure o método de backup de volumes escolhido e teste a restauração dos dados e metadados. A linha de instalação é intencionalmente incompleta, não uma configuração executável de provedor:

```bash
# Install Velero (example with cloud provider plugin)
velero install --provider aws --bucket my-backup-bucket ...

# Create a backup
velero backup create my-backup --include-namespaces production

# Schedule recurring backups
velero schedule create daily --schedule="0 2 * * *" --include-namespaces production

# Restore
velero restore create --from-backup my-backup
```

Backups de recursos não são estritamente atômicos. Um snapshot de disco/CSI sozinho pode ser consistente apenas em caso de falha, não consistente com a aplicação; bancos de dados podem exigir pausa controlada de gravações, hooks de backup, captura de logs de transação ou backups nativos do banco. Coordene os pontos de recuperação de recursos e dados, verifique erros/falhas parciais de backup e ensaie a restauração em um cluster isolado com APIs, CRDs, classes de armazenamento e credenciais compatíveis. Consulte [como o Velero funciona](https://velero.io/docs/v1.18/how-velero-works/) e [hooks de backup](https://velero.io/docs/v1.18/backup-hooks/).

---

## CRDs e Operators

### Custom Resource Definitions (CRDs)

O Kubernetes vem com tipos de recursos nativos: Pods, Deployments, Services, ConfigMaps. CRDs permitem estender a API com seus próprios tipos de recursos.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                engine:
                  type: string
                version:
                  type: string
                storage:
                  type: string
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames:
    - db
```

Após aplicar este CRD, você pode criar recursos `Database` como qualquer tipo nativo:

```yaml
apiVersion: example.com/v1
kind: Database
metadata:
  name: orders-db
spec:
  engine: postgresql
  version: "16"
  storage: "50Gi"
```

E consultá-los com kubectl:

```bash
kubectl get databases
kubectl describe database orders-db
```

### O Padrão Operator

Um CRD sozinho é apenas dados — não faz nada. Um **Operator** é um controller customizado que observa seu CRD e toma ações. É o padrão de loop de reconciliação (do Capítulo 4) aplicado a lógica específica de domínio.

Pense desta forma:
- **CRD** = "Eu quero um banco de dados PostgreSQL com 3 réplicas e 50 GiB de armazenamento"
- **Operator** = A automação que cria o StatefulSet, configura replicação, gerencia backups, lida com failover e mantém o banco de dados no estado desejado

Operators codificam conhecimento operacional em software. Em vez de um DBA executar comandos manuais de failover às 3 da manhã, o operator detecta que o primário caiu e promove uma réplica automaticamente.

### Operators Populares

| Operator | O Que Gerencia | Equivalente Linux |
|----------|----------------|-----------------|
| cert-manager | Certificados TLS (Let's Encrypt) | certbot + cron |
| prometheus-operator | Stack de monitoramento (Prometheus + Alertmanager) | prometheus + systemd |
| postgres-operator | Clusters PostgreSQL (replicação, backup, failover) | patroni + trabalho manual de DBA |
| strimzi | Clusters Apache Kafka | Scripts Kafka + operações manuais |
| ArgoCD | Deploy contínuo GitOps | Scripts de deploy + webhooks |

### Encontrando Operators

- **OperatorHub.io** (https://operatorhub.io) — Catálogo de operators da comunidade
- **Artifact Hub** (https://artifacthub.io) — Inclui Helm charts e operators
- **GitHub** — Muitos operators são projetos open source

---

## Ferramentas de Ciclo de Vida do Cluster

| Ferramenta | Caso de Uso | Tipo |
|------|----------|------|
| **kubeadm** | Bootstrap de clusters bare-metal/VM | Autogerenciado |
| **Kind** | Desenvolvimento local e testes | Dev/CI |
| **k3s** | Clusters leves, edge, IoT | Autogerenciado (mínimo) |
| **AKS** | Kubernetes gerenciado Azure | Gerenciado em nuvem |
| **EKS** | Kubernetes gerenciado AWS | Gerenciado em nuvem |
| **GKE** | Kubernetes gerenciado Google Cloud | Gerenciado em nuvem |

### Comparação Gerenciado vs. Autogerenciado

| Aspecto | Autogerenciado (kubeadm) | Gerenciado em Nuvem (AKS/EKS/GKE) |
|--------|----------------------|------------------------------|
| Control plane | Você gerencia | Provedor gerencia |
| etcd | Você faz backup/restore | Provedor cuida |
| Upgrades | Manual, passo a passo | Semi-automatizado |
| Escala de nós | Manual ou cluster-autoscaler | Autoscaler integrado |
| Custo | Apenas infraestrutura | Infraestrutura + taxa de gerenciamento |
| Customização | Controle total | Guardrails do provedor |
| SLA | Sua responsabilidade | SLA do provedor (99.95%+) |

---

## Comparação de Operações Linux ↔ Kubernetes

| Operação Linux | Equivalente K8s | Notas |
|----------------|---------------|-------|
| `apt upgrade` + reboot | `kubeadm upgrade` + drain/uncordon | Upgrade rolling nó por nó |
| `rsync /etc`, cron backups | `etcdctl snapshot save` | Backup do estado do cluster |
| Restaurar de backup | `etcdutl snapshot restore` | Recuperação do etcd compatível com a versão e específica da topologia |
| Modo de manutenção | `kubectl cordon` + `drain` | Tirar nó do ar com segurança |
| Agentes `puppet`/`chef` | Operators | Gerenciamento automatizado de aplicações |
| `/etc/cron.d/backup` | Backups agendados do Velero | Backups de objetos da API e proteção de volumes configurada, não snapshots do etcd |
| Scripts init customizados | CRDs + controllers customizados | Estender comportamento do sistema |
| Cluster (Pacemaker/Corosync) | HA do control plane K8s | Disponibilidade baseada em quórum |

---

> ### ⚠️ Onde a Analogia com Linux Quebra
>
> **Upgrades in-place vs. rolling:** Upgrades Linux podem ser feitos in-place — `apt upgrade` em uma máquina rodando. Upgrades Kubernetes são rolling: você faz upgrade do control plane primeiro, depois drena e faz upgrade de cada worker node individualmente. Você não pode simplesmente executar "yum update kubernetes" em um nó ativo com pods rodando. A orquestração é parte do processo.
>
> **Nenhum "backup de tudo" único:** No Linux, você pode fazer `tar` de todo o filesystem e chamar de backup. No Kubernetes, o estado do cluster (etcd) e os dados da aplicação (PersistentVolumes) vivem em lugares diferentes com mecanismos de backup diferentes. Você precisa de ambos, mais estado externo como registros DNS, certificados TLS e configurações IAM. Uma estratégia completa de backup toca múltiplos sistemas.
>
> **Operators são mais que scripts:** Um cron job Linux roda periodicamente e aplica uma correção. Um Operator roda continuamente, observa recursos em tempo real e reage imediatamente. É a diferença entre verificar seu email uma vez por hora e ter notificações push. Operators não apenas automatizam tarefas — eles codificam expertise de domínio e fazem self-heal em tempo real.

---

## Laboratório Diagnóstico: Operações em Produção

### Pré-requisitos

Use **Kind v0.31.0**, **kubectl v1.35.0** e o digest da imagem Kubernetes **v1.35.0** publicado na [versão do Kind](https://github.com/kubernetes-sigs/kind/releases/tag/v0.31.0). Estes labs Bash exigem um daemon Docker em execução com containers Linux/cgroup v2 (por exemplo, via WSL2 no Windows), internet e capacidade para três nós. Use um contexto de lab descartável. O Kind é recriado com uma nova imagem de node em vez de atualizado in-place; os modelos de APT/restore de produção acima **não** são comandos para executar dentro desses nós Kind. Os resultados abaixo são observações esperadas, não evidências de testes em runtime.

```bash
# Create a multi-node Kind cluster
kind create cluster --name prod-ops-lab \
  --image kindest/node:v1.35.0@sha256:452d707d4862f52530247495d180205e029056831160e22870e37e3f6c1ac31f \
  --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
  labels:
    node-role: worker
- role: worker
  labels:
    node-role: worker
EOF

# Verify cluster
kubectl get nodes
```

---

### Lab 1: Drain e Uncordon

**Passo 1 — Implantar um workload entre os workers:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-fleet
spec:
  replicas: 6
  selector:
    matchLabels:
      app: web-fleet
  template:
    metadata:
      labels:
        app: web-fleet
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
        resources:
          requests:
            cpu: 50m
            memory: 32Mi
          limits:
            memory: 128Mi
EOF

# Wait for pods to be ready
kubectl wait --for=condition=Ready pod -l app=web-fleet --timeout=60s
kubectl get pods -l app=web-fleet -o wide
```

Observe quais pods estão em quais nós.

**Passo 2 — Fazer cordon de um worker:**

```bash
# Get worker node names
WORKER=$(kubectl get nodes --selector='!node-role.kubernetes.io/control-plane' -o jsonpath='{.items[0].metadata.name}')
echo "Cordoning node: $WORKER"

kubectl cordon $WORKER
kubectl get nodes
```

**Passo 3 — Verificar que novos pods não podem ser agendados no nó com cordon:**

```bash
# Scale up — new pods should only go to the other worker
kubectl scale deployment web-fleet --replicas=10
sleep 5
kubectl get pods -l app=web-fleet -o wide | grep -c "$WORKER"
# Existing pods remain, but no new pods land on the cordoned node
```

**Passo 4 — Drenar o worker com cordon:**

Verifique se o outro worker tem capacidade para as réplicas desejadas e se os PDBs permitem a interrupção. Esta demonstração não tem dados de `emptyDir` para descartar; não adicione a flag de exclusão por hábito.

```bash
kubectl get pdb
kubectl drain "$WORKER" --ignore-daemonsets --timeout=120s
kubectl rollout status deployment/web-fleet --timeout=120s
kubectl get pods -l app=web-fleet -o wide
# Verify desired replicas are Ready on remaining schedulable nodes.
# Drain completion alone did not establish that.
```

**Passo 5 — Fazer uncordon e rebalancear:**

```bash
kubectl uncordon $WORKER
kubectl get nodes

# Scale down and back up to trigger redistribution
kubectl scale deployment web-fleet --replicas=3
sleep 5
kubectl scale deployment web-fleet --replicas=6
sleep 10
kubectl get pods -l app=web-fleet -o wide
# Pods should spread across both workers again
```

**Limpeza:**

```bash
kubectl delete deployment web-fleet
```

---

### Lab 2: Backup do etcd

Este é um **lab apenas de snapshot**, não um exercício de restore. Verifique a imagem do etcd e as versões de `etcdctl`/`etcdutl`; estes exemplos exigem ferramentas compatíveis na imagem do etcd. Se `etcdutl` estiver ausente, inspecione o snapshot copiado com uma ferramenta offline correspondente em um host administrativo; não recorra aos comandos removidos `etcdctl snapshot status/restore`. Não instale ferramentas arbitrárias no control plane.

**Passo 1 — Identificar o pod do etcd:**

```bash
kubectl get pods -n kube-system | grep etcd
# etcd-prod-ops-lab-control-plane
kubectl get pod -n kube-system etcd-prod-ops-lab-control-plane \
  -o jsonpath='{.spec.containers[0].image}{"\n"}'
kubectl exec -n kube-system etcd-prod-ops-lab-control-plane -- etcdctl version
kubectl exec -n kube-system etcd-prod-ops-lab-control-plane -- etcdutl version
```

**Passo 2 — Verificar saúde do etcd:**

```bash
kubectl exec -n kube-system etcd-prod-ops-lab-control-plane -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```

**Passo 3 — Criar um snapshot (armazenamento temporário na montagem gravável do etcd):**

```bash
kubectl exec -n kube-system etcd-prod-ops-lab-control-plane -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/lib/etcd/lab-snapshot.db
```

**Passo 4 — Verificar o backup:**

```bash
kubectl exec -n kube-system etcd-prod-ops-lab-control-plane -- etcdutl \
  snapshot status /var/lib/etcd/lab-snapshot.db --write-out=table

# Kind's hostPath is inside the node container. Copy out before deleting the cluster.
# This avoids kubectl cp, which requires tar inside the etcd image.
docker cp prod-ops-lab-control-plane:/var/lib/etcd/lab-snapshot.db ./etcd-lab-snapshot.db
# Only after successful inspection/copy, remove the staging file:
docker exec prod-ops-lab-control-plane rm /var/lib/etcd/lab-snapshot.db
```

Você deve ver hash, revisão, quantidade de chaves e tamanho. Inspecionar/copiar não é um teste de restore bem-sucedido. A cópia local ainda está na mesma estação: produção exige armazenamento criptografado fora do cluster e um restore ensaiado e específico da topologia. Proteja esse arquivo como dado sensível do cluster.

**Passo 5 — Verificar o que está armazenado no etcd:**

```bash
# Count the keys (gives you a sense of cluster state size)
kubectl exec -n kube-system etcd-prod-ops-lab-control-plane -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get / --prefix --keys-only | head -20
```

---

### Lab 3: Custom Resource Definitions

**Passo 1 — Criar um CRD:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: webapps.lab.example.com
spec:
  group: lab.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                language:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
                domain:
                  type: string
      additionalPrinterColumns:
      - name: Language
        type: string
        jsonPath: .spec.language
      - name: Replicas
        type: integer
        jsonPath: .spec.replicas
      - name: Domain
        type: string
        jsonPath: .spec.domain
  scope: Namespaced
  names:
    plural: webapps
    singular: webapp
    kind: WebApp
    shortNames:
    - wa
EOF
```

**Passo 2 — Verificar que o CRD está registrado:**

```bash
kubectl get crd | grep webapp
kubectl api-resources | grep webapp
```

**Passo 3 — Criar recursos customizados:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: lab.example.com/v1
kind: WebApp
metadata:
  name: frontend
spec:
  language: React
  replicas: 3
  domain: app.example.com
---
apiVersion: lab.example.com/v1
kind: WebApp
metadata:
  name: api-service
spec:
  language: Go
  replicas: 5
  domain: api.example.com
---
apiVersion: lab.example.com/v1
kind: WebApp
metadata:
  name: admin-panel
spec:
  language: Python
  replicas: 2
  domain: admin.example.com
EOF
```

**Passo 4 — Consultar seus recursos customizados:**

```bash
# List all webapps (note the custom columns from additionalPrinterColumns)
kubectl get webapps

# Use the short name
kubectl get wa

# Describe one
kubectl describe webapp frontend

# Get as YAML
kubectl get webapp api-service -o yaml
```

**Passo 5 — Limpar recursos do CRD:**

```bash
kubectl delete webapp --all
kubectl delete crd webapps.lab.example.com
```

---

### Lab 4: Simular Fluxo de Upgrade do Cluster

O Kind não suporta upgrade in-place da versão do Kubernetes. Este lab ensaia apenas inspeção e drain/uncordon; não valida o procedimento de upgrade de pacotes ou do control plane de produção acima.

**Passo 1 — Verificar versões atuais:**

```bash
kubectl get nodes -o wide
kubectl version
```

**Passo 2 — Revisar o plano de upgrade (conceitual):**

```bash
# On a real kubeadm cluster, you would run:
# sudo kubeadm upgrade plan
# This shows available versions, required actions, and deprecation warnings

# In our lab, let's at least check what kubeadm would report
echo "=== Current Cluster Version ==="
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kubeletVersion}{"\n"}{end}'
```

**Passo 3 — Praticar o ciclo drain/uncordon:**

```bash
# Deploy a workload
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: upgrade-test
spec:
  replicas: 4
  selector:
    matchLabels:
      app: upgrade-test
  template:
    metadata:
      labels:
        app: upgrade-test
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
EOF

kubectl wait --for=condition=Ready pod -l app=upgrade-test --timeout=60s

# Get a worker node
WORKER=$(kubectl get nodes --selector='!node-role.kubernetes.io/control-plane' -o jsonpath='{.items[0].metadata.name}')

# Step A: Drain (simulating pre-upgrade)
echo "Draining $WORKER..."
kubectl drain "$WORKER" --ignore-daemonsets --timeout=120s

# Drain waited for termination; independently verify replacements.
kubectl rollout status deployment/upgrade-test --timeout=120s
kubectl get pods -l app=upgrade-test -o wide
echo "Check desired Ready replicas and placement before continuing"

# Step B: Uncordon (simulating post-upgrade)
echo "Uncordoning $WORKER..."
kubectl uncordon $WORKER

# Step C: Verify node is back in rotation
kubectl get nodes
```

**Passo 4 — Documentar os passos reais de upgrade:**

```bash
echo "
=== PRODUCTION UPGRADE CHECKLIST ===
1. Read release notes/skew; choose latest supported patch, one minor at a time
2. Verify etcd, PKI/encryption-key and application backups; rehearse recovery
3. Check CNI/CSI/add-on compatibility, spare capacity, PDBs and local data
4. First control plane: select target-minor APT repo, update package index
   a. Unhold kubeadm, install exact target package, hold; verify version
   b. kubeadm upgrade plan <target>; review, then upgrade apply <target>
   c. Follow the CNI provider upgrade procedure and verify networking
   d. Drain this control plane before its kubelet upgrade
   e. Unhold kubelet/kubectl, install target packages, hold
   f. daemon-reload, restart kubelet; verify health/version, then uncordon
5. EACH additional control plane, then EACH worker, one at a time:
   a. Select target-minor repo; unhold/install/hold target kubeadm
   b. kubeadm upgrade node
   c. Drain; assess emptyDir loss explicitly, do not bypass PDBs
   d. Unhold/install/hold target kubelet and kubectl
   e. daemon-reload, restart kubelet; verify health/version, then uncordon
6. Between nodes, verify replacement readiness, DNS/CNI and application traffic
7. Verify whole-cluster versions, readiness, package holds and monitoring
"
```

**Limpeza:**

```bash
kubectl delete deployment upgrade-test
```

---

### Limpeza do Laboratório

```bash
# Delete the Kind cluster
kind delete cluster --name prod-ops-lab
# Remove the sensitive local lab snapshot when no longer needed.
rm -f ./etcd-lab-snapshot.db
```

---

## Principais Conclusões

1. **Faça upgrade dos control planes primeiro, depois dos workers, uma versão minor por vez.** Instale o kubeadm de destino antes de planejar/aplicar; selecione o repositório de pacotes da versão minor de destino e execute unhold/install/hold dos pacotes. Drene cada nó antes do upgrade de seu kubelet, incluindo control planes, reinicie o kubelet e verifique a saúde entre nós. O skew geral do kubelet permite até 3 versões minor atrás do API server, nunca à frente; restrições de versões com suporte, kubeadm e HA continuam valendo.

2. **Drain espera o término, não a prontidão dos substitutos.** Revise capacidade, PDBs e dados locais antes; verifique explicitamente a prontidão dos workloads depois. `--delete-emptydir-data` aceita perda de dados, e pods estáticos/DaemonSet ficam fora da remoção comum pelo drain.

3. **Faça backup do etcd e ensaie a recuperação específica da topologia.** Use `etcdctl snapshot save`, `etcdutl` compatível para status/restore, armazenamento criptografado fora do cluster e PKI/chaves de criptografia protegidas separadamente. Restore de membro único não é recuperação de HA; planeje incremento de revisão/compactação e atualização dos caches de watches conforme a orientação oficial.

4. **Recuperação de desastres requer múltiplos alvos de backup.** O etcd cobre o estado da API do cluster, não dados de PVs. O Velero faz backup de objetos da API e dados de volumes configurados, não snapshots do etcd. Valide plugins, consistência com a aplicação, estado externo e recuperação ponta a ponta.

5. **CRDs estendem a API do Kubernetes.** Custom Resource Definitions permitem adicionar seus próprios tipos de recursos que funcionam com kubectl, RBAC e toda a maquinaria da API do Kubernetes.

6. **Operators automatizam operações Day-2.** Operators são controllers customizados que codificam expertise operacional (backup, failover, scaling) em software que reage em tempo real — não cron jobs periódicos.

7. **Kubernetes gerenciado cuida das partes difíceis.** AKS, EKS e GKE gerenciam o control plane, etcd e upgrades. Você foca em node pools e workloads. Para produção, gerenciado é quase sempre a escolha certa a menos que você tenha requisitos específicos.

---

## Leitura Adicional

- [Upgrading kubeadm Clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [Alteração do Repositório de Pacotes Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/)
- [Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
- [Backing Up an etcd Cluster](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)
- [Recuperação de Desastres do etcd 3.6 — Incrementos de Revisão e Membros](https://etcd.io/docs/v3.6/op-guide/recovery/)
- [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [Operator Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)
- [Velero Documentation](https://velero.io/docs/)
- [OperatorHub.io](https://operatorhub.io/)

---

**Anterior:** [Capítulo 13 — Solução de Problemas](13-troubleshooting.md)
**Próximo:** [Capítulo 15 — Próximos Passos](15-next-steps.md)
