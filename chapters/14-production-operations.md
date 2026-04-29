# Chapter 14: Production Operations

*"Deploying to production is not the finish line — it's the starting line."*

---

## From Single Server Maintenance to Cluster Lifecycle

On a Linux server, Day-2 operations are familiar: you run `apt upgrade`, schedule `cron` backups of `/etc` and databases, put machines in maintenance mode for hardware swaps, and restore from backup when things go sideways. You've done it a hundred times.

In Kubernetes, the same responsibilities exist — but they're distributed across multiple nodes, multiple control plane components, and a shared state store (etcd) that holds the truth of your entire cluster. Upgrading isn't "run `apt upgrade` and reboot." It's a choreographed dance: upgrade the control plane, then drain each worker one by one, upgrade it, uncordon it, and move to the next. Miss a step and you have version skew. Rush it and you have downtime.

This chapter covers the essential Day-2 operations that keep a Kubernetes cluster healthy: upgrades, backups, node management, disaster recovery, and the Operator pattern that automates complex operational tasks.

---

## Cluster Upgrades

### The Upgrade Lifecycle

Kubernetes follows a strict upgrade sequence:

```
1. Upgrade control plane (API server, controller-manager, scheduler, etcd)
2. Upgrade workers one at a time (drain → upgrade kubelet/kubectl → uncordon)
3. Verify cluster health after each step
```

You cannot skip versions. Kubernetes supports upgrades one minor version at a time (e.g., 1.31 → 1.32, not 1.30 → 1.32).

### The kubeadm Upgrade Workflow

For self-managed clusters using kubeadm:

**Step 1 — Check available upgrades:**

```bash
sudo kubeadm upgrade plan
```

This command inspects your current cluster, checks the installed kubeadm version, and reports what versions you can upgrade to. It also warns about deprecated API versions and required actions.

**Step 2 — Upgrade the first control plane node:**

```bash
# Upgrade kubeadm itself first
sudo apt-get update
sudo apt-get install -y kubeadm=1.32.0-1.1

# Run the upgrade
sudo kubeadm upgrade apply v1.32.0

# Upgrade kubelet and kubectl on this node
sudo apt-get install -y kubelet=1.32.0-1.1 kubectl=1.32.0-1.1
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

**Step 3 — Upgrade additional control plane nodes (if HA):**

```bash
sudo kubeadm upgrade node
```

**Step 4 — Upgrade worker nodes (one at a time):**

```bash
# From a machine with kubectl access:
kubectl drain <worker-node> --ignore-daemonsets --delete-emptydir-data

# On the worker node:
sudo apt-get update
sudo apt-get install -y kubeadm=1.32.0-1.1
sudo kubeadm upgrade node
sudo apt-get install -y kubelet=1.32.0-1.1 kubectl=1.32.0-1.1
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# From the kubectl machine:
kubectl uncordon <worker-node>
```

Repeat for each worker.

### Version Skew Policy

Kubernetes components have strict compatibility rules:

| Component | Allowed Skew from API Server |
|-----------|------------------------------|
| kubelet | Up to 3 minor versions behind |
| kube-proxy | Up to 3 minor versions behind |
| controller-manager, scheduler | Up to 1 minor version behind |
| kubectl | ±1 minor version |

This means with an API server at v1.32, your kubelets can be v1.32, v1.31, v1.30, or v1.29 — but not v1.28 or v1.33.

### Managed Kubernetes Upgrades

In managed services (AKS, EKS, GKE), the provider handles control plane upgrades. You manage node pool upgrades:

| Provider | Control Plane | Node Pool |
|----------|--------------|-----------|
| AKS | `az aks upgrade --kubernetes-version` | `az aks nodepool upgrade` |
| EKS | `aws eks update-cluster-version` | Update node group AMI |
| GKE | Automatic or `gcloud container clusters upgrade` | `gcloud container clusters upgrade --node-pool` |

---

## etcd Backup and Restore

### Why etcd Is Critical

etcd is a distributed key-value store that holds **all** cluster state: Deployments, Services, ConfigMaps, Secrets, RBAC rules, custom resources — everything. If etcd is lost without a backup, your cluster is gone. The workloads keep running (kubelet continues managing containers), but you lose all management capability.

Think of etcd as the `/etc` directory of your entire cluster, plus the process table, plus the routing table, all in one database.

### Backup with etcdctl

```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

The certificate flags are required because etcd uses mutual TLS for all communication.

### Verify the Backup

```bash
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table
```

This outputs a table showing hash, revision, total keys, and total size — confirming the backup is valid and complete.

### Restore from Backup

```bash
# Stop the API server and etcd (on kubeadm clusters, move static pod manifests)
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/

# Restore the snapshot to a new data directory
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-from-backup

# Update etcd config to use the new data directory
# (edit the etcd.yaml manifest to point to /var/lib/etcd-from-backup)

# Restart components
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/
```

