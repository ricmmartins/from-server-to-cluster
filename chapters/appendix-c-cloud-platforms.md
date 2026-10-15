# Appendix C: Cloud Platform Comparison

*A practical guide to choosing and using managed Kubernetes across major cloud providers.*

---

## Why Managed Kubernetes?

Running Kubernetes yourself means operating etcd, the API server, controller-manager, and scheduler — all of which must be highly available, patched, and backed up. Managed services eliminate that operational overhead:

- **Control plane managed for you** — the provider handles upgrades, patching, HA, and etcd backups.
- **Conditional SLA coverage** — depends on tier and topology; AKS Free has no financially backed uptime SLA. A control-plane SLA is not an application availability guarantee.
- **Deep integrations** — IAM, networking, storage, monitoring, and marketplace services work out of the box.
- **Faster time to production** — spin up a production-grade cluster in minutes, not days.

You still manage your workloads, node pools, networking policies, and application-level concerns — but the undifferentiated heavy lifting of the control plane is off your plate.

---

## Quick Comparison Table

**Documentation review: 2026-10-15.** These are configuration choices, not certified defaults or capacity limits. Defaults, quotas, add-on maturity, machine availability, and prices vary by region, account, cluster mode, and version; confirm them before provisioning. No authenticated regional discovery or cloud creation was performed for this review.

| Feature | AKS (Azure) | EKS (AWS) | GKE (Google Cloud) |
|---------|-------------|-----------|---------------------|
| **CLI tool** | `az aks` | `eksctl` / `aws eks` | `gcloud container clusters` |
| **Cluster management cost (not workloads)** | Free tier: no management fee, no uptime SLA; Standard/Premium: paid tiers (see pricing below) | $0.10/cluster-hour standard support; $0.60/cluster-hour extended support | $0.10/cluster-hour base fee; conditional free-tier credit; extended-support surcharge |
| **Networking option** | Azure CNI Overlay | AWS VPC CNI | VPC-native (alias IPs) |
| **L4 load balancer option** | Azure Load Balancer | AWS Network Load Balancer (controller/mode dependent) | Google Cloud Load Balancer |
| **L7 routing implementation** | Application Gateway for Containers | AWS Load Balancer Controller (ALB) | GKE Gateway Controller |
| **Identity/Workload Identity** | Entra ID Workload Identity | IAM Roles for Service Accounts (IRSA) / Pod Identity | Workload Identity Federation |
| **Secrets integration** | Azure Key Vault CSI driver | AWS Secrets Manager CSI driver | Secret Manager CSI driver |
| **Monitoring** | Azure Monitor Container Insights | Amazon CloudWatch Container Insights | Google Cloud Monitoring + GKE dashboards |
| **Node OS options** | Azure Linux, Ubuntu | Amazon Linux 2023, Bottlerocket, Ubuntu | Container-Optimized OS, Ubuntu |
| **Autopilot/serverless mode** | AKS Automatic | EKS Auto Mode | GKE Autopilot |
| **Pod density** | Verify network mode and `maxPods` | Verify instance ENI/IP limits, prefix delegation, and kubelet configuration | Verify cluster mode and node-pool settings |
| **Version support policy** | Community-support calendar; optional explicitly enabled Premium LTS | 14 months standard + 12 months paid extended support | Release-channel lifecycle; Extended channel has separate support/pricing terms |
| **Available Kubernetes versions** | Run `az aks get-versions` for the region | Run `aws eks describe-cluster-versions` for the region | Run `gcloud container get-server-config` for the region/channel |
| **Node scaling options** | Cluster Autoscaler / node auto-provisioning (check current support) | Karpenter / Cluster Autoscaler / Auto Mode | GKE cluster autoscaler / node auto-provisioning |

---

## Cluster Creation

The book's **local teaching baseline is Kubernetes 1.35.0 on kind 0.31.0 with kubectl 1.35.x**. Managed services do not necessarily offer upstream patch `1.35.0`; choose a currently supported regional minor/patch and check client/server version skew (kubectl supports one minor either side of the API server). Prefer an offered 1.35 release when suitable, not an obsolete 1.32 pin or an assumed `.0`.

The commands below are **unexecuted Bash templates**, not deployment evidence. They assume installed, authenticated CLIs, permissions, quota, network/IAM prerequisites, and approved billing. First run each read-only discovery command; select a supported version and verify machine types, zones, quota, controller/CSI compatibility, and support tier before running any billable create command. Documentation availability does not prove that a subscription can create that configuration.

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

