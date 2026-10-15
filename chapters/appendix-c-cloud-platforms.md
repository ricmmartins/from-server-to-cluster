# Apêndice C: Comparação de Plataformas Cloud

*Um guia prático para escolher e usar Kubernetes gerenciado nos principais provedores de cloud.*

---

## Por que Kubernetes Gerenciado?

Executar Kubernetes por conta própria significa operar etcd, o API server, controller-manager e scheduler — todos os quais devem ter alta disponibilidade, patches aplicados e backups. Serviços gerenciados eliminam essa sobrecarga operacional:

- **Control plane gerenciado para você** — o provedor cuida de upgrades, patches, HA e backups do etcd.
- **Cobertura de SLA condicionada** — depende da camada e da topologia; o AKS Free não tem SLA de uptime com respaldo financeiro. Um SLA do control plane não garante a disponibilidade da aplicação.
- **Integrações profundas** — IAM, rede, armazenamento, monitoramento e serviços de marketplace funcionam nativamente.
- **Tempo mais rápido para produção** — crie um cluster de nível produção em minutos, não em dias.

Você ainda gerencia suas cargas de trabalho, node pools, políticas de rede e questões no nível da aplicação — mas o trabalho pesado indiferenciado do control plane sai da sua responsabilidade.

---

## Tabela de Comparação Rápida

**Revisão da documentação: 2026-10-15.** Estas são opções de configuração, não padrões ou limites de capacidade certificados. Padrões, cotas, maturidade dos addons, disponibilidade de máquinas e preços variam conforme a região, a conta, o modo do cluster e a versão; confirme-os antes do provisionamento. Não foram realizadas consultas autenticadas de disponibilidade regional nem criação de recursos cloud nesta revisão.

| Recurso | AKS (Azure) | EKS (AWS) | GKE (Google Cloud) |
|---------|-------------|-----------|---------------------|
| **Ferramenta CLI** | `az aks` | `eksctl` / `aws eks` | `gcloud container clusters` |
| **Custo de gerenciamento do cluster (sem cargas de trabalho)** | Camada Free: sem taxa de gerenciamento e sem SLA de uptime; Standard/Premium: camadas pagas (veja os preços abaixo) | US$ 0,10/cluster-hora no suporte padrão; US$ 0,60/cluster-hora no suporte estendido | Taxa base de US$ 0,10/cluster-hora; crédito de gratuidade condicionado; acréscimo por suporte estendido |
| **Opção de rede** | Azure CNI Overlay | AWS VPC CNI | VPC-native (alias IPs) |
| **Opção de load balancer L4** | Azure Load Balancer | AWS Network Load Balancer (depende do controller/modo) | Google Cloud Load Balancer |
| **Implementação de roteamento L7** | Application Gateway for Containers | AWS Load Balancer Controller (ALB) | GKE Gateway Controller |
| **Identidade/Workload Identity** | Entra ID Workload Identity | IAM Roles for Service Accounts (IRSA) / Pod Identity | Workload Identity Federation |
| **Integração de secrets** | Azure Key Vault CSI driver | AWS Secrets Manager CSI driver | Secret Manager CSI driver |
| **Monitoramento** | Azure Monitor Container Insights | Amazon CloudWatch Container Insights | Google Cloud Monitoring + dashboards GKE |
| **Opções de SO dos nós** | Azure Linux, Ubuntu | Amazon Linux 2023, Bottlerocket, Ubuntu | Container-Optimized OS, Ubuntu |
| **Modo autopilot/serverless** | AKS Automatic | EKS Auto Mode | GKE Autopilot |
| **Densidade de Pods** | Verifique o modo de rede e `maxPods` | Verifique os limites de ENI/IP da instância, a delegação de prefixos e a configuração do kubelet | Verifique o modo do cluster e as configurações do node pool |
| **Política de suporte de versão** | Calendário de suporte da comunidade; LTS opcional na camada Premium, habilitado explicitamente | 14 meses de suporte padrão + 12 meses de suporte estendido pago | Ciclo de vida por canal de lançamento; o canal Extended tem condições próprias de suporte e preços |
| **Versões Kubernetes disponíveis** | Execute `az aks get-versions` para a região | Execute `aws eks describe-cluster-versions` para a região | Execute `gcloud container get-server-config` para a região/canal |
| **Opções de escalonamento de nós** | Cluster Autoscaler / provisionamento automático de nós (verifique o suporte atual) | Karpenter / Cluster Autoscaler / Auto Mode | GKE cluster autoscaler / provisionamento automático de nós |

