# Appendix C: Cloud Platform Comparison

*A practical guide to choosing and using managed Kubernetes across major cloud providers.*

---

## Why Managed Kubernetes?

Running Kubernetes yourself means operating etcd, the API server, controller-manager, and scheduler — all of which must be highly available, patched, and backed up. Managed services eliminate that operational overhead:

- **Control plane managed for you** — the provider handles upgrades, patching, HA, and etcd backups.
- **SLA-backed availability** — financially backed uptime guarantees (typically 99.5%–99.95%).
- **Deep integrations** — IAM, networking, storage, monitoring, and marketplace services work out of the box.
- **Faster time to production** — spin up a production-grade cluster in minutes, not days.

You still manage your workloads, node pools, networking policies, and application-level concerns — but the undifferentiated heavy lifting of the control plane is off your plate.

---

## Quick Comparison Table

| Feature | AKS (Azure) | EKS (AWS) | GKE (Google Cloud) |
|---------|-------------|-----------|---------------------|
| **CLI tool** | `az aks` | `eksctl` / `aws eks` | `gcloud container clusters` |
| **Control plane cost** | Free | ~$73/month ($0.10/hr) | Standard: $0.10/hr; Autopilot: $0.10/hr (compute billed per pod) |
| **Default CNI** | Azure CNI Overlay | AWS VPC CNI | VPC-native (alias IPs) |
| **Default load balancer** | Azure Load Balancer | AWS Network Load Balancer | Google Cloud Load Balancer |
| **Managed Ingress** | Application Gateway for Containers | AWS ALB Ingress Controller | GKE Gateway Controller |
| **Identity/Workload Identity** | Entra ID Workload Identity | IAM Roles for Service Accounts (IRSA) / Pod Identity | Workload Identity Federation |
| **Secrets integration** | Azure Key Vault CSI driver | AWS Secrets Manager CSI driver | Secret Manager CSI driver |
| **Monitoring** | Azure Monitor Container Insights | Amazon CloudWatch Container Insights | Google Cloud Monitoring + GKE dashboards |
| **Node OS options** | Azure Linux, Ubuntu | Amazon Linux 2023, Bottlerocket, Ubuntu | Container-Optimized OS, Ubuntu |
| **Autopilot/serverless mode** | AKS Automatic | EKS Auto Mode | GKE Autopilot |
| **Max pods per node (default)** | 250 (Azure CNI Overlay) | Varies by instance type (17–110; 110 with prefix delegation) | 110 |
| **Version support policy** | N-2 (current + 2 prior minors) | 14 months standard + 12 months extended | 3 most recent minors |
| **Supported K8s versions (mid-2025)** | 1.30 – 1.32 | 1.30 – 1.32 (1.29 on extended) | 1.30 – 1.32 |
| **Cluster autoscaler** | Cluster Autoscaler / Karpenter (preview) | Karpenter (recommended) / Cluster Autoscaler | GKE Autoscaler (built-in) / Karpenter (preview) |

---

## Cluster Creation

### Azure (AKS)

```bash
# Install/update Azure CLI
az upgrade

# Create a resource group
az group create --name myResourceGroup --location eastus

# Create an AKS cluster
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --node-count 3 \
  --node-vm-size Standard_DS3_v2 \
  --kubernetes-version 1.32 \
  --network-plugin azure \
  --enable-managed-identity \
  --generate-ssh-keys

# Get credentials for kubectl
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster

# Verify connectivity
kubectl get nodes
```

**Key options:**
- `--enable-addons monitoring` — enable Container Insights at creation.
- `--os-sku AzureLinux` — use Azure Linux (recommended for new clusters).
- `--enable-oidc-issuer --enable-workload-identity` — enable Workload Identity.

### AWS (EKS)

```bash
# Install eksctl (https://eksctl.io)
# Create a cluster with managed node group
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

# eksctl automatically updates kubeconfig
kubectl get nodes
```

**Alternative: AWS CLI only**
```bash
aws eks create-cluster \
  --name myEKSCluster \
  --kubernetes-version 1.32 \
  --role-arn arn:aws:iam::ACCOUNT:role/eks-cluster-role \
  --resources-vpc-config subnetIds=subnet-xxx,subnet-yyy,securityGroupIds=sg-zzz

# Then create a managed node group separately
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
# Standard cluster
gcloud container clusters create myGKECluster \
  --region us-central1 \
  --num-nodes 3 \
  --machine-type e2-standard-4 \
  --cluster-version 1.32 \
  --enable-ip-alias \
  --workload-pool=PROJECT_ID.svc.id.goog

# Get credentials
gcloud container clusters get-credentials myGKECluster --region us-central1

kubectl get nodes
```