**Key options:**
- `--tier standard` — paid cluster management with uptime SLA; workload resources are charged separately. Use `--tier free` only when its no-SLA terms suit the workload.
- **Do not reuse Kubernetes 1.32 as a standard-support example.** Its AKS community support ended March 2026. The published LTS calendar runs through March 2027, but LTS requires **both** `--tier premium` and `--k8s-support-plan AKSLongTermSupport`; it is not automatic. Supported patches and regional creation eligibility still require discovery. Creation of an AKS 1.32 LTS cluster in any particular region was **not demonstrated**.
- `--enable-addons monitoring` — enable Container Insights at creation.
- `--os-sku AzureLinux3` — choose a supported Azure Linux image after checking version/region compatibility; do not select retired Azure Linux 2.0.
- `--enable-oidc-issuer --enable-workload-identity` — enable Workload Identity.

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

**Alternative: AWS CLI only**
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

The AWS CLI alternative assumes existing cluster/node IAM roles, correctly configured subnets, security groups, and sufficient permissions; it is not a complete VPC/IAM bootstrap.

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

GKE names use lowercase letters, digits, and hyphens, starting with a letter and ending with a letter or digit. Replace `PROJECT_ID` with your project ID and prepare the node service account/IAM permissions first. In a regional multi-zone pool, `--num-nodes` is **per zone**: the example starts with **1 × 3 = 3 nodes**, whereas `--num-nodes 3` across those zones would start **9 nodes**, not 3. Verify zone capacity and quota; later scaling/upgrades can change the count.

**GKE Autopilot (evaluate workload constraints and billing model first):**
```bash
# Run regional discovery above first; the channel selects an offered version.
gcloud container clusters create-auto my-autopilot-cluster \
  --region us-central1 \
  --release-channel regular

gcloud container clusters get-credentials my-autopilot-cluster --region us-central1
```

---

## Storage

| Feature | AKS (Azure) | EKS (AWS) | GKE (Google Cloud) |
|---------|-------------|-----------|---------------------|
| **CSI driver (block)** | Azure Disk CSI (`disk.csi.azure.com`) | EBS CSI (`ebs.csi.aws.com`); Auto Mode uses `ebs.csi.eks.amazonaws.com` | Persistent Disk CSI (`pd.csi.storage.gke.io`) |
| **CSI driver (file/NFS)** | Azure Files CSI (`file.csi.azure.com`) | EFS CSI (`efs.csi.aws.com`) | Filestore CSI (`filestore.csi.storage.gke.io`) |
| **StorageClass examples (inspect defaults)** | `managed-csi` (inspect actual redundancy/SKU) | No universal `gp3` default; define/inspect the class for the selected provisioner | `standard-rwo` (PD balanced; inspect cluster default) |
| **Higher-performance storage option** | `managed-csi-premium` (Premium SSD) | A separately configured StorageClass with EBS type `io2` | `premium-rwo` (PD SSD) |
| **ReadWriteMany support** | Azure Files (SMB/NFS) | EFS (NFS) | Filestore (NFS) |
| **Volume snapshots** | Supported (Azure Disk CSI) | Supported (EBS CSI) | Supported (PD CSI) |
| **Volume expansion** | Online resize supported | Online resize supported | Online resize supported |
| **Volume attachment limits** | Verify VM SKU/CSI limits | Verify instance type/EBS/CSI limits | Verify machine type/CSI limits |

Inspect `kubectl get storageclass` and `kubectl get csidriver` before choosing `storageClassName`; class names are configuration, not an API guarantee. Standard EKS needs the EBS CSI add-on/driver and IAM permissions (EKS Pod Identity or IRSA); missing permissions cause provisioning failures. Auto Mode manages its own storage controller, but does **not** create a StorageClass for you: define one with provisioner `ebs.csi.eks.amazonaws.com`, the desired disk type (for example `gp3`), and suitable topology/binding settings, and verify the cluster IAM role's block-storage permissions. Standard CSI and Auto Mode volumes are managed separately; migration requires a supported snapshot workflow. Snapshots require the appropriate snapshot controller. See the official storage references below.

---

## Networking

