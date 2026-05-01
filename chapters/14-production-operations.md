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

Você não pode pular versões. O Kubernetes suporta upgrades de uma versão minor por vez (ex: 1.31 → 1.32, não 1.30 → 1.32).

### O Fluxo de Upgrade com kubeadm

Para clusters autogerenciados usando kubeadm:

**Passo 1 — Verificar upgrades disponíveis:**

```bash
sudo kubeadm upgrade plan
```

Este comando inspeciona seu cluster atual, verifica a versão instalada do kubeadm e reporta para quais versões você pode fazer upgrade. Ele também alerta sobre versões de API depreciadas e ações necessárias.

**Passo 2 — Fazer upgrade do primeiro nó do control plane:**

```bash
# Fazer upgrade do próprio kubeadm primeiro
sudo apt-get update
sudo apt-get install -y kubeadm=1.32.0-1.1

# Executar o upgrade
sudo kubeadm upgrade apply v1.32.0

# Fazer upgrade do kubelet e kubectl neste nó
sudo apt-get install -y kubelet=1.32.0-1.1 kubectl=1.32.0-1.1
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

**Passo 3 — Fazer upgrade de nós adicionais do control plane (se HA):**

```bash
sudo kubeadm upgrade node
```

**Passo 4 — Fazer upgrade dos worker nodes (um de cada vez):**

```bash
# De uma máquina com acesso kubectl:
kubectl drain <worker-node> --ignore-daemonsets --delete-emptydir-data

# No worker node:
sudo apt-get update
sudo apt-get install -y kubeadm=1.32.0-1.1
sudo kubeadm upgrade node
sudo apt-get install -y kubelet=1.32.0-1.1 kubectl=1.32.0-1.1
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Da máquina com kubectl:
kubectl uncordon <worker-node>
```

Repita para cada worker.

### Política de Version Skew

Componentes do Kubernetes têm regras estritas de compatibilidade:

| Componente | Skew Permitido do API Server |
|-----------|------------------------------|
| kubelet | Até 3 versões minor atrás |
| kube-proxy | Até 3 versões minor atrás |
| controller-manager, scheduler | Até 1 versão minor atrás |
| kubectl | ±1 versão minor |

Isso significa que com um API server na v1.32, seus kubelets podem ser v1.32, v1.31, v1.30 ou v1.29 — mas não v1.28 ou v1.33.

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

```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

As flags de certificado são obrigatórias porque o etcd usa mutual TLS para toda comunicação.

### Verificar o Backup

```bash
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table
```

Isso gera uma tabela mostrando hash, revisão, total de chaves e tamanho total — confirmando que o backup é válido e completo.

### Restaurar de Backup

```bash
# Parar o API server e etcd (em clusters kubeadm, mover manifestos de static pod)
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/

# Restaurar o snapshot para um novo diretório de dados
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-from-backup

# Atualizar config do etcd para usar o novo diretório de dados
# (editar o manifesto etcd.yaml para apontar para /var/lib/etcd-from-backup)

# Reiniciar componentes
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/
```

> **Nota:** A partir do etcd 3.5+, a operação de restore está migrando para `etcdutl snapshot restore`. Ambas as ferramentas funcionam para versões atuais do Kubernetes, mas `etcdutl` é a ferramenta recomendada daqui em diante.

### Agendando Backups Regulares

Em produção, automatize backups do etcd:

- **CronJob no cluster:** Montar certificados do etcd e executar etcdctl em um agendamento
- **Cron externo:** O crontab de um nó do control plane executa o backup e copia para armazenamento remoto
- **Velero:** Ferramenta de backup com consciência do cluster que lida tanto com estado do etcd quanto volumes persistentes

### Kubernetes Gerenciado: etcd É Invisível

No AKS, EKS e GKE, o provedor gerencia o etcd completamente. Você não pode acessá-lo diretamente — e não precisa. O provedor cuida da replicação, backup e recuperação de desastres do armazenamento de dados do control plane.

---

## Operações de Nó

### Cordon — Marcar como Não-Agendável

```bash
kubectl cordon <node-name>
```

Fazer cordon marca um nó como não-agendável. Pods existentes continuam rodando, mas o scheduler não colocará novos pods lá. É o "modo de manutenção" do Kubernetes.

```bash
# Verificar status do nó — procure por SchedulingDisabled
kubectl get nodes
# NAME                      STATUS                     ROLES
# worker-1                  Ready,SchedulingDisabled   <none>
```

### Drain — Remover Pods Graciosamente

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

Fazer drain faz três coisas:
1. Faz cordon do nó (marca como não-agendável)
2. Remove todos os pods (respeitando PodDisruptionBudgets)
3. Aguarda os pods serem reagendados em outro lugar

As flags:
- `--ignore-daemonsets` — Pods de DaemonSet não podem ser removidos (devem rodar em todo nó), então ignore-os
- `--delete-emptydir-data` — Permitir remoção de pods com volumes emptyDir (seus dados são efêmeros de qualquer forma)