---

## Criação de Cluster

A **base didática local do livro é Kubernetes 1.35.0 no kind 0.31.0 com kubectl 1.35.x**. Os serviços gerenciados não necessariamente oferecem o patch upstream `1.35.0`; escolha uma versão minor/patch atualmente suportada na região e verifique a compatibilidade de versões entre cliente e servidor (o kubectl suporta diferença de uma minor para mais ou para menos em relação ao API server). Prefira uma versão 1.35 disponível quando for adequada, não uma versão 1.32 obsoleta fixada nem um patch `.0` presumido.

Os comandos abaixo são **modelos Bash não executados**, não evidências de implantação. Eles pressupõem CLIs instaladas e autenticadas, permissões, cotas, pré-requisitos de rede/IAM e aprovação para cobrança. Primeiro execute cada comando de consulta somente leitura; selecione uma versão suportada e verifique tipos de máquina, zonas, cotas, compatibilidade de controllers/CSI e camada de suporte antes de executar qualquer comando de criação que gere cobrança. A disponibilidade da documentação não comprova que uma assinatura consiga criar essa configuração.

### Azure (AKS)

```bash
# Read-only discovery; choose a version offered in this region.
az aks get-versions --location eastus --output table
read -r -p "Supported regional AKS version: " AKS_VERSION
: "${AKS_VERSION:?Select a supported regional version first}"

# Create a resource group
az group create --name myResourceGroup --location eastus

# Create an AKS cluster
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --tier standard \
  --node-count 3 \
  --node-vm-size Standard_DS3_v2 \
  --kubernetes-version "$AKS_VERSION" \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --enable-managed-identity \
  --generate-ssh-keys

# Get credentials for kubectl
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster

# Verify connectivity
kubectl get nodes
```

**Opções principais:**
- `--tier standard` — gerenciamento pago do cluster com SLA de uptime; os recursos das cargas de trabalho são cobrados separadamente. Use `--tier free` apenas quando as condições sem SLA forem adequadas à carga de trabalho.
- **Não reutilize Kubernetes 1.32 como exemplo de suporte padrão.** Seu suporte da comunidade no AKS terminou em março de 2026. O calendário LTS publicado vai até março de 2027, mas o LTS exige **tanto** `--tier premium` **quanto** `--k8s-support-plan AKSLongTermSupport`; não é automático. Ainda é necessário consultar os patches suportados e a elegibilidade para criação na região. A criação de um cluster AKS 1.32 LTS em qualquer região específica **não foi demonstrada**.
- `--enable-addons monitoring` — habilitar Container Insights na criação.
- `--os-sku AzureLinux3` — escolha uma imagem Azure Linux suportada após verificar a compatibilidade de versão/região; não selecione o Azure Linux 2.0 descontinuado.
- `--enable-oidc-issuer --enable-workload-identity` — habilitar Workload Identity.

### AWS (EKS)

```bash
# Read-only discovery; inspect standard/extended support dates and status.
aws eks describe-cluster-versions --region us-west-2
read -r -p "Supported regional EKS minor version: " EKS_VERSION
: "${EKS_VERSION:?Select a supported regional version first}"

# Create a cluster with managed node group
eksctl create cluster \
  --name myEKSCluster \
  --version "$EKS_VERSION" \
  --region us-west-2 \
  --nodegroup-name linux-nodes \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 5 \
  --managed

# eksctl automatically updates kubeconfig
kubectl get nodes
```