| Feature | AKS (Azure) | EKS (AWS) | GKE (Google Cloud) |
|---------|-------------|-----------|---------------------|
| **Networking option** | Azure CNI Overlay | AWS VPC CNI | VPC-native (alias IPs) |
| **Alternative CNI** | Azure CNI + Cilium (powered by Cilium) | Calico, Cilium | Calico (for network policy), Cilium (via Dataplane V2) |
| **Pod networking** | Overlay CIDR (separate from VNet) | Flat VPC IPs (ENI-based) | Alias IP ranges (secondary CIDRs) |
| **Network policy engine (verify enabled)** | Azure CNI powered by Cilium / other supported policy configurations | VPC CNI Network Policy / compatible alternatives | Dataplane V2 or a supported policy-enabled configuration |
| **L4 Load Balancer** | Azure Load Balancer (Standard) | Network Load Balancer (NLB) | Google Cloud TCP/UDP Load Balancer |
| **L7 Ingress (managed)** | Application Gateway for Containers | AWS ALB (via Load Balancer Controller) | GKE Gateway Controller / Ingress for HTTP(S) |
| **Gateway API implementation** | Application Gateway for Containers | Select an AWS-supported controller and verify its API/feature support | GKE Gateway Controller |
| **Service mesh option** | Istio-based service mesh add-on | Istio on EKS (self-managed; verify compatibility/support) | Cloud Service Mesh |
| **Internal load balancer** | Annotation: `service.beta.kubernetes.io/azure-load-balancer-internal: "true"` | Annotation: `service.beta.kubernetes.io/aws-load-balancer-internal: "true"` | Annotation: `networking.gke.io/load-balancer-type: "Internal"` |
| **DNS** | CoreDNS (default) | CoreDNS (default) | kube-dns / CoreDNS |

NetworkPolicy objects need an enabled enforcing implementation; namespaced resources alone do not provide network isolation. L4/L7 annotations and Gateway API support are controller/version specific, so validate them against the chosen implementation. **AWS App Mesh is not a current option:** AWS ended support and resource/console access on **2026-09-30**.

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
| **No cluster management fee for a lab** | Evaluate AKS Free | No uptime SLA; nodes, storage, networking, and monitoring still cost money. This does not establish the lowest total cost |
| **Existing AWS ecosystem** | EKS | Native integration with IAM, VPC, ALB, CloudWatch, and 200+ AWS services |
| **Existing Azure ecosystem** | AKS | Native integration with Entra ID, Azure DevOps, App Gateway, Azure Monitor |
| **Existing GCP ecosystem** | GKE | Native integration with Cloud IAM, Cloud Run, Cloud Build, BigQuery |
| **Kubernetes-native experience** | GKE | Google authored Kubernetes; GKE often has the newest features first |
| **Reduced node-management work** | Evaluate GKE Autopilot | Managed node lifecycle; Pod- or node-based compute billing depends on workload/compute class |
| **Hybrid/multi-cloud** | AKS (Azure Arc) or EKS Anywhere | Arc supports K8s anywhere; EKS Anywhere runs on-premises |
| **Windows containers** | AKS | Most mature Windows node pool support; mixed Linux/Windows clusters |
| **AI/ML workloads (GPU)** | All three | All support GPU node pools; GKE has TPU; AKS has AMD MI300X; EKS has Inferentia/Trainium |
| **Compliance-heavy industries** | All three | All have FedRAMP, HIPAA, SOC2; evaluate specific certifications needed |
| **Team expertise** | Match existing cloud skills | The provider your team already knows will yield faster results |

### Cost Considerations Beyond the Control Plane

Cluster management and workload resources are separate charges; their relative share depends on the workload. Budget for:
- **Compute** (node VMs/instances or Autopilot billable resources).
- **Storage** (persistent disks, file shares) — varies by workload.
- **Networking** (load balancers, NAT gateways, cross-AZ traffic, egress).
- **Monitoring** (log ingestion, metrics storage, alerting).

Committed-use discounts and Spot capacity have eligibility, commitment, and interruption conditions. Compare provider estimates for the actual workload rather than assuming a fixed discount.

### Pricing and Free-Tier Conditions — Checked 2026-10-15

USD public list-price examples below are **management fees only**, not account quotes. Use the linked official pricing pages/calculators for your region, currency, contract, taxes, and additional features. A **730-hour illustrative month** is not every calendar month.

