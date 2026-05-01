# Apêndice C: Comparação de Plataformas Cloud

*Um guia prático para escolher e usar Kubernetes gerenciado nos principais provedores de cloud.*

---

## Por que Kubernetes Gerenciado?

Executar Kubernetes por conta própria significa operar etcd, o API server, controller-manager e scheduler — todos os quais devem ter alta disponibilidade, patches aplicados e backups. Serviços gerenciados eliminam essa sobrecarga operacional:

- **Control plane gerenciado para você** — o provedor cuida de upgrades, patches, HA e backups do etcd.
- **Disponibilidade com SLA** — garantias de uptime com respaldo financeiro (tipicamente 99,5%–99,95%).
- **Integrações profundas** — IAM, rede, armazenamento, monitoramento e serviços de marketplace funcionam nativamente.
- **Tempo mais rápido para produção** — crie um cluster de nível produção em minutos, não em dias.

Você ainda gerencia suas cargas de trabalho, node pools, políticas de rede e questões no nível da aplicação — mas o trabalho pesado indiferenciado do control plane sai da sua responsabilidade.

---

## Tabela de Comparação Rápida

| Recurso | AKS (Azure) | EKS (AWS) | GKE (Google Cloud) |
|---------|-------------|-----------|---------------------|
| **Ferramenta CLI** | `az aks` | `eksctl` / `aws eks` | `gcloud container clusters` |
| **Custo do control plane** | Gratuito | ~$73/mês ($0,10/hr) | Standard: $0,10/hr; Autopilot: $0,10/hr (computação cobrada por pod) |
| **CNI padrão** | Azure CNI Overlay | AWS VPC CNI | VPC-native (alias IPs) |
| **Load balancer padrão** | Azure Load Balancer | AWS Network Load Balancer | Google Cloud Load Balancer |
| **Ingress gerenciado** | Application Gateway for Containers | AWS ALB Ingress Controller | GKE Gateway Controller |
| **Identidade/Workload Identity** | Entra ID Workload Identity | IAM Roles for Service Accounts (IRSA) / Pod Identity | Workload Identity Federation |
| **Integração de secrets** | Azure Key Vault CSI driver | AWS Secrets Manager CSI driver | Secret Manager CSI driver |
| **Monitoramento** | Azure Monitor Container Insights | Amazon CloudWatch Container Insights | Google Cloud Monitoring + dashboards GKE |
| **Opções de SO dos nós** | Azure Linux, Ubuntu | Amazon Linux 2023, Bottlerocket, Ubuntu | Container-Optimized OS, Ubuntu |
| **Modo autopilot/serverless** | AKS Automatic | EKS Auto Mode | GKE Autopilot |
| **Máx. pods por nó (padrão)** | 250 (Azure CNI Overlay) | Varia por tipo de instância (17–110; 110 com prefix delegation) | 110 |
| **Política de suporte de versão** | N-2 (atual + 2 minors anteriores) | 14 meses padrão + 12 meses estendido | 3 minors mais recentes |
| **Versões K8s suportadas (meados 2025)** | 1.30 – 1.32 | 1.30 – 1.32 (1.29 no estendido) | 1.30 – 1.32 |
| **Cluster autoscaler** | Cluster Autoscaler / Karpenter (preview) | Karpenter (recomendado) / Cluster Autoscaler | GKE Autoscaler (nativo) / Karpenter (preview) |

---

## Criação de Cluster

### Azure (AKS)

```bash
# Instalar/atualizar Azure CLI
az upgrade

# Criar um resource group
az group create --name myResourceGroup --location eastus

# Criar um cluster AKS
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --node-count 3 \
  --node-vm-size Standard_DS3_v2 \
  --kubernetes-version 1.32 \
  --network-plugin azure \
  --enable-managed-identity \
  --generate-ssh-keys

# Obter credenciais para kubectl
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster

# Verificar conectividade
kubectl get nodes
```