### Uncordon — Retornar ao Serviço

```bash
kubectl uncordon <node-name>
```

O nó se torna agendável novamente. Pods existentes em outros nós não migrarão automaticamente de volta — mas novos pods agora podem ser colocados aqui.

### O Padrão de Substituição de Nó

```bash
# 1. Drenar o nó antigo
kubectl drain old-node --ignore-daemonsets --delete-emptydir-data

# 2. Terminar o nó antigo (específico do provedor cloud)
# aws ec2 terminate-instances --instance-ids ...
# az vm delete ...

# 3. Adicionar um novo nó (token de join do kubeadm)
kubeadm token create --print-join-command
# No novo nó:
kubeadm join <control-plane>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

# 4. Verificar
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
| **Secrets** (criptografados) | Credenciais, chaves de API | Sealed Secrets, armazenamentos de secrets externos |

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

Velero é um projeto CNCF que faz backup de recursos Kubernetes e volumes persistentes:

```bash
# Instalar Velero (exemplo com plugin de provedor cloud)
velero install --provider aws --bucket my-backup-bucket ...

# Criar um backup
velero backup create my-backup --include-namespaces production

# Agendar backups recorrentes
velero schedule create daily --schedule="0 2 * * *" --include-namespaces production

# Restaurar
velero restore create --from-backup my-backup
```

Velero é o equivalente Kubernetes de ferramentas empresariais de backup como Veeam ou Commvault — mas construído especificamente para o modelo de recursos do K8s.

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
| Restaurar de backup | `etcdctl snapshot restore` | Recuperação do cluster |
| Modo de manutenção | `kubectl cordon` + `drain` | Tirar nó do ar com segurança |
| Agentes `puppet`/`chef` | Operators | Gerenciamento automatizado de aplicações |
| `/etc/cron.d/backup` | Backups agendados do Velero | Snapshots periódicos do cluster |
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

```bash
# Criar um cluster Kind multi-nó
kind create cluster --name prod-ops-lab --config - <<EOF
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

# Verificar cluster
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
EOF

# Aguardar pods ficarem prontos
kubectl wait --for=condition=Ready pod -l app=web-fleet --timeout=60s
kubectl get pods -l app=web-fleet -o wide
```

Observe quais pods estão em quais nós.

**Passo 2 — Fazer cordon de um worker:**

```bash
# Obter nomes dos worker nodes
WORKER=$(kubectl get nodes --selector='!node-role.kubernetes.io/control-plane' -o jsonpath='{.items[0].metadata.name}')
echo "Cordoning node: $WORKER"

kubectl cordon $WORKER
kubectl get nodes
```

**Passo 3 — Verificar que novos pods não podem ser agendados no nó com cordon:**

```bash
# Escalar para cima — novos pods devem ir apenas para o outro worker
kubectl scale deployment web-fleet --replicas=10
sleep 5
kubectl get pods -l app=web-fleet -o wide | grep -c "$WORKER"
# Pods existentes permanecem, mas nenhum novo pod vai para o nó com cordon
```

**Passo 4 — Drenar o worker com cordon:**

```bash
kubectl drain $WORKER --ignore-daemonsets --delete-emptydir-data
kubectl get pods -l app=web-fleet -o wide
# Todos os pods web-fleet agora estão no(s) worker(s) restante(s)
```

**Passo 5 — Fazer uncordon e rebalancear:**

```bash
kubectl uncordon $WORKER
kubectl get nodes

# Escalar para baixo e de volta para cima para disparar redistribuição
kubectl scale deployment web-fleet --replicas=3
sleep 5
kubectl scale deployment web-fleet --replicas=6
sleep 10
kubectl get pods -l app=web-fleet -o wide
# Pods devem se espalhar entre ambos os workers novamente
```

**Limpeza:**

```bash
kubectl delete deployment web-fleet
```

---

### Lab 2: Backup do etcd

**Passo 1 — Identificar o pod do etcd:**

```bash
kubectl get pods -n kube-system | grep etcd
# etcd-prod-ops-lab-control-plane
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

**Passo 3 — Criar um snapshot:**

```bash
kubectl exec -n kube-system etcd-prod-ops-lab-control-plane -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/lib/etcd/backup.db
```

**Passo 4 — Verificar o backup:**

```bash
kubectl exec -n kube-system etcd-prod-ops-lab-control-plane -- etcdctl \
  snapshot status /var/lib/etcd/backup.db --write-out=table
```

Você deve ver uma tabela com hash, revisão, total de chaves e tamanho total — confirmando um backup válido.

**Passo 5 — Verificar o que está armazenado no etcd:**

```bash
# Contar as chaves (dá uma noção do tamanho do estado do cluster)
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
# Listar todos os webapps (observe as colunas customizadas do additionalPrinterColumns)
kubectl get webapps

# Usar o nome curto
kubectl get wa

# Descrever um
kubectl describe webapp frontend

# Obter como YAML
kubectl get webapp api-service -o yaml
```

