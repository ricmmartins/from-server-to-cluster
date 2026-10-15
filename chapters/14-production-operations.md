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

You cannot skip minor versions in a kubeadm upgrade (for example, 1.34 → 1.35, not 1.33 → 1.35). Use a supported minor and its latest supported patch, not the `.0` release pinned for this book's disposable labs.

### The kubeadm Upgrade Workflow

This is a **Debian/Ubuntu, systemd, self-managed kubeadm template**, not a Kind upgrade recipe or a managed-service procedure. It assumes an existing `pkgs.k8s.io` APT repository and trusted signing key, root access on nodes, and an administrator kubeconfig. Follow the [version-specific kubeadm guide](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) and [package repository instructions](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/). Adapt the minor if your supported upgrade target differs.

**Step 1 — Prepare and back up before changing packages:**

- Read target release notes, removed APIs, version skew, OS/runtime requirements, and the CNI/CSI/add-on compatibility matrices. Upgrade only one minor at a time; use current supported patches and validate the path in staging.
- Take and verify an off-node etcd snapshot, back up `/etc/kubernetes` and encryption keys securely, and take application-consistent data backups. Rehearse recovery; kubeadm's local backup directories are not a disaster recovery plan.
- Confirm healthy nodes/control plane, etcd quorum, CNI/DNS, workload readiness, sufficient spare capacity, and workable PDBs. Review local/`emptyDir` data and maintenance windows. A single-control-plane cluster will have API downtime.

**Step 2 — On each node, when it is that node's turn, select the target minor's APT repository and exact package version:**

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

The angle-bracket values are placeholders, **not executable version choices**. Confirm the same selected version is available for all three packages. Re-establish these variables in each node's shell; shell variables do not travel between machines. If the repository/key is absent or differently named, configure it using the official instructions instead of blindly overwriting files.

**Step 3 — Upgrade the first control plane node:**

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

Follow the CNI provider's supported upgrade sequence and upgrade the CNI as required; kubeadm does **not** upgrade it for you. Check network/DNS health. Then complete **Step 5 on this control plane node**, including drain before its kubelet upgrade, before proceeding.

**Step 4 — Upgrade each additional control plane, then each worker, one node at a time:**

Perform Step 2 on that node, then:

```bash
# On the node currently being upgraded (additional control plane OR worker):
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm="$PACKAGE_VERSION"
sudo apt-mark hold kubeadm
kubeadm version
sudo kubeadm upgrade node
```

Use `upgrade apply` only on the first control plane; `upgrade node` updates the remaining nodes' configuration and, on additional control planes, their static manifests. Complete Step 5 for **each** node before moving on. Finish all control planes before starting workers; do not run this loop concurrently across etcd members.

**Step 5 — Drain and upgrade kubelet/kubectl on EVERY node, including all control planes:**

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

Before starting the next node, verify its reported kubelet version, API/etcd health where applicable, CNI/DNS and application readiness/end-to-end traffic. `drain` waits for termination of evicted Pods, **not** replacement readiness; explicitly check each affected Deployment/StatefulSet rollout, capacity, and PDB health. After the last node, verify all component versions and held packages, and check monitoring for regressions. Stop and investigate failures rather than continuing through the fleet.

### Version Skew Policy

Kubernetes components have strict compatibility rules:

| Component | Allowed Skew from API Server |
|-----------|------------------------------|
| kubelet | Up to 3 minor versions behind |
| kube-proxy | Up to 3 minor versions behind |
| controller-manager, scheduler | Up to 1 minor version behind |
| kubectl | ±1 minor version |