> **Note:** As of etcd 3.5+, the restore operation is migrating to `etcdutl snapshot restore`. Both tools work for current Kubernetes versions, but `etcdutl` is the recommended tool going forward.

### Scheduling Regular Backups

In production, automate etcd backups:

- **CronJob in the cluster:** Mount etcd certs and run etcdctl on a schedule
- **External cron:** A control plane node's crontab runs the backup and copies to remote storage
- **Velero:** Cluster-aware backup tool that handles both etcd state and persistent volumes

### Managed Kubernetes: etcd Is Invisible

In AKS, EKS, and GKE, the provider manages etcd completely. You cannot access it directly — and you don't need to. The provider handles replication, backup, and disaster recovery of the control plane data store.

---

## Node Operations

### Cordon — Mark Unschedulable

```bash
kubectl cordon <node-name>
```

Cordoning marks a node as unschedulable. Existing pods continue running, but the scheduler won't place new pods there. It's the "maintenance mode" of Kubernetes.

```bash
# Check node status — look for SchedulingDisabled
kubectl get nodes
# NAME                      STATUS                     ROLES
# worker-1                  Ready,SchedulingDisabled   <none>
```

### Drain — Evict Pods Gracefully

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

Draining does three things:
1. Cordons the node (marks unschedulable)
2. Evicts all pods (respecting PodDisruptionBudgets)
3. Waits for pods to be rescheduled elsewhere

The flags:
- `--ignore-daemonsets` — DaemonSet pods can't be evicted (they must run on every node), so ignore them
- `--delete-emptydir-data` — Allow eviction of pods with emptyDir volumes (their data is ephemeral anyway)

### Uncordon — Return to Service

```bash
kubectl uncordon <node-name>
```

The node becomes schedulable again. Existing pods on other nodes won't automatically migrate back — but new pods can now be placed here.

### The Node Replacement Pattern