**Passo 5 — Limpar recursos do CRD:**

```bash
kubectl delete webapp --all
kubectl delete crd webapps.lab.example.com
```

---

### Lab 4: Simular Fluxo de Upgrade do Cluster

Embora não possamos realmente fazer upgrade do Kind (é uma versão fixa), podemos percorrer o fluxo exato:

**Passo 1 — Verificar versões atuais:**

```bash
kubectl get nodes -o wide
kubectl version
```

**Passo 2 — Revisar o plano de upgrade (conceitual):**

```bash
# Em um cluster kubeadm real, você executaria:
# sudo kubeadm upgrade plan
# Isso mostra versões disponíveis, ações necessárias e avisos de depreciação

# Em nosso lab, vamos pelo menos verificar o que o kubeadm reportaria
echo "=== Current Cluster Version ==="
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kubeletVersion}{"\n"}{end}'
```

**Passo 3 — Praticar o ciclo drain/uncordon:**

```bash
# Implantar um workload
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

# Obter um worker node
WORKER=$(kubectl get nodes --selector='!node-role.kubernetes.io/control-plane' -o jsonpath='{.items[0].metadata.name}')

# Passo A: Drain (simulando pré-upgrade)
echo "Draining $WORKER..."
kubectl drain $WORKER --ignore-daemonsets --delete-emptydir-data

# Verificar que os pods foram movidos
kubectl get pods -l app=upgrade-test -o wide
echo "All pods moved off $WORKER"

# Passo B: Uncordon (simulando pós-upgrade)
echo "Uncordoning $WORKER..."
kubectl uncordon $WORKER

# Passo C: Verificar que o nó está de volta em rotação
kubectl get nodes
```

**Passo 4 — Documentar os passos reais de upgrade:**

```bash
echo "
=== PRODUCTION UPGRADE CHECKLIST ===
1. Read release notes for target version
2. Backup etcd: etcdctl snapshot save
3. Upgrade kubeadm on first control plane node
4. Run: kubeadm upgrade plan
5. Run: kubeadm upgrade apply v1.XX.Y
6. Upgrade kubelet and kubectl on control plane
7. For each worker node:
   a. kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
   b. Upgrade kubeadm, then run: kubeadm upgrade node
   c. Upgrade kubelet and kubectl
   d. systemctl daemon-reload && systemctl restart kubelet
   e. kubectl uncordon <node>
8. Verify: kubectl get nodes (all Ready, correct version)
"
```

**Limpeza:**

```bash
kubectl delete deployment upgrade-test
```

---

### Limpeza do Laboratório

```bash
# Deletar o cluster Kind
kind delete cluster --name prod-ops-lab
```

---

## Principais Conclusões

1. **Faça upgrade do control plane primeiro, depois dos workers.** Nunca faça upgrade dos workers antes do control plane. Siga a política de version skew: kubelet pode ficar até 2 versões minor atrás do API server, mas nunca à frente.

2. **Sempre drene antes da manutenção.** `kubectl drain` remove pods graciosamente e respeita PodDisruptionBudgets. Nunca faça manutenção de nó (upgrades, hardware) sem drenar primeiro.

3. **etcd é o cérebro do seu cluster — faça backup.** Todo o estado do Kubernetes vive no etcd. Agende snapshots regulares com `etcdctl snapshot save`. Armazene backups fora do cluster. Teste seu procedimento de restore antes de precisar dele.

4. **Recuperação de desastres requer múltiplos alvos de backup.** etcd cobre o estado do cluster, mas PersistentVolumes precisam de CSI snapshots, e estado externo (DNS, certificados, IAM) precisa de Infrastructure-as-Code. Nenhuma ferramenta única faz backup de tudo.

5. **CRDs estendem a API do Kubernetes.** Custom Resource Definitions permitem adicionar seus próprios tipos de recursos que funcionam com kubectl, RBAC e toda a maquinaria da API do Kubernetes.

6. **Operators automatizam operações Day-2.** Operators são controllers customizados que codificam expertise operacional (backup, failover, scaling) em software que reage em tempo real — não cron jobs periódicos.

7. **Kubernetes gerenciado cuida das partes difíceis.** AKS, EKS e GKE gerenciam o control plane, etcd e upgrades. Você foca em node pools e workloads. Para produção, gerenciado é quase sempre a escolha certa a menos que você tenha requisitos específicos.

---

## Leitura Adicional

- [Upgrading kubeadm Clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
- [Backing Up an etcd Cluster](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)
- [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [Operator Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)
- [Velero Documentation](https://velero.io/docs/)
- [OperatorHub.io](https://operatorhub.io/)

---

**Anterior:** [Capítulo 13 — Solução de Problemas](13-troubleshooting.md)
**Próximo:** [Capítulo 15 — Próximos Passos](15-next-steps.md)