**GKE Autopilot (recommended for most workloads):**
```bash
gcloud container clusters create-auto myAutopilotCluster \
  --region us-central1 \
  --release-channel regular

gcloud container clusters get-credentials myAutopilotCluster --region us-central1
```

---

## Storage

| Feature | AKS (Azure) | EKS (AWS) | GKE (Google Cloud) |
|---------|-------------|-----------|---------------------|
| **CSI driver (block)** | Azure Disk CSI (`disk.csi.azure.com`) | EBS CSI (`ebs.csi.aws.com`) | Persistent Disk CSI (`pd.csi.storage.gke.io`) |
| **CSI driver (file/NFS)** | Azure Files CSI (`file.csi.azure.com`) | EFS CSI (`efs.csi.aws.com`) | Filestore CSI (`filestore.csi.storage.gke.io`) |
| **Default StorageClass** | `managed-csi` (Azure Disk LRS) | `gp3` (EBS gp3) | `standard-rwo` (PD balanced) |
| **Premium storage class** | `managed-csi-premium` (Premium SSD) | `io2` (Provisioned IOPS) | `premium-rwo` (PD SSD) |
| **ReadWriteMany support** | Azure Files (SMB/NFS) | EFS (NFS) | Filestore (NFS) |
| **Volume snapshots** | Supported (Azure Disk CSI) | Supported (EBS CSI) | Supported (PD CSI) |
| **Volume expansion** | Online resize supported | Online resize supported | Online resize supported |
| **Max volumes per node** | ~64 (depends on VM size) | Varies by instance type (EBS limits) | 128 PDs per node |

---

## Networking

| Feature | AKS (Azure) | EKS (AWS) | GKE (Google Cloud) |
|---------|-------------|-----------|---------------------|
| **Default CNI** | Azure CNI Overlay | AWS VPC CNI | VPC-native (alias IPs) |
| **Alternative CNI** | Azure CNI + Cilium (powered by Cilium) | Calico, Cilium | Calico (for network policy), Cilium (via Dataplane V2) |
| **Pod networking** | Overlay CIDR (separate from VNet) | Flat VPC IPs (ENI-based) | Alias IP ranges (secondary CIDRs) |
| **Network policy engine** | Cilium (default in new clusters) / Calico | Calico / VPC CNI Network Policy | Dataplane V2 (Cilium-based, built-in) |
| **L4 Load Balancer** | Azure Load Balancer (Standard) | Network Load Balancer (NLB) | Google Cloud TCP/UDP Load Balancer |
| **L7 Ingress (managed)** | Application Gateway for Containers | AWS ALB (via Load Balancer Controller) | GKE Gateway Controller / Ingress for HTTP(S) |
| **Gateway API support** | Supported (via App Gw for Containers) | Supported (via AWS Gateway API controller) | Native support (GKE Gateway Controller) |
| **Service mesh** | Istio-based service mesh (addon) | App Mesh / Istio on EKS | Managed Istio (Cloud Service Mesh) |
| **Internal load balancer** | Annotation: `service.beta.kubernetes.io/azure-load-balancer-internal: "true"` | Annotation: `service.beta.kubernetes.io/aws-load-balancer-internal: "true"` | Annotation: `networking.gke.io/load-balancer-type: "Internal"` |
| **DNS** | CoreDNS (default) | CoreDNS (default) | kube-dns / CoreDNS |

---

## Security