```bash
# 1. Drain the old node
kubectl drain old-node --ignore-daemonsets --delete-emptydir-data

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

## Disaster Recovery Strategy

### What to Back Up

A complete disaster recovery plan covers:

| What | Why | How |
|------|-----|-----|
| **etcd** | All cluster state (Deployments, Services, RBAC) | `etcdctl snapshot save` |
| **Persistent Volumes** | Application data (databases, uploads) | CSI snapshots, Velero |
| **External state** | DNS records, TLS certificates, IAM configs | Infrastructure-as-Code (Terraform) |
| **YAML manifests** | Source of truth for all resources | Git repository (GitOps) |
| **Secrets** (encrypted) | Credentials, API keys | Sealed Secrets, external secret stores |

### RPO and RTO

| Term | Meaning | Example |
|------|---------|---------|
| **RPO** (Recovery Point Objective) | How much data can you lose? | "1 hour" = hourly backups acceptable |
| **RTO** (Recovery Time Objective) | How fast must you recover? | "15 minutes" = automated recovery needed |

Your backup frequency (RPO) and recovery automation (RTO) determine your disaster recovery investment.

### Multi-Cluster Strategies

| Strategy | Description | Complexity | Use Case |
|----------|-------------|-----------|----------|
| **Active/Passive** | Primary cluster serves traffic; standby receives backups | Medium | Cost-sensitive, moderate RTO |
| **Active/Active** | Multiple clusters serve traffic simultaneously | High | Zero-downtime requirements |
| **Backup/Restore** | Single cluster with off-site backups | Low | Dev/staging, long RTO acceptable |

### Velero for Cluster Backup

Velero is a CNCF project that backs up Kubernetes resources and persistent volumes:

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

Velero is the Kubernetes equivalent of enterprise backup tools like Veeam or Commvault — but purpose-built for the K8s resource model.

---

## CRDs and Operators

### Custom Resource Definitions (CRDs)

Kubernetes ships with built-in resource types: Pods, Deployments, Services, ConfigMaps. CRDs let you extend the API with your own resource types.

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

After applying this CRD, you can create `Database` resources like any built-in type:

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

And query them with kubectl:

```bash
kubectl get databases
kubectl describe database orders-db
```

### The Operator Pattern

A CRD alone is just data — it doesn't do anything. An **Operator** is a custom controller that watches your CRD and takes action. It's the reconciliation loop pattern (from Chapter 4) applied to domain-specific logic.

Think of it this way:
- **CRD** = "I want a PostgreSQL database with 3 replicas and 50 GiB storage"
- **Operator** = The automation that creates the StatefulSet, configures replication, manages backups, handles failover, and keeps the database in the desired state

Operators encode operational knowledge into software. Instead of a DBA running manual failover commands at 3 AM, the operator detects the primary is down and promotes a replica automatically.

### Popular Operators

| Operator | What It Manages | Linux Equivalent |
|----------|----------------|-----------------|
| cert-manager | TLS certificates (Let's Encrypt) | certbot + cron |
| prometheus-operator | Monitoring stack (Prometheus + Alertmanager) | prometheus + systemd |
| postgres-operator | PostgreSQL clusters (replication, backup, failover) | patroni + manual DBA work |
| strimzi | Apache Kafka clusters | Kafka scripts + manual ops |
| ArgoCD | GitOps continuous deployment | deploy scripts + webhooks |

### Finding Operators

- **OperatorHub.io** (https://operatorhub.io) — Catalog of community operators
- **Artifact Hub** (https://artifacthub.io) — Includes Helm charts and operators
- **GitHub** — Many operators are open source projects

---

## Cluster Lifecycle Tools

| Tool | Use Case | Type |
|------|----------|------|
| **kubeadm** | Bootstrap bare-metal/VM clusters | Self-managed |
| **Kind** | Local development and testing | Dev/CI |
| **k3s** | Lightweight clusters, edge, IoT | Self-managed (minimal) |
| **AKS** | Azure managed Kubernetes | Cloud-managed |
| **EKS** | AWS managed Kubernetes | Cloud-managed |
| **GKE** | Google Cloud managed Kubernetes | Cloud-managed |

### Managed vs. Self-Managed Comparison

| Aspect | Self-Managed (kubeadm) | Cloud-Managed (AKS/EKS/GKE) |
|--------|----------------------|------------------------------|
| Control plane | You manage | Provider manages |
| etcd | You backup/restore | Provider handles |
| Upgrades | Manual, step-by-step | Semi-automated |
| Node scaling | Manual or cluster-autoscaler | Integrated autoscaler |
| Cost | Infrastructure only | Infrastructure + management fee |
| Customization | Full control | Provider's guardrails |
| SLA | Your responsibility | Provider SLA (99.95%+) |

---

## Linux ↔ Kubernetes Operations Comparison

| Linux Operation | K8s Equivalent | Notes |
|----------------|---------------|-------|
| `apt upgrade` + reboot | `kubeadm upgrade` + drain/uncordon | Rolling node-by-node upgrade |
| `rsync /etc`, cron backups | `etcdctl snapshot save` | Cluster state backup |
| Restore from backup | `etcdctl snapshot restore` | Cluster recovery |
| Maintenance mode | `kubectl cordon` + `drain` | Take node offline safely |
| `puppet`/`chef` agents | Operators | Automated application management |
| `/etc/cron.d/backup` | Velero scheduled backups | Periodic cluster snapshots |
| Custom init scripts | CRDs + custom controllers | Extend system behavior |
| Cluster (Pacemaker/Corosync) | K8s control plane HA | Quorum-based availability |

---

> ### ⚠️ Where the Linux Analogy Breaks
>
> **In-place vs. rolling upgrades:** Linux upgrades can be done in-place — `apt upgrade` on a running machine. Kubernetes upgrades are rolling: you upgrade the control plane first, then drain and upgrade each worker node individually. You can't just run "yum update kubernetes" on a live node with running pods. The orchestration is part of the process.
>
> **No single "backup everything":** On Linux, you can `tar` the whole filesystem and call it a backup. In Kubernetes, cluster state (etcd) and application data (PersistentVolumes) live in different places with different backup mechanisms. You need both, plus external state like DNS records, TLS certificates, and IAM configurations. A complete backup strategy touches multiple systems.
>
> **Operators are more than scripts:** A Linux cron job runs periodically and applies a fix. An Operator runs continuously, watches resources in real-time, and reacts immediately. It's the difference between checking your email once an hour and having push notifications. Operators don't just automate tasks — they encode domain expertise and self-heal in real-time.

---

## Diagnostic Lab: Production Operations

### Prerequisites

```bash
# Create a multi-node Kind cluster
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

# Verify cluster
kubectl get nodes
```

---

### Lab 1: Drain and Uncordon

**Step 1 — Deploy a workload across workers:**

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

# Wait for pods to be ready
kubectl wait --for=condition=Ready pod -l app=web-fleet --timeout=60s
kubectl get pods -l app=web-fleet -o wide
```

Note which pods are on which nodes.

**Step 2 — Cordon a worker:**

```bash
# Get worker node names
WORKER=$(kubectl get nodes --selector='!node-role.kubernetes.io/control-plane' -o jsonpath='{.items[0].metadata.name}')
echo "Cordoning node: $WORKER"

kubectl cordon $WORKER
kubectl get nodes
```

**Step 3 — Verify new pods can't be scheduled on the cordoned node:**

```bash
# Scale up — new pods should only go to the other worker
kubectl scale deployment web-fleet --replicas=10
sleep 5
kubectl get pods -l app=web-fleet -o wide | grep -c "$WORKER"
# Existing pods remain, but no new pods land on the cordoned node
```

