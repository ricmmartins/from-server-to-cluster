# Chapter 8: Storage and Persistence

*"Data that doesn't survive a reboot isn't data — it's a suggestion."*

---

## When Everything Is Ephemeral, Where Does Your Data Live?

On a Linux server, storage feels permanent. You partition a disk with `fdisk`, format it with `mkfs.ext4`, mount it to `/data`, and add an entry in `/etc/fstab` so it comes back after a reboot. Your files are right there on the metal. You can `ls /data` and see them. You can `du -sh /data` and know exactly how much space you're using. The relationship between your data and the disk is direct, physical, and obvious.

Kubernetes breaks that relationship — deliberately.

Containers are ephemeral by design. When a pod dies (and pods die all the time — crashes, rescheduling, rolling updates), its filesystem vanishes with it. Every file the container wrote, every log entry, every temporary calculation — gone. It's as if every process on your Linux server ran on a tmpfs mount that got wiped on every reboot.

For stateless applications (a web frontend, an API gateway), this is fine. The container starts, serves requests, and dies without remorse. But the moment you have a database, a file upload service, or anything that needs to remember something between restarts, you need persistence. You need storage that outlives the pod.

This chapter is about how Kubernetes solves that problem — and how the concepts map back to the Linux storage stack you already know.

---

## The Ephemeral Problem

Let's make this concrete. When you run a container, it gets a writable layer on top of the image's read-only layers (this is the overlay filesystem from Chapter 2). Any file the container creates lives in that writable layer.

The problem: that writable layer is tied to the container's lifecycle. Delete the pod, and the layer is garbage-collected. Reschedule the pod to a different node, and it starts fresh with a clean writable layer.

This is like running every service on your Linux server with its working directory on a `tmpfs` mount:

```bash
mount -t tmpfs tmpfs /var/lib/myapp
systemctl start myapp
# myapp writes data to /var/lib/myapp...
systemctl stop myapp
# Data is gone. Forever.
```

Nobody would do this on a Linux server (at least not for data they care about). But in Kubernetes, this is the default behavior. You have to explicitly opt into persistence.

---

## Volume Types

Kubernetes provides several volume types, each solving a different storage problem. Let's walk through them from simplest to most powerful.

### emptyDir — Temporary Shared Storage

An `emptyDir` volume is created when a pod is assigned to a node and exists as long as the pod runs on that node. When the pod is removed, the data in the `emptyDir` is deleted permanently.

Think of it as a `/tmp` directory shared between all containers in the same pod:

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

Two containers, one volume. The writer puts a file in `/shared`, and the reader can see it. This is exactly like two processes on a Linux box sharing `/tmp`.

You can also back an `emptyDir` with memory instead of disk:

```yaml
volumes:
- name: cache
  emptyDir:
    medium: Memory
    sizeLimit: 64Mi
```

This creates a tmpfs mount — fast, but counts against the container's memory limit. Useful for scratch space, caches, or shared sockets between sidecar containers.

### hostPath — Direct Host Filesystem Access

A `hostPath` volume mounts a file or directory from the host node's filesystem into a pod. This is the Kubernetes equivalent of a bind mount:

```yaml
volumes:
- name: host-data
  hostPath:
    path: /var/log/host
    type: Directory
```

On a Linux server, this is like:

```bash
mount --bind /var/log/host /container/logs
```

**Warning:** `hostPath` is dangerous in production. Your pod is now coupled to a specific node's filesystem. If the pod gets rescheduled to a different node, the data isn't there. It also creates security risks — a container with access to the host filesystem can do a lot of damage. Use it for development and specific system-level DaemonSets (like log collectors), but never for application data in production.

### PersistentVolume (PV) — Cluster-Level Storage

A PersistentVolume is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned by a StorageClass. Think of it as a disk partition that the cluster admin has prepared and made available:

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

This is like the admin running `fdisk`, `mkfs`, and adding the entry to `/etc/fstab` — preparing storage for use, but not yet mounting it to any specific application.

A PV exists independently of any pod. It's a cluster-level resource (not namespaced), just like a physical disk exists independently of any process that might use it.

### PersistentVolumeClaim (PVC) — Requesting Storage

A PersistentVolumeClaim is how a pod requests storage. It's like a user saying "I need 1GB of read-write storage" without caring about which specific disk provides it:

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

Kubernetes looks at the available PVs, finds one that satisfies the claim (enough capacity, compatible access mode), and binds them together. Once bound, the PVC is exclusively tied to that PV.

A pod then references the PVC in its volume spec:

```yaml
volumes:
- name: data
  persistentVolumeClaim:
    claimName: my-pvc
```

The separation is intentional. The admin manages PVs (the "what storage exists"). The developer manages PVCs (the "what storage I need"). The pod just says "give me the storage that PVC promised." This decouples infrastructure from application concerns.