| Feature | AKS (Azure) | EKS (AWS) | GKE (Google Cloud) |
|---------|-------------|-----------|---------------------|
| **Identity provider** | Microsoft Entra ID | AWS IAM | Google Cloud IAM |
| **Workload-to-cloud auth** | Workload Identity (Entra ID federated tokens) | IRSA / EKS Pod Identity | Workload Identity Federation |
| **RBAC integration** | Entra ID groups → K8s RBAC | IAM roles mapped to K8s RBAC via `aws-auth` ConfigMap or access entries | Google groups → K8s RBAC |
| **Secrets management** | Azure Key Vault (Secrets Store CSI) | AWS Secrets Manager / SSM Parameter Store (CSI) | Secret Manager (CSI) |
| **Image signing/verification** | Notation + Azure Policy | ECR image scanning + Sigstore | Binary Authorization |
| **Pod Security** | Pod Security Admission (built-in) + Azure Policy | Pod Security Admission (built-in) | Pod Security Admission + GKE Policy Controller |
| **Node security** | Azure Linux / Mariner (hardened) | Bottlerocket (hardened, immutable) | Container-Optimized OS (hardened, minimal) |
| **Encryption at rest** | Etcd encrypted by default; BYOK via Azure Key Vault | Etcd encrypted by default; BYOK via KMS envelope encryption | Etcd encrypted by default; BYOK via Cloud KMS |
| **Private cluster** | Private cluster (API server on private endpoint) | Private endpoint for API server | Private cluster (VPC-internal control plane) |

---

## Monitoring

| Feature | AKS (Azure) | EKS (AWS) | GKE (Google Cloud) |
|---------|-------------|-----------|---------------------|
| **Native monitoring** | Azure Monitor Container Insights | Amazon CloudWatch Container Insights | Google Cloud Monitoring (GKE dashboards) |
| **Metrics collection** | Managed Prometheus (Azure Monitor) | Amazon Managed Service for Prometheus | Google Cloud Managed Service for Prometheus |
| **Log aggregation** | Azure Monitor Logs (Log Analytics) | CloudWatch Logs + FluentBit | Cloud Logging |
| **Dashboards** | Azure Monitor workbooks + Grafana (managed) | CloudWatch dashboards + Amazon Managed Grafana | Cloud Monitoring dashboards + Google Cloud Managed Grafana |
| **Alerting** | Azure Monitor alerts | CloudWatch Alarms + SNS | Cloud Monitoring alerting policies |
| **Tracing** | Azure Monitor Application Insights | AWS X-Ray | Cloud Trace |
| **Cost analysis** | AKS cost analysis (built-in) | AWS Cost Explorer + Kubecost | GKE cost allocation (built-in) |
| **Prometheus-compatible** | Yes (Azure Managed Prometheus) | Yes (Amazon Managed Prometheus - AMP) | Yes (Google Managed Prometheus - GMP) |

---

## When to Choose Which

### Decision Matrix

| Priority | Best Choice | Why |
|----------|------------|-----|
| **Lowest cost** | AKS | Free control plane; competitive VM pricing; good free-tier tooling |
| **Existing AWS ecosystem** | EKS | Native integration with IAM, VPC, ALB, CloudWatch, and 200+ AWS services |
| **Existing Azure ecosystem** | AKS | Native integration with Entra ID, Azure DevOps, App Gateway, Azure Monitor |
| **Existing GCP ecosystem** | GKE | Native integration with Cloud IAM, Cloud Run, Cloud Build, BigQuery |
| **Kubernetes-native experience** | GKE | Google authored Kubernetes; GKE often has the newest features first |
| **Simplest operations (serverless K8s)** | GKE Autopilot | No node management; pay-per-pod; auto-security; built-in best practices |
| **Hybrid/multi-cloud** | AKS (Azure Arc) or EKS Anywhere | Arc supports K8s anywhere; EKS Anywhere runs on-premises |
| **Windows containers** | AKS | Most mature Windows node pool support; mixed Linux/Windows clusters |
| **AI/ML workloads (GPU)** | All three | All support GPU node pools; GKE has TPU; AKS has AMD MI300X; EKS has Inferentia/Trainium |
| **Compliance-heavy industries** | All three | All have FedRAMP, HIPAA, SOC2; evaluate specific certifications needed |
| **Team expertise** | Match existing cloud skills | The provider your team already knows will yield faster results |

### Cost Considerations Beyond the Control Plane

The control plane cost is a small fraction of the total bill. Budget for:
- **Compute** (node VMs/instances) — typically 60–80% of the bill.
- **Storage** (persistent disks, file shares) — varies by workload.
- **Networking** (load balancers, NAT gateways, cross-AZ traffic, egress).
- **Monitoring** (log ingestion, metrics storage, alerting).

> **Tip:** All three providers offer committed-use discounts (Reserved Instances, Savings Plans, Committed Use Discounts) that can reduce compute costs by 30–60%.

---

*Navigation: [← Appendix B: kubectl Cheatsheet](appendix-b-cheatsheet.md)*