**Alternativa: apenas AWS CLI**
```bash
aws eks create-cluster \
  --name myEKSCluster \
  --region us-west-2 \
  --kubernetes-version "${EKS_VERSION:?Run regional discovery first}" \
  --role-arn arn:aws:iam::ACCOUNT:role/eks-cluster-role \
  --resources-vpc-config subnetIds=subnet-xxx,subnet-yyy,securityGroupIds=sg-zzz

# Wait for the control plane before creating the managed node group.
aws eks wait cluster-active --name myEKSCluster --region us-west-2
aws eks create-nodegroup \
  --cluster-name myEKSCluster \
  --region us-west-2 \
  --nodegroup-name linux-nodes \
  --node-role arn:aws:iam::ACCOUNT:role/eks-node-role \
  --subnets subnet-xxx subnet-yyy \
  --instance-types t3.medium \
  --scaling-config minSize=2,maxSize=5,desiredSize=3
```

A alternativa com AWS CLI pressupõe roles IAM existentes para cluster/nós, sub-redes e security groups corretamente configurados e permissões suficientes; não é uma configuração inicial completa de VPC/IAM.

### GCP (GKE)

```bash
# Read-only discovery; inspect versions valid for the selected release channel.
gcloud container get-server-config --region us-central1
read -r -p "Supported regional GKE version in the regular channel: " GKE_VERSION
: "${GKE_VERSION:?Select a supported regional version first}"

# Standard regional cluster: one initial node per explicitly selected zone.
gcloud container clusters create my-gke-cluster \
  --region us-central1 \
  --node-locations us-central1-a,us-central1-b,us-central1-c \
  --num-nodes 1 \
  --machine-type e2-standard-4 \
  --release-channel regular \
  --cluster-version "$GKE_VERSION" \
  --enable-ip-alias \
  --workload-pool=PROJECT_ID.svc.id.goog

# Get credentials
gcloud container clusters get-credentials my-gke-cluster --region us-central1

kubectl get nodes
```

Os nomes do GKE usam letras minúsculas, dígitos e hífens, começam com uma letra e terminam com uma letra ou dígito. Substitua `PROJECT_ID` pelo ID do seu projeto e prepare antes a conta de serviço dos nós e as permissões IAM. Em um pool regional com várias zonas, `--num-nodes` é **por zona**: o exemplo começa com **1 × 3 = 3 nós**, enquanto `--num-nodes 3` nessas zonas iniciaria **9 nós**, não 3. Verifique a capacidade das zonas e as cotas; escalonamentos/upgrades posteriores podem alterar a quantidade.

**GKE Autopilot (avalie antes as restrições das cargas de trabalho e o modelo de cobrança):**
```bash
# Run regional discovery above first; the channel selects an offered version.
gcloud container clusters create-auto my-autopilot-cluster \
  --region us-central1 \
  --release-channel regular

gcloud container clusters get-credentials my-autopilot-cluster --region us-central1
```

---

## Armazenamento

| Recurso | AKS (Azure) | EKS (AWS) | GKE (Google Cloud) |
|---------|-------------|-----------|---------------------|
| **Driver CSI (bloco)** | Azure Disk CSI (`disk.csi.azure.com`) | EBS CSI (`ebs.csi.aws.com`); Auto Mode usa `ebs.csi.eks.amazonaws.com` | Persistent Disk CSI (`pd.csi.storage.gke.io`) |
| **Driver CSI (arquivo/NFS)** | Azure Files CSI (`file.csi.azure.com`) | EFS CSI (`efs.csi.aws.com`) | Filestore CSI (`filestore.csi.storage.gke.io`) |
| **Exemplos de StorageClass (inspecione os padrões)** | `managed-csi` (inspecione a redundância/SKU real) | Não há padrão `gp3` universal; defina/inspecione a classe do provisionador selecionado | `standard-rwo` (PD balanced; inspecione o padrão do cluster) |
| **Opção de armazenamento de maior desempenho** | `managed-csi-premium` (Premium SSD) | Uma StorageClass configurada separadamente com tipo EBS `io2` | `premium-rwo` (PD SSD) |
| **Suporte ReadWriteMany** | Azure Files (SMB/NFS) | EFS (NFS) | Filestore (NFS) |
| **Volume snapshots** | Suportado (Azure Disk CSI) | Suportado (EBS CSI) | Suportado (PD CSI) |
| **Expansão de volume** | Redimensionamento online suportado | Redimensionamento online suportado | Redimensionamento online suportado |
| **Limites de anexação de volumes** | Verifique os limites do SKU da VM/CSI | Verifique os limites do tipo de instância/EBS/CSI | Verifique os limites do tipo de máquina/CSI |