**Step 4 — Drain the cordoned worker:**

```bash
kubectl drain $WORKER --ignore-daemonsets --delete-emptydir-data
kubectl get pods -l app=web-fleet -o wide
# All web-fleet pods are now on the remaining worker(s)
```

**Step 5 — Uncordon and rebalance:**

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

**Clean up:**

```bash
kubectl delete deployment web-fleet
```

---

### Lab 2: etcd Backup

**Step 1 — Identify the etcd pod:**

```bash
kubectl get pods -n kube-system | grep etcd
# etcd-prod-ops-lab-control-plane
```

**Step 2 — Check etcd health:**

```bash
kubectl exec -n kube-system etcd-prod-ops-lab-control-plane -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```

**Step 3 — Create a snapshot:**

```bash
kubectl exec -n kube-system etcd-prod-ops-lab-control-plane -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/lib/etcd/backup.db
```

**Step 4 — Verify the backup:**

```bash
kubectl exec -n kube-system etcd-prod-ops-lab-control-plane -- etcdctl \
  snapshot status /var/lib/etcd/backup.db --write-out=table
```

You should see a table with hash, revision, total keys, and total size — confirming a valid backup.

**Step 5 — Check what's stored in etcd:**

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

**Step 1 — Create a CRD:**

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

**Step 2 — Verify the CRD is registered:**

```bash
kubectl get crd | grep webapp
kubectl api-resources | grep webapp
```

**Step 3 — Create custom resources:**

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

**Step 4 — Query your custom resources:**

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

**Step 5 — Clean up CRD resources:**

```bash
kubectl delete webapp --all
kubectl delete crd webapps.lab.example.com
```

---

### Lab 4: Simulate Cluster Upgrade Workflow

While we can't actually upgrade Kind (it's a fixed version), we can walk through the exact workflow:

**Step 1 — Check current versions:**

```bash
kubectl get nodes -o wide
kubectl version
```

**Step 2 — Review the upgrade plan (conceptual):**

```bash
# On a real kubeadm cluster, you would run:
# sudo kubeadm upgrade plan
# This shows available versions, required actions, and deprecation warnings

# In our lab, let's at least check what kubeadm would report
echo "=== Current Cluster Version ==="
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kubeletVersion}{"\n"}{end}'
```

**Step 3 — Practice the drain/uncordon cycle:**

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
kubectl drain $WORKER --ignore-daemonsets --delete-emptydir-data

# Verify pods moved
kubectl get pods -l app=upgrade-test -o wide
echo "All pods moved off $WORKER"

# Step B: Uncordon (simulating post-upgrade)
echo "Uncordoning $WORKER..."
kubectl uncordon $WORKER

# Step C: Verify node is back in rotation
kubectl get nodes
```

**Step 4 — Document the real upgrade steps:**

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

**Clean up:**

```bash
kubectl delete deployment upgrade-test
```

---

### Lab Cleanup

```bash
# Delete the Kind cluster
kind delete cluster --name prod-ops-lab
```

---

## Key Takeaways

1. **Upgrade control plane first, then workers.** Never upgrade workers before the control plane. Follow the version skew policy: kubelet can trail API server by up to 2 minor versions, but never be ahead.

2. **Always drain before maintenance.** `kubectl drain` gracefully evicts pods and respects PodDisruptionBudgets. Never perform node maintenance (upgrades, hardware) without draining first.

3. **etcd is your cluster's brain — back it up.** All Kubernetes state lives in etcd. Schedule regular snapshots with `etcdctl snapshot save`. Store backups off-cluster. Test your restore procedure before you need it.

4. **Disaster recovery requires multiple backup targets.** etcd covers cluster state, but PersistentVolumes need CSI snapshots, and external state (DNS, certs, IAM) needs Infrastructure-as-Code. No single tool backs up everything.

5. **CRDs extend the Kubernetes API.** Custom Resource Definitions let you add your own resource types that work with kubectl, RBAC, and the full Kubernetes API machinery.

6. **Operators automate Day-2 operations.** Operators are custom controllers that encode operational expertise (backup, failover, scaling) into software that reacts in real-time — not periodic cron jobs.

7. **Managed Kubernetes handles the hard parts.** AKS, EKS, and GKE manage the control plane, etcd, and upgrades. You focus on node pools and workloads. For production, managed is almost always the right choice unless you have specific requirements.

---

## Further Reading

- [Upgrading kubeadm Clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
- [Backing Up an etcd Cluster](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)
- [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [Operator Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)
- [Velero Documentation](https://velero.io/docs/)
- [OperatorHub.io](https://operatorhub.io/)

---

**Previous:** [Chapter 13 — Troubleshooting](13-troubleshooting.md)
**Next:** [Chapter 15 — Next Steps](15-next-steps.md)