---

## StorageClasses — Dynamic Provisioning

In the PV/PVC model above, someone has to manually create PVs before pods can claim them. That's fine for small clusters, but it doesn't scale. Imagine an admin manually creating disk partitions every time a developer needs storage — that's a bottleneck.

StorageClasses solve this by enabling dynamic provisioning. Instead of pre-creating PVs, you define a StorageClass that describes *how* to create storage on demand:

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

Now when a PVC references this StorageClass, the provisioner automatically creates the backing storage (in this case, an Azure Premium SSD), creates a PV for it, and binds it to the PVC. No admin intervention needed.

This is like LVM thin provisioning on Linux — you define a pool and a policy, and logical volumes are carved out on demand as applications request them. Or think of it as the difference between the admin manually partitioning disks versus setting up LVM to auto-allocate from a volume group.

Most Kubernetes distributions come with a default StorageClass. Kind uses `standard` backed by the Rancher local-path-provisioner. Cloud providers ship their own (Azure: `managed-premium`, AWS: `gp3`, GKE: `standard-rwo`). You can see what's available with:

```bash
kubectl get storageclass
```

The default StorageClass (marked with `(default)`) is used when a PVC doesn't specify one.

---

## Access Modes

PersistentVolumes support different access modes that control how many nodes can mount the volume simultaneously:

| Access Mode | Abbreviation | Description |
|-------------|-------------|-------------|
| ReadWriteOnce | RWO | Mounted as read-write by a single node |
| ReadOnlyMany | ROX | Mounted as read-only by many nodes |
| ReadWriteMany | RWX | Mounted as read-write by many nodes |
| ReadWriteOncePod | RWOP | Mounted as read-write by a single pod (K8s 1.27+) |

Think of these like NFS mount options on Linux:

- RWO is like mounting an ext4 partition — only one machine can mount it read-write.
- ROX is like `mount -o ro` on NFS — many machines can read, none can write.
- RWX is like a full NFS read-write mount — many machines can read and write.
- RWOP is the strictest — even if multiple pods run on the same node, only one can mount it.

Not all storage backends support all access modes. Local disks typically only support RWO. Network filesystems like NFS or Azure Files support RWX. Check your CSI driver's documentation for what's supported.

---

## Reclaim Policies

When a PVC is deleted, what happens to the underlying PV and its data? The reclaim policy decides:

| Policy | Behavior | Linux Analogy |
|--------|----------|---------------|
| Retain | PV and data are preserved. Admin must manually clean up. | `umount /data` — the partition and its files still exist |
| Delete | PV and the backing storage are both deleted. | `umount /data && wipefs -a /dev/sdb1` — everything is gone |

The `Retain` policy is safer — you won't accidentally lose data. But it means someone has to manually reclaim the PV after the PVC is gone. The `Delete` policy is convenient for ephemeral workloads but dangerous if you delete a PVC by mistake.

---

## CSI — The Container Storage Interface

CSI is the plugin system that lets Kubernetes talk to any storage backend. It's like Linux device drivers, but for storage. Instead of Kubernetes knowing about every possible storage system (Azure Disks, AWS EBS, NFS, Ceph, etc.), it defines a standard interface that storage providers implement.

Common CSI drivers:

| Storage Backend | CSI Driver |
|----------------|------------|
| Azure Disk | disk.csi.azure.com |
| Azure Files | file.csi.azure.com |
| AWS EBS | ebs.csi.aws.com |
| GCE Persistent Disk | pd.csi.storage.gke.io |
| NFS | nfs.csi.k8s.io |
| Ceph | rbd.csi.ceph.com |

On a Linux server, you install a kernel module to support a new filesystem (`modprobe nfs`). In Kubernetes, you install a CSI driver to support a new storage backend. The principle is the same — a plugin-based architecture that separates the core system from the storage implementation.

---

## Volume Snapshots

Volume snapshots are point-in-time copies of a PersistentVolume's data. If you've used LVM snapshots on Linux (`lvcreate --snapshot`), the concept is identical.

In Kubernetes, snapshots are managed through three objects:

- **VolumeSnapshot** — the request to take a snapshot (like running `lvcreate --snapshot`)
- **VolumeSnapshotContent** — the actual snapshot data (like the LVM snapshot logical volume)
- **VolumeSnapshotClass** — the policy for how snapshots are taken (like the LVM snapshot settings)

Snapshots require a CSI driver that supports them — not all do. They're implemented as CRDs (Custom Resource Definitions), not built into the core Kubernetes API. This is a more advanced feature that we mention here for completeness; your Kind cluster won't have snapshot support out of the box.

---

## Linux ↔ Kubernetes Comparison