Inspecione `kubectl get storageclass` e `kubectl get csidriver` antes de escolher `storageClassName`; os nomes das classes são configuração, não uma garantia da API. O EKS padrão precisa do addon/driver EBS CSI e de permissões IAM (EKS Pod Identity ou IRSA); a falta de permissões causa falhas de provisionamento. O Auto Mode gerencia seu próprio controller de armazenamento, mas **não** cria uma StorageClass para você: defina uma com o provisionador `ebs.csi.eks.amazonaws.com`, o tipo de disco desejado (por exemplo, `gp3`) e configurações adequadas de topologia e vinculação, e verifique as permissões de armazenamento em bloco da role IAM do cluster. Volumes do CSI padrão e do Auto Mode são gerenciados separadamente; a migração exige um fluxo de snapshots suportado. Snapshots exigem o snapshot controller apropriado. Consulte as referências oficiais de armazenamento abaixo.

---

## Rede

| Recurso | AKS (Azure) | EKS (AWS) | GKE (Google Cloud) |
|---------|-------------|-----------|---------------------|
| **Opção de rede** | Azure CNI Overlay | AWS VPC CNI | VPC-native (alias IPs) |
| **CNI alternativo** | Azure CNI + Cilium (powered by Cilium) | Calico, Cilium | Calico (para network policy), Cilium (via Dataplane V2) |
| **Rede de Pods** | Overlay CIDR (separado da VNet) | IPs flat VPC (baseado em ENI) | Alias IP ranges (CIDRs secundários) |
| **Engine de network policy (verifique se está habilitada)** | Azure CNI powered by Cilium / outras configurações de política suportadas | VPC CNI Network Policy / alternativas compatíveis | Dataplane V2 ou uma configuração suportada com políticas habilitadas |
| **Load Balancer L4** | Azure Load Balancer (Standard) | Network Load Balancer (NLB) | Google Cloud TCP/UDP Load Balancer |
| **Ingress L7 (gerenciado)** | Application Gateway for Containers | AWS ALB (via Load Balancer Controller) | GKE Gateway Controller / Ingress for HTTP(S) |
| **Implementação de Gateway API** | Application Gateway for Containers | Selecione um controller suportado pela AWS e verifique seu suporte às APIs/funcionalidades | GKE Gateway Controller |
| **Opção de service mesh** | Addon de service mesh baseado em Istio | Istio no EKS (autogerenciado; verifique compatibilidade/suporte) | Cloud Service Mesh |
| **Load balancer interno** | Annotation: `service.beta.kubernetes.io/azure-load-balancer-internal: "true"` | Annotation: `service.beta.kubernetes.io/aws-load-balancer-internal: "true"` | Annotation: `networking.gke.io/load-balancer-type: "Internal"` |
| **DNS** | CoreDNS (padrão) | CoreDNS (padrão) | kube-dns / CoreDNS |

Objetos NetworkPolicy precisam de uma implementação habilitada que aplique as políticas; recursos com escopo de namespace, por si só, não fornecem isolamento de rede. Annotations L4/L7 e suporte a Gateway API são específicos de cada controller/versão, portanto valide-os conforme a implementação escolhida. **O AWS App Mesh não é mais uma opção atual:** a AWS encerrou o suporte e o acesso aos recursos e ao console em **2026-09-30**.

---

## Segurança