| Service | Management/support charges | Workload charges and free-tier conditions |
|---------|----------------------------|------------------------------------------|
| AKS Free | No cluster management fee; **no financially backed uptime SLA** | Underlying VMs, disks, networking, and other services remain billable; "Free" does not mean a free cluster workload |
| AKS Standard / Premium | Paid management tiers; Standard/Premium include API-server uptime SLA (99.95% with availability zones, 99.9% without), subject to terms. Premium plus explicit LTS enrollment provides extended version support | Workload resources charged separately. The retrieved public pricing content did not expose a reliable numeric tier quote; check the official regional calculator instead of assuming a rate |
| EKS standard support | **$0.10/cluster-hour → $73 for 730 hours** | EC2/Fargate, EBS, load balancers, networking, and observability are additional; Auto Mode has an additional management charge for managed EC2 capacity. Do not assume promotional AWS credits waive the cluster fee |
| EKS extended support | **$0.60/cluster-hour total → $438 for 730 hours**, not $0.10 + $0.60 | Support status determines the fee: as of this review, EKS **1.35** is in standard support (ends 2027-03-27); **1.32** entered extended support 2026-03-23 (ends 2027-03-23). Recheck the lifecycle and region before selection |
| GKE base management | **$0.10/cluster-hour → $73 for 730 hours**, before eligible credits | **$74.40/month per billing account** credit applies only to Autopilot and zonal Standard cluster management fees; not regional Standard fees, compute, or other SKUs. Unused credits do not roll over; they are not a per-cluster allowance |
| GKE extended support | Extended-period surcharge **$0.50/cluster-hour**, plus the $0.10 base = **$0.60/hour → $438 for 730 hours** before applicable credits | Applies when the minor version is in its extended-support period; verify Extended-channel enrollment, lifecycle, and exact credit treatment |
| GKE Autopilot compute | Separate from the cluster management fee | General-purpose eligible compute classes use **Pod-based billing** for requested CPU, memory, and ephemeral storage (after defaults/minimum adjustments). Specific-hardware classes/workloads use **node-based billing** for the whole VM plus an Autopilot management premium; do not describe all Autopilot as pay-per-Pod |

For example, a single eligible GKE zonal Standard or Autopilot cluster's $73 base management fee can be covered by an otherwise unused $74.40 billing-account credit; the regional Standard cluster above still incurs $73, and compute is not covered in either case. EKS examples assume no additional paid control-plane capacity/features.

### Official Sources and Remaining Verification

All links below were checked **2026-10-15**; public documentation confirms the stated policies, not successful provisioning in an account:

- **Azure:** [AKS pricing](https://azure.microsoft.com/en-us/pricing/details/kubernetes-service/), [Free/Standard/Premium and SLA terms](https://learn.microsoft.com/en-us/azure/aks/free-standard-pricing-tiers), [version/LTS calendar](https://learn.microsoft.com/en-us/azure/aks/supported-kubernetes-versions), [explicit Premium LTS enrollment](https://learn.microsoft.com/en-us/azure/aks/long-term-support), and [Azure CLI discovery](https://learn.microsoft.com/en-us/cli/azure/aks#az-aks-get-versions).
- **AWS:** [EKS pricing](https://aws.amazon.com/eks/pricing/), [version lifecycle and discovery](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html), [EBS CSI provisioners and IAM](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html), [Auto Mode StorageClasses](https://docs.aws.amazon.com/eks/latest/userguide/create-storage-class.html), and [App Mesh end-of-support notice](https://docs.aws.amazon.com/app-mesh/latest/userguide/what-is-app-mesh.html).
- **Google Cloud:** [GKE pricing, free tier, Autopilot billing, and extended support](https://cloud.google.com/kubernetes-engine/pricing), [regional cluster creation](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-regional-cluster), [create command and node-count semantics](https://cloud.google.com/sdk/gcloud/reference/container/clusters/create), and [regional version discovery](https://cloud.google.com/sdk/gcloud/reference/container/get-server-config).

**Still requires account/regional verification:** exact offered patches and new-cluster eligibility (especially old AKS LTS versions), SKU/zone capacity and quotas, IAM permissions, current StorageClass defaults, policy enforcement, add-on/controller support, and final workload/tier prices. The capability tables are orientation, not certification of limits, compliance eligibility, or a working deployment. No cloud mutation or tooling installation was executed.

---

*Navigation: [← Appendix B: kubectl Cheatsheet](appendix-b-cheatsheet.md)*