| Linux Concept | K8s Equivalent | Notes |
|---------------|----------------|-------|
| `/etc/fstab` | PersistentVolume spec | Defines available storage |
| `mount /dev/sdb1 /data` | PVC volumeMount | Attaches storage to a pod |
| LVM logical volume | PersistentVolume | Abstracted block/file storage |
| `mount -t nfs` | PV with NFS provisioner | Network-attached storage |
| tmpfs | emptyDir (medium: Memory) | Ephemeral in-memory storage |
| bind mount | hostPath | Direct host filesystem access |
| LVM thin provisioning | StorageClass + dynamic provisioning | On-demand allocation |
| `lvcreate --snapshot` | VolumeSnapshot | Point-in-time copy |
| `/etc/fstab` options (ro, rw) | Access Modes (ROX, RWO, RWX) | Read/write permissions |
| `fdisk` + `mkfs` + `mount` | Static PV provisioning | Admin manually prepares storage |
| LVM auto-extend | Dynamic provisioning via StorageClass | Automatic storage creation on demand |

---

> ### 🔀 Where the Linux Analogy Breaks
>
> - **Storage is decoupled from compute.** In Linux, storage is directly attached to the machine — your SSD is physically inside (or cable-connected to) the server. In Kubernetes, storage is a cluster resource that can be attached to any pod on any node, thanks to CSI drivers and network-attached storage. You don't think about "which server has the disk" — you think about "which pod needs the data."
>
> - **PVCs have their own lifecycle.** A PVC isn't just a mount request that disappears when you unmount. It's a persistent Kubernetes object that exists independently of any pod. Deleting a pod doesn't delete its PVC (unless explicit cascading is configured). This means data survives pod restarts, rescheduling, and even deliberate pod deletion.
>
> - **Dynamic provisioning eliminates the admin workflow.** On Linux, the storage workflow is `fdisk` → `mkfs` → `mount` → update `/etc/fstab`. A human runs each step. With StorageClasses, you skip all of that — the provisioner creates the backing storage, formats it, and makes it available automatically. The developer writes a PVC, and the storage appears. This is fundamentally different from the traditional sysadmin experience.

---

## Diagnostic Lab: Storage Hands-On

### Prerequisites

Make sure you have a Kind cluster running. If not, create one:

```bash
kind create cluster --name storage-lab
```

### Lab 1: emptyDir — Shared Temporary Storage

Two containers in one pod sharing an `emptyDir` volume. The writer creates content, and a web server serves it.

Save this as `emptydir-pod.yaml`:

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

Apply and verify:

```bash
# Create the pod
kubectl apply -f emptydir-pod.yaml

# Wait for it to be running
kubectl wait --for=condition=Ready pod/emptydir-demo --timeout=60s

# Verify the writer created the file
kubectl exec emptydir-demo -c content-writer -- cat /usr/share/nginx/html/index.html

# Verify the web server can serve it
kubectl exec emptydir-demo -c web-server -- curl -s http://localhost:80

# Clean up
kubectl delete pod emptydir-demo
```

**What just happened:** Two containers in the same pod shared a filesystem. The busybox sidecar wrote an HTML file, and the nginx container served it. Neither container knew about the other — they just agreed on a mount path. This is the sidecar pattern in action.

### Lab 2: PersistentVolume and PersistentVolumeClaim — Manual Provisioning

Create a PV backed by `hostPath` (suitable for Kind local testing), then claim it with a PVC.

Save this as `pv-demo.yaml`:

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

Note the `storageClassName: ""` — this tells Kubernetes "don't use dynamic provisioning; bind to an existing PV."

```bash
# Create the PV and PVC
kubectl apply -f pv-demo.yaml

# Check that the PVC is bound to the PV
kubectl get pv
kubectl get pvc
```

You should see the PVC status as `Bound` and the PV showing `manual-pvc` as its claim.

Now deploy a pod that writes data to the volume. Save as `pv-writer.yaml`:

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
# Deploy the writer pod
kubectl apply -f pv-writer.yaml
kubectl wait --for=condition=Ready pod/pv-writer --timeout=60s

# Verify data was written
kubectl exec pv-writer -- cat /data/message.txt

# Delete the pod (but NOT the PVC)
kubectl delete pod pv-writer

# Deploy a new pod that reads from the same PVC
# Save as pv-reader.yaml:
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

# The data survived the pod deletion!
kubectl exec pv-reader -- cat /data/message.txt