| Recurso | AKS (Azure) | EKS (AWS) | GKE (Google Cloud) |
|---------|-------------|-----------|---------------------|
| **Provedor de identidade** | Microsoft Entra ID | AWS IAM | Google Cloud IAM |
| **Autenticação workload-para-cloud** | Workload Identity (tokens federados Entra ID) | IRSA / EKS Pod Identity | Workload Identity Federation |
| **Integração RBAC** | Grupos Entra ID → K8s RBAC | IAM roles mapeadas para K8s RBAC via ConfigMap `aws-auth` ou access entries | Grupos Google → K8s RBAC |
| **Gerenciamento de secrets** | Azure Key Vault (Secrets Store CSI) | AWS Secrets Manager / SSM Parameter Store (CSI) | Secret Manager (CSI) |
| **Assinatura/verificação de imagens** | Notation + Azure Policy | ECR image scanning + Sigstore | Binary Authorization |
| **Segurança de Pod** | Pod Security Admission (nativo) + Azure Policy | Pod Security Admission (nativo) | Pod Security Admission + GKE Policy Controller |
| **Segurança dos nós** | Azure Linux / Mariner (hardened) | Bottlerocket (hardened, imutável) | Container-Optimized OS (hardened, mínimo) |
| **Criptografia em repouso** | Etcd criptografado por padrão; BYOK via Azure Key Vault | Etcd criptografado por padrão; BYOK via KMS envelope encryption | Etcd criptografado por padrão; BYOK via Cloud KMS |
| **Cluster privado** | Private cluster (API server em private endpoint) | Private endpoint para API server | Private cluster (control plane interno à VPC) |

---

## Monitoramento

| Recurso | AKS (Azure) | EKS (AWS) | GKE (Google Cloud) |
|---------|-------------|-----------|---------------------|
| **Monitoramento nativo** | Azure Monitor Container Insights | Amazon CloudWatch Container Insights | Google Cloud Monitoring (dashboards GKE) |
| **Coleta de métricas** | Managed Prometheus (Azure Monitor) | Amazon Managed Service for Prometheus | Google Cloud Managed Service for Prometheus |
| **Agregação de logs** | Azure Monitor Logs (Log Analytics) | CloudWatch Logs + FluentBit | Cloud Logging |
| **Dashboards** | Azure Monitor workbooks + Grafana (gerenciado) | CloudWatch dashboards + Amazon Managed Grafana | Cloud Monitoring dashboards + Google Cloud Managed Grafana |
| **Alertas** | Azure Monitor alerts | CloudWatch Alarms + SNS | Cloud Monitoring alerting policies |
| **Rastreamento** | Azure Monitor Application Insights | AWS X-Ray | Cloud Trace |
| **Análise de custos** | AKS cost analysis (nativo) | AWS Cost Explorer + Kubecost | GKE cost allocation (nativo) |
| **Compatível com Prometheus** | Sim (Azure Managed Prometheus) | Sim (Amazon Managed Prometheus - AMP) | Sim (Google Managed Prometheus - GMP) |

---

## Quando Escolher Qual

### Matriz de Decisão