For an API server at v1.35, the general skew rule permits kubelets v1.35, v1.34, v1.33, or v1.32 — never v1.36. This is a compatibility bound, not a recommendation to keep unsupported releases. In HA, the oldest API server constrains shared clients during a rollout; kubeadm and providers can impose tighter rules. Consult the [full version skew policy](https://kubernetes.io/releases/version-skew-policy/).

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

The following examples assume a kubeadm control plane with local TLS-protected etcd and the standard certificate paths. Run on the **Linux control plane host**, not your workstation; adapt endpoints and client credentials to the actual deployment. Use `etcdctl` for online snapshots and `etcdutl` for offline status/restore. Check the running etcd image/version and both tools' versions before use. For recovery, use tools and an etcd server compatible with the snapshot's **major.minor** version (prefer the same patch); do not combine restoration with an unplanned etcd minor upgrade.

Create a protected backup directory outside the live data directory, then take a snapshot:

```bash
sudo install -d -m 0700 /backup
sudo etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

The certificate flags are required because etcd uses mutual TLS for all communication.

### Verify the Backup

```bash
sudo etcdutl snapshot status /backup/etcd-snapshot.db --write-out=table
```

This reports hash, revision, key count, and size; it is a useful inspection, **not proof of recoverability or application-data consistency**. Restore checks the integrity hash of a snapshot made by `snapshot save`; rehearse an isolated restore as well. Encrypt and copy the snapshot off-node/off-cluster, with retention and access controls. Back up PKI, API server encryption configuration/keys, and other host configuration separately. etcd does not contain PersistentVolume contents.

### Single-Member Restore Example — Disposable Self-Managed Cluster Only

**Boundary:** this template is for exactly **one control plane and one local etcd member**, with kubeadm static Pods, unchanged node identity/IP/PKI, and etcd **3.6.x**. It is not an HA restore, a managed-service procedure, or a command to run in the Kind backup lab below. It requires console/root access independent of the API server, a verified snapshot, compatible `etcdutl`/`etcdctl` available on that host, and a maintenance window. For other versions/layouts, use their versioned recovery documentation.

Record `--name`, peer URLs, client URLs, certificates, `--data-dir`, `volumeMounts`, and `hostPath` from the existing etcd manifest. A restore creates a **new logical cluster**; do not use default member names/localhost peer URLs against a manifest that expects something else.

Restoring an older revision can leave Kubernetes informer/watch caches inconsistent. The official [etcd 3.6 recovery guide](https://etcd.io/docs/v3.6/op-guide/recovery/#restoring-with-revision-bump) recommends **both** `--bump-revision` and `--mark-compacted` for Kubernetes: the bump must exceed revisions written since the snapshot, and marking compacted invalidates watches so clients relist. The example bump of one billion is illustrative, not a universal bound; calculate headroom from snapshot age and write rate, and confirm your compatible `etcdutl snapshot restore --help` supports the flags.

1. Stop API writers and control plane clients, then etcd. Keep kubelet running so it observes the manifest removals. Keep saved files **outside** its watched directory; even backup files inside that directory may be processed.

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

   Do not proceed while any old etcd/API server process is still running. Preserve the original data directory for investigation; ensure `/var/lib/etcd-restored` does not already exist.

2. Restore into a **new host directory**, with explicit single-member identity. Replace the placeholder values from the recorded manifest and choose the revision bump before running:

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

   Do not bypass integrity checking. Ensure the restored directory has the owner, permissions, and (if applicable) SELinux labeling required by the etcd container.

3. Edit the **held** `/etc/kubernetes/restore-hold/etcd.yaml`. For the standard layout, change only the `etcd-data` volume's **hostPath**, while retaining its container mount and the container's data-dir:

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

   Here `/var/lib/etcd-restored` is on the **host**, mounted at `/var/lib/etcd` **inside** the container. Changing just `--data-dir` would not establish the host mount. If your original layout differs, map all three paths explicitly. Ensure name/peer/bootstrap settings match the restored single-member membership; preserve valid TLS SANs and client endpoints (or update API server configuration if endpoints change).

4. Return **only** the etcd manifest and wait for health:

   ```bash
   sudo mv /etc/kubernetes/restore-hold/etcd.yaml /etc/kubernetes/manifests/
   sudo etcdctl --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
     --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
     endpoint health
   ```

   Retry health checks while the static Pod starts; inspect runtime logs on failure. Also verify endpoint status/revision and member list using the same TLS flags. Only after confirming healthy **single-member** membership, return the remaining manifests and restart kubelet as recommended by the [Kubernetes restore guide](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#restoring-an-etcd-cluster):

   ```bash
   sudo mv /etc/kubernetes/restore-hold/kube-apiserver.yaml /etc/kubernetes/manifests/
   sudo mv /etc/kubernetes/restore-hold/kube-controller-manager.yaml /etc/kubernetes/manifests/
   sudo mv /etc/kubernetes/restore-hold/kube-scheduler.yaml /etc/kubernetes/manifests/
   sudo systemctl restart kubelet
   # Once the API server is available, using an administrator kubeconfig:
   kubectl get --raw='/readyz?verbose'
   kubectl get nodes
   ```

   Verify controllers, encrypted Secret reads, and application recovery as well. Restart/reconcile other watch consumers as needed; revision bump/compaction and client restarts address cache staleness, not lost PV data.

### HA Recovery Is a Different Runbook

**Do not repeat the single-member restore independently on each control plane.** If quorum survives, use the official [member replacement procedure](https://etcd.io/docs/v3.6/op-guide/runtime-configuration/) to replace failed members one at a time, rather than rolling the whole database back.

If quorum is irretrievably lost, fence the old cluster, stop **all** API servers, and coordinate a new logical etcd cluster from the **same snapshot**. Restore each new member with its own name, data directory, and peer URL, but a **shared full `--initial-cluster` membership list**, token, and revision-bump/compaction policy. Configure peer/client TLS and API server endpoints to match, verify quorum and all members, then restart API servers and watch consumers. External etcd requires its own operator-managed service and storage procedure. Follow the version-matched [etcd disaster recovery guide](https://etcd.io/docs/v3.6/op-guide/recovery/) and the Kubernetes restore guide; this chapter intentionally does not present a generic copy-paste HA restore.

### Scheduling Regular Backups

In production, automate etcd backups:

- **CronJob in the cluster:** Mount etcd certs and run etcdctl on a schedule
- **External cron:** A control plane node's crontab runs the backup and copies to remote storage
- **Velero:** Backs up selected Kubernetes objects **through the API**, with separately configured volume data protection; it does **not** take an etcd database snapshot

### Managed Kubernetes: etcd Is Invisible

In AKS, EKS, and GKE, the provider manages etcd; direct member access and this restore procedure are generally unavailable. Confirm the provider's control plane recovery guarantees and customer restore options. Provider-managed replication/backups are not a substitute for your own resource, volume, external-state, and application recovery plan.

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
kubectl drain <node-name> --ignore-daemonsets --timeout=10m
```

Draining does three things:
1. Cordons the node (marks unschedulable)
2. Evicts eligible pods using the eviction API, respecting PodDisruptionBudgets
3. Waits for those pods to **terminate**, not for replacements to be scheduled or Ready

The flags:
- `--ignore-daemonsets` — Leaves DaemonSet pods in place; drain does not remove them
- `--delete-emptydir-data` — Optional, explicit acceptance of deletion of `emptyDir` data; ephemeral does **not** mean safe to lose
- `--timeout=10m` — Bounds this attempt; a timeout is a failed drain, not authorization to perform maintenance

Before draining, inspect remaining capacity, affinity/taints, PVC topology, replica readiness, PDBs, and local data. Mirror/static Pods remain; unmanaged Pods can block drain and need a deliberate recovery plan. Do not routinely add `--force` or `--disable-eviction` to bypass blockers. After drain succeeds, check affected rollouts, ready Service backends, and application traffic explicitly before disrupting another node; replacements may still be Pending. See the official [drain contract](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_drain/).

### Uncordon — Return to Service

```bash
kubectl uncordon <node-name>
```

The node becomes schedulable again. Existing pods on other nodes won't automatically migrate back — but new pods can now be placed here.

### The Node Replacement Pattern

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

## Disaster Recovery Strategy

### What to Back Up

A complete disaster recovery plan covers:

| What | Why | How |
|------|-----|-----|
| **etcd** | All cluster state (Deployments, Services, RBAC) | `etcdctl snapshot save` |
| **Persistent Volumes** | Application data (databases, uploads) | CSI snapshots, Velero |
| **External state** | DNS records, TLS certificates, IAM configs | Infrastructure-as-Code (Terraform) |
| **YAML manifests** | Source of truth for all resources | Git repository (GitOps) |
| **Secrets and encryption keys** | Credentials plus ability to decrypt restored objects | External secret stores; separately protected encryption configuration/keys |

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

Velero backs up Kubernetes resource objects by querying the **API server**, not by snapshotting etcd. Volume protection depends on compatible object-store/provider plugins, CSI snapshot support and `VolumeSnapshotClass` configuration, or supported file-system backup/data-mover configuration. It is not automatically a portable copy of every PV.

Before using these illustrative commands, install a Velero version supported for your Kubernetes release, its matching provider/object-store plugin, credentials/IAM, and backup storage location. Configure the selected volume backup method and test both data and metadata restore. The install line is intentionally incomplete, not a runnable provider setup:

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

Resource backups are not strictly atomic. A disk/CSI snapshot alone may be crash-consistent rather than application-consistent; databases can require quiescing, backup hooks, transaction-log capture, or database-native backups. Coordinate the resource and data recovery points, check backup errors/partial failures, and rehearse restore into an isolated cluster with compatible APIs, CRDs, storage classes, and credentials. See [how Velero works](https://velero.io/docs/v1.18/how-velero-works/) and [backup hooks](https://velero.io/docs/v1.18/backup-hooks/).

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
| Restore from backup | `etcdutl snapshot restore` | Version-compatible, topology-specific etcd recovery |
| Maintenance mode | `kubectl cordon` + `drain` | Take node offline safely |
| `puppet`/`chef` agents | Operators | Automated application management |
| `/etc/cron.d/backup` | Velero scheduled backups | API-object backups plus configured volume protection, not etcd snapshots |
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

Use **Kind v0.31.0**, **kubectl v1.35.0**, and the Kubernetes **v1.35.0** image digest from the [Kind release](https://github.com/kubernetes-sigs/kind/releases/tag/v0.31.0). These Bash labs require a running Docker daemon with Linux containers/cgroup v2 (for example, through WSL2 on Windows), internet access, and capacity for three nodes. Use a disposable lab context. Kind is recreated with a new node image rather than upgraded in place; the production APT/restore templates above are **not** commands to run inside these Kind nodes. Results below are expected observations, not runtime test evidence.

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

Check that the other worker has capacity for the desired replicas and that any PDB allows the disruption. This demo has no `emptyDir` data to discard; do not add the deletion flag habitually.

```bash
kubectl get pdb
kubectl drain "$WORKER" --ignore-daemonsets --timeout=120s
kubectl rollout status deployment/web-fleet --timeout=120s
kubectl get pods -l app=web-fleet -o wide
# Verify desired replicas are Ready on remaining schedulable nodes.
# Drain completion alone did not establish that.
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

This is a **snapshot-only lab**, not a restore exercise. Verify the etcd image and `etcdctl`/`etcdutl` versions; these examples require compatible tools in the etcd image. If `etcdutl` is absent, inspect the copied snapshot with a matching offline tool on an administrative host; do not fall back to removed `etcdctl snapshot status/restore` commands. Do not install arbitrary tools into the control plane.

**Step 1 — Identify the etcd pod:**

```bash
kubectl get pods -n kube-system | grep etcd
# etcd-prod-ops-lab-control-plane
kubectl get pod -n kube-system etcd-prod-ops-lab-control-plane \
  -o jsonpath='{.spec.containers[0].image}{"\n"}'
kubectl exec -n kube-system etcd-prod-ops-lab-control-plane -- etcdctl version
kubectl exec -n kube-system etcd-prod-ops-lab-control-plane -- etcdutl version
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

**Step 3 — Create a snapshot (temporary staging in the writable etcd mount):**

```bash
kubectl exec -n kube-system etcd-prod-ops-lab-control-plane -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/lib/etcd/lab-snapshot.db
```

**Step 4 — Verify the backup:**

```bash
kubectl exec -n kube-system etcd-prod-ops-lab-control-plane -- etcdutl \
  snapshot status /var/lib/etcd/lab-snapshot.db --write-out=table

# Kind's hostPath is inside the node container. Copy out before deleting the cluster.
# This avoids kubectl cp, which requires tar inside the etcd image.
docker cp prod-ops-lab-control-plane:/var/lib/etcd/lab-snapshot.db ./etcd-lab-snapshot.db
# Only after successful inspection/copy, remove the staging file:
docker exec prod-ops-lab-control-plane rm /var/lib/etcd/lab-snapshot.db
```

You should see hash, revision, key count, and size. Inspection/copy is not a successful restore test. The local copy is still on the same workstation: production needs encrypted off-cluster storage and a rehearsed topology-specific restore. Protect this file as sensitive cluster data.

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

Kind does not support an in-place Kubernetes version upgrade. This lab rehearses only inspection and drain/uncordon; it does not validate the production package or control plane upgrade procedure above.

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

**Step 4 — Document the real upgrade steps:**

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

**Clean up:**

```bash
kubectl delete deployment upgrade-test
```

---

### Lab Cleanup

```bash
# Delete the Kind cluster
kind delete cluster --name prod-ops-lab
# Remove the sensitive local lab snapshot when no longer needed.
rm -f ./etcd-lab-snapshot.db
```

---

## Key Takeaways

1. **Upgrade control planes first, then workers, one minor at a time.** Install the target kubeadm before planning/applying; select the target-minor package repository and unhold/install/hold packages. Drain every node before its kubelet upgrade, including control planes, restart kubelet, and verify health between nodes. General kubelet skew is up to 3 minors behind the API server, never ahead; supported-release, kubeadm, and HA constraints still apply.

2. **Drain waits for termination, not replacement readiness.** Review capacity, PDBs and local data first; explicitly verify workload readiness afterward. `--delete-emptydir-data` accepts data loss, and static/DaemonSet pods remain outside ordinary drain eviction.

3. **Back up etcd and rehearse topology-specific recovery.** Use `etcdctl snapshot save`, compatible `etcdutl` for status/restore, encrypted off-cluster storage, and separately protected PKI/encryption keys. Single-member restore is not HA recovery; plan revision bumps/compaction and watch-cache refresh using official guidance.

4. **Disaster recovery requires multiple backup targets.** etcd covers cluster API state, not PV data. Velero backs up API objects and configured volume data, not etcd snapshots. Validate plugins, application consistency, external state, and recovery end to end.

5. **CRDs extend the Kubernetes API.** Custom Resource Definitions let you add your own resource types that work with kubectl, RBAC, and the full Kubernetes API machinery.

6. **Operators automate Day-2 operations.** Operators are custom controllers that encode operational expertise (backup, failover, scaling) into software that reacts in real-time — not periodic cron jobs.

7. **Managed Kubernetes handles the hard parts.** AKS, EKS, and GKE manage the control plane, etcd, and upgrades. You focus on node pools and workloads. For production, managed is almost always the right choice unless you have specific requirements.

---

## Further Reading

- [Upgrading kubeadm Clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [Changing the Kubernetes Package Repository](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/)
- [Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
- [Backing Up an etcd Cluster](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)
- [etcd 3.6 Disaster Recovery — Revision Bumps and Membership](https://etcd.io/docs/v3.6/op-guide/recovery/)
- [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [Operator Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)
- [Velero Documentation](https://velero.io/docs/)
- [OperatorHub.io](https://operatorhub.io/)

---

**Previous:** [Chapter 13 — Troubleshooting](13-troubleshooting.md)
**Next:** [Chapter 15 — Next Steps](15-next-steps.md)