**Opções principais:**
- `--enable-addons monitoring` — habilitar Container Insights na criação.
- `--os-sku AzureLinux` — usar Azure Linux (recomendado para novos clusters).
- `--enable-oidc-issuer --enable-workload-identity` — habilitar Workload Identity.

### AWS (EKS)

```bash
# Instalar eksctl (https://eksctl.io)
# Criar um cluster com managed node group
eksctl create cluster \
  --name myEKSCluster \
  --version 1.32 \
  --region us-west-2 \
  --nodegroup-name linux-nodes \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 5 \
  --managed

# eksctl atualiza automaticamente o kubeconfig
kubectl get nodes
```

**Alternativa: apenas AWS CLI**
```bash
aws eks create-cluster \
  --name myEKSCluster \
  --kubernetes-version 1.32 \
  --role-arn arn:aws:iam::ACCOUNT:role/eks-cluster-role \
  --resources-vpc-config subnetIds=subnet-xxx,subnet-yyy,securityGroupIds=sg-zzz

# Depois crie um managed node group separadamente
aws eks create-nodegroup \
  --cluster-name myEKSCluster \
  --nodegroup-name linux-nodes \
  --node-role arn:aws:iam::ACCOUNT:role/eks-node-role \
  --subnets subnet-xxx subnet-yyy \
  --instance-types t3.medium \
  --scaling-config minSize=2,maxSize=5,desiredSize=3
```

### GCP (GKE)

```bash
# Cluster Standard
gcloud container clusters create myGKECluster \
  --region us-central1 \
  --num-nodes 3 \
  --machine-type e2-standard-4 \
  --cluster-version 1.32 \
  --enable-ip-alias \
  --workload-pool=PROJECT_ID.svc.id.goog

# Obter credenciais
gcloud container clusters get-credentials myGKECluster --region us-central1

kubectl get nodes
```

**GKE Autopilot (recomendado para a maioria das cargas de trabalho):**
```bash
gcloud container clusters create-auto myAutopilotCluster \
  --region us-central1 \
  --release-channel regular

gcloud container clusters get-credentials myAutopilotCluster --region us-central1
```

---

## Armazenamento

| Recurso | AKS (Azure) | EKS (AWS) | GKE (Google Cloud) |
|---------|-------------|-----------|---------------------|
| **Driver CSI (bloco)** | Azure Disk CSI (`disk.csi.azure.com`) | EBS CSI (`ebs.csi.aws.com`) | Persistent Disk CSI (`pd.csi.storage.gke.io`) |
| **Driver CSI (arquivo/NFS)** | Azure Files CSI (`file.csi.azure.com`) | EFS CSI (`efs.csi.aws.com`) | Filestore CSI (`filestore.csi.storage.gke.io`) |
| **StorageClass padrão** | `managed-csi` (Azure Disk LRS) | `gp3` (EBS gp3) | `standard-rwo` (PD balanced) |
| **Storage class premium** | `managed-csi-premium` (Premium SSD) | `io2` (Provisioned IOPS) | `premium-rwo` (PD SSD) |
| **Suporte ReadWriteMany** | Azure Files (SMB/NFS) | EFS (NFS) | Filestore (NFS) |
| **Volume snapshots** | Suportado (Azure Disk CSI) | Suportado (EBS CSI) | Suportado (PD CSI) |
| **Expansão de volume** | Redimensionamento online suportado | Redimensionamento online suportado | Redimensionamento online suportado |
| **Máx. volumes por nó** | ~64 (depende do tamanho da VM) | Varia por tipo de instância (limites EBS) | 128 PDs por nó |

---

## Rede