| Prioridade | Melhor Escolha | Por quê |
|----------|------------|-----|
| **Sem taxa de gerenciamento de cluster para laboratório** | Avalie o AKS Free | Sem SLA de uptime; nós, armazenamento, rede e monitoramento continuam gerando custos. Isso não comprova o menor custo total |
| **Ecossistema AWS existente** | EKS | Integração nativa com IAM, VPC, ALB, CloudWatch e mais de 200 serviços AWS |
| **Ecossistema Azure existente** | AKS | Integração nativa com Entra ID, Azure DevOps, App Gateway, Azure Monitor |
| **Ecossistema GCP existente** | GKE | Integração nativa com Cloud IAM, Cloud Run, Cloud Build, BigQuery |
| **Experiência Kubernetes-nativa** | GKE | Google criou o Kubernetes; GKE frequentemente tem os recursos mais novos primeiro |
| **Menos trabalho de gerenciamento de nós** | Avalie o GKE Autopilot | Ciclo de vida dos nós gerenciado; cobrança de computação por Pod ou por nó depende da carga de trabalho/classe de computação |
| **Híbrido/multi-cloud** | AKS (Azure Arc) ou EKS Anywhere | Arc suporta K8s em qualquer lugar; EKS Anywhere roda on-premises |
| **Containers Windows** | AKS | Suporte mais maduro a node pools Windows; clusters mistos Linux/Windows |
| **Cargas de trabalho AI/ML (GPU)** | Todos os três | Todos suportam node pools GPU; GKE tem TPU; AKS tem AMD MI300X; EKS tem Inferentia/Trainium |
| **Indústrias com compliance rigoroso** | Todos os três | Todos possuem FedRAMP, HIPAA, SOC2; avalie certificações específicas necessárias |
| **Expertise da equipe** | Corresponda às habilidades cloud existentes | O provedor que sua equipe já conhece trará resultados mais rápidos |

### Considerações de Custo Além do Control Plane

O gerenciamento do cluster e os recursos das cargas de trabalho são cobrados separadamente; a participação de cada um no total depende da carga de trabalho. Planeje o orçamento para:
- **Computação** (VMs/instâncias dos nós ou recursos faturáveis do Autopilot).
- **Armazenamento** (discos persistentes, file shares) — varia por carga de trabalho.
- **Rede** (load balancers, NAT gateways, tráfego cross-AZ, egress).
- **Monitoramento** (ingestão de logs, armazenamento de métricas, alertas).

Descontos por compromisso de uso e capacidade Spot têm condições de elegibilidade, compromisso e interrupção. Compare estimativas dos provedores para a carga de trabalho real, em vez de presumir um desconto fixo.

### Preços e Condições da Camada Gratuita — Consultados em 2026-10-15

Os exemplos abaixo usam preços públicos de tabela em dólares americanos e cobrem **apenas taxas de gerenciamento**, não cotações para uma conta. Use as páginas oficiais de preços/calculadoras nos links para sua região, moeda, contrato, impostos e funcionalidades adicionais. Um **mês ilustrativo de 730 horas** não representa todos os meses do calendário.

| Serviço | Cobranças de gerenciamento/suporte | Cobranças de cargas de trabalho e condições de gratuidade |
|---------|-----------------------------------|---------------------------------------------------------|
| AKS Free | Sem taxa de gerenciamento de cluster; **sem SLA de uptime com respaldo financeiro** | VMs, discos, rede e outros serviços subjacentes continuam sendo cobrados; "Free" não significa que a carga de trabalho do cluster seja gratuita |
| AKS Standard / Premium | Camadas pagas de gerenciamento; Standard/Premium incluem SLA de uptime do API server (99,95% com zonas de disponibilidade, 99,9% sem elas), sujeito às condições. Premium com adesão explícita ao LTS oferece suporte estendido de versão | Recursos das cargas de trabalho cobrados separadamente. O conteúdo público de preços consultado não apresentou um valor numérico confiável para as camadas; consulte a calculadora regional oficial em vez de presumir uma tarifa |
| EKS com suporte padrão | **US$ 0,10/cluster-hora → US$ 73 por 730 horas** | EC2/Fargate, EBS, load balancers, rede e observabilidade são adicionais; Auto Mode tem uma cobrança adicional de gerenciamento pela capacidade EC2 gerenciada. Não presuma que créditos promocionais da AWS eliminem a taxa do cluster |
| EKS com suporte estendido | **US$ 0,60/cluster-hora no total → US$ 438 por 730 horas**, não US$ 0,10 + US$ 0,60 | O status de suporte determina a taxa: na data desta revisão, EKS **1.35** está em suporte padrão (termina em 2027-03-27); **1.32** entrou em suporte estendido em 2026-03-23 (termina em 2027-03-23). Verifique novamente o ciclo de vida e a região antes da seleção |
| Gerenciamento base do GKE | **US$ 0,10/cluster-hora → US$ 73 por 730 horas**, antes dos créditos elegíveis | O crédito de **US$ 74,40/mês por conta de faturamento** aplica-se apenas às taxas de gerenciamento de clusters Autopilot e Standard zonais; não às taxas do Standard regional, computação ou outros SKUs. Créditos não usados não se acumulam; não são uma franquia por cluster |
| GKE com suporte estendido | Acréscimo do período estendido de **US$ 0,50/cluster-hora**, mais a base de US$ 0,10 = **US$ 0,60/hora → US$ 438 por 730 horas** antes dos créditos aplicáveis | Aplica-se quando a versão minor está no período de suporte estendido; verifique a adesão ao canal Extended, o ciclo de vida e o tratamento exato dos créditos |
| Computação do GKE Autopilot | Separada da taxa de gerenciamento do cluster | Classes elegíveis de computação de uso geral usam **cobrança por Pod** para CPU, memória e armazenamento efêmero solicitados (após ajustes de padrões/mínimos). Classes/cargas de trabalho com hardware específico usam **cobrança por nó** pela VM inteira, mais um adicional de gerenciamento do Autopilot; não descreva todo Autopilot como pagamento por Pod |