# Clean up
kubectl delete pod pv-reader
kubectl delete pvc manual-pvc
kubectl delete pv manual-pv
```

**What just happened:** You created storage (PV), requested it (PVC), wrote data, destroyed the pod, created a new pod, and the data was still there. The PVC kept the PV bound and the data intact even though the original pod was gone.

### Lab 3: StorageClass — Dynamic Provisioning

Kind ships with a default StorageClass called `standard` backed by the Rancher local-path-provisioner. Let's use it.

```bash
# Inspect the default StorageClass
kubectl get storageclass
```

You should see something like:

```
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      AGE
standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   5m
```

Now create a PVC without specifying a PV — the StorageClass will provision one automatically. Save as `dynamic-pvc.yaml`:

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
# Create the PVC
kubectl apply -f dynamic-pvc.yaml

# Check PVC status — it will be Pending until a pod uses it
# (because the volumeBindingMode is WaitForFirstConsumer)
kubectl get pvc dynamic-pvc

# Deploy a pod that uses the PVC
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

# Now check again — the PVC should be Bound
kubectl get pvc dynamic-pvc

# And a PV was automatically created!
kubectl get pv

# Verify the data
kubectl exec dynamic-demo -- cat /data/proof.txt

# Clean up
kubectl delete pod dynamic-demo
kubectl delete pvc dynamic-pvc
```

**What just happened:** You created a PVC without creating a PV first. When the pod was scheduled, the `standard` StorageClass automatically provisioned a PV and bound it to your PVC. No admin intervention. This is dynamic provisioning — the pattern used in production clusters everywhere.

### Lab 4: StatefulSet with Per-Replica Storage

StatefulSets are how Kubernetes handles stateful applications (databases, message queues). Each replica gets its own persistent storage that follows it across restarts.

Save as `statefulset-demo.yaml`:

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
# Deploy the StatefulSet
kubectl apply -f statefulset-demo.yaml

# Wait for all pods to be ready
kubectl rollout status statefulset/stateful-demo --timeout=120s

# Each pod has its own identity and its own PVC
kubectl get pods -l app=stateful-demo
kubectl get pvc

# Check each pod's unique data
kubectl exec stateful-demo-0 -- cat /data/identity.txt
kubectl exec stateful-demo-1 -- cat /data/identity.txt
kubectl exec stateful-demo-2 -- cat /data/identity.txt

# Delete a pod — the StatefulSet recreates it with the SAME name and SAME PVC
kubectl delete pod stateful-demo-1
kubectl wait --for=condition=Ready pod/stateful-demo-1 --timeout=60s

# The data from before the deletion is still there!
kubectl exec stateful-demo-1 -- cat /data/identity.txt

# Clean up
kubectl delete statefulset stateful-demo
kubectl delete svc stateful-svc
kubectl delete pvc data-stateful-demo-0 data-stateful-demo-1 data-stateful-demo-2
```

**What just happened:** The StatefulSet created three pods (stateful-demo-0, -1, -2), each with its own PVC (data-stateful-demo-0, -1, -2). When you deleted pod -1, the StatefulSet recreated it with the same name and reattached it to the same PVC. The data survived. This is how databases run on Kubernetes — each replica has its own persistent identity and storage.

### Clean Up

```bash
# Delete the Kind cluster when done
kind delete cluster --name storage-lab

# Remove the YAML files
rm -f emptydir-pod.yaml pv-demo.yaml pv-writer.yaml dynamic-pvc.yaml statefulset-demo.yaml
```

---

## Key Takeaways

1. **Container storage is ephemeral by default.** Anything written inside a container's filesystem is lost when the pod dies. If you need data to survive, you must use volumes.

2. **emptyDir is for temporary, pod-scoped shared storage.** It's perfect for sidecar patterns where containers in the same pod need to share files, but the data doesn't need to outlive the pod.

3. **PersistentVolumes and PersistentVolumeClaims separate storage management from consumption.** Admins provision PVs; developers request storage via PVCs. This separation of concerns is a core Kubernetes design principle.

4. **StorageClasses enable dynamic provisioning.** In production, you almost never create PVs manually. You define a StorageClass, and the provisioner creates storage on demand when PVCs are submitted.

5. **Access Modes control concurrent access.** RWO for single-node writes, ROX for multi-node reads, RWX for multi-node writes. Your storage backend determines which modes are available.

6. **StatefulSets give each replica its own persistent storage.** Through `volumeClaimTemplates`, each pod in a StatefulSet gets a dedicated PVC that follows it across restarts and rescheduling.

7. **Reclaim policies determine what happens to data after PVCs are deleted.** Use `Retain` for data you can't afford to lose, `Delete` for ephemeral storage that should be cleaned up automatically.

---

## Further Reading

- [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Dynamic Volume Provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
- [CSI Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#csi)
- [Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Rancher Local Path Provisioner (used by Kind)](https://github.com/rancher/local-path-provisioner)

---

**Previous:** [Chapter 7 — Networking and Services](07-networking-and-services.md)
**Next:** [Chapter 9 — Configuration and Secrets](09-configuration-and-secrets.md)