| Recurso | AKS (Azure) | EKS (AWS) | GKE (Google Cloud) |
|---------|-------------|-----------|---------------------|
| **CNI padrão** | Azure CNI Overlay | AWS VPC CNI | VPC-native (alias IPs) |
| **CNI alternativo** | Azure CNI + Cilium (powered by Cilium) | Calico, Cilium | Calico (para network policy), Cilium (via Dataplane V2) |
| **Rede de Pods** | Overlay CIDR (separado da VNet) | IPs flat VPC (baseado em ENI) | Alias IP ranges (CIDRs secundários) |
| **Engine de network policy** | Cilium (padrão em novos clusters) / Calico | Calico / VPC CNI Network Policy | Dataplane V2 (baseado em Cilium, nativo) |
| **Load Balancer L4** | Azure Load Balancer (Standard) | Network Load Balancer (NLB) | Google Cloud TCP/UDP Load Balancer |
| **Ingress L7 (gerenciado)** | Application Gateway for Containers | AWS ALB (via Load Balancer Controller) | GKE Gateway Controller / Ingress for HTTP(S) |
| **Suporte a Gateway API** | Suportado (via App Gw for Containers) | Suportado (via AWS Gateway API controller) | Suporte nativo (GKE Gateway Controller) |
| **Service mesh** | Istio-based service mesh (addon) | App Mesh / Istio on EKS | Managed Istio (Cloud Service Mesh) |
| **Load balancer interno** | Annotation: `service.beta.kubernetes.io/azure-load-balancer-internal: "true"` | Annotation: `service.beta.kubernetes.io/aws-load-balancer-internal: "true"` | Annotation: `networking.gke.io/load-balancer-type: "Internal"` |
| **DNS** | CoreDNS (padrão) | CoreDNS (padrão) | kube-dns / CoreDNS |

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
| **Menor custo** | AKS | Control plane gratuito; preços competitivos de VMs; boas ferramentas gratuitas |
| **Ecossistema AWS existente** | EKS | Integração nativa com IAM, VPC, ALB, CloudWatch e mais de 200 serviços AWS |
| **Ecossistema Azure existente** | AKS | Integração nativa com Entra ID, Azure DevOps, App Gateway, Azure Monitor |
| **Ecossistema GCP existente** | GKE | Integração nativa com Cloud IAM, Cloud Run, Cloud Build, BigQuery |
| **Experiência Kubernetes-nativa** | GKE | Google criou o Kubernetes; GKE frequentemente tem os recursos mais novos primeiro |
| **Operações mais simples (K8s serverless)** | GKE Autopilot | Sem gerenciamento de nós; pagamento por pod; segurança automática; boas práticas embutidas |
| **Híbrido/multi-cloud** | AKS (Azure Arc) ou EKS Anywhere | Arc suporta K8s em qualquer lugar; EKS Anywhere roda on-premises |
| **Containers Windows** | AKS | Suporte mais maduro a node pools Windows; clusters mistos Linux/Windows |
| **Cargas de trabalho AI/ML (GPU)** | Todos os três | Todos suportam node pools GPU; GKE tem TPU; AKS tem AMD MI300X; EKS tem Inferentia/Trainium |
| **Indústrias com compliance rigoroso** | Todos os três | Todos possuem FedRAMP, HIPAA, SOC2; avalie certificações específicas necessárias |
| **Expertise da equipe** | Corresponda às habilidades cloud existentes | O provedor que sua equipe já conhece trará resultados mais rápidos |

### Considerações de Custo Além do Control Plane

O custo do control plane é uma pequena fração da conta total. Planeje o orçamento para:
- **Computação** (VMs/instâncias dos nós) — tipicamente 60–80% da conta.
- **Armazenamento** (discos persistentes, file shares) — varia por carga de trabalho.
- **Rede** (load balancers, NAT gateways, tráfego cross-AZ, egress).
- **Monitoramento** (ingestão de logs, armazenamento de métricas, alertas).

> **Dica:** Todos os três provedores oferecem descontos por compromisso de uso (Reserved Instances, Savings Plans, Committed Use Discounts) que podem reduzir custos de computação em 30–60%.

---

*Navegação: [← Apêndice B: Referência Rápida do kubectl](appendix-b-cheatsheet.md)*