Por exemplo, a taxa base de gerenciamento de US$ 73 de um único cluster GKE Standard zonal ou Autopilot elegível pode ser coberta por um crédito de US$ 74,40 da conta de faturamento que não tenha sido usado por outros recursos; o cluster Standard regional acima continua gerando US$ 73 de cobrança, e a computação não é coberta em nenhum dos casos. Os exemplos do EKS pressupõem ausência de capacidade/funcionalidades adicionais pagas do control plane.

### Fontes Oficiais e Verificações Pendentes

Todos os links abaixo foram consultados em **2026-10-15**; a documentação pública confirma as políticas descritas, não o provisionamento bem-sucedido em uma conta:

- **Azure:** [preços do AKS](https://azure.microsoft.com/en-us/pricing/details/kubernetes-service/), [Free/Standard/Premium e condições de SLA](https://learn.microsoft.com/en-us/azure/aks/free-standard-pricing-tiers), [calendário de versões/LTS](https://learn.microsoft.com/en-us/azure/aks/supported-kubernetes-versions), [adesão explícita ao Premium LTS](https://learn.microsoft.com/en-us/azure/aks/long-term-support) e [consulta via Azure CLI](https://learn.microsoft.com/en-us/cli/azure/aks#az-aks-get-versions).
- **AWS:** [preços do EKS](https://aws.amazon.com/eks/pricing/), [ciclo de vida e consulta de versões](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html), [provisionadores EBS CSI e IAM](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html), [StorageClasses do Auto Mode](https://docs.aws.amazon.com/eks/latest/userguide/create-storage-class.html) e [aviso de fim de suporte do App Mesh](https://docs.aws.amazon.com/app-mesh/latest/userguide/what-is-app-mesh.html).
- **Google Cloud:** [preços do GKE, camada gratuita, cobrança do Autopilot e suporte estendido](https://cloud.google.com/kubernetes-engine/pricing), [criação de clusters regionais](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-regional-cluster), [comando create e semântica da contagem de nós](https://cloud.google.com/sdk/gcloud/reference/container/clusters/create) e [consulta regional de versões](https://cloud.google.com/sdk/gcloud/reference/container/get-server-config).

**Ainda exige verificação na conta/região:** patches exatos disponíveis e elegibilidade para novos clusters (especialmente versões antigas do AKS LTS), capacidade de SKUs/zonas e cotas, permissões IAM, padrões atuais de StorageClass, aplicação de políticas, suporte a addons/controllers e preços finais de cargas de trabalho/camadas. As tabelas de funcionalidades servem de orientação, não de certificação de limites, elegibilidade de conformidade ou implantação funcional. Nenhuma alteração em recursos cloud nem instalação de ferramentas foi executada.

---

*Navegação: [← Apêndice B: Referência Rápida do kubectl](appendix-b-cheatsheet.md)*
