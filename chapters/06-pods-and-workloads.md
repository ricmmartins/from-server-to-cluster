# Chapter 6: Pods and Workloads

*"A single process is fragile. A system that manages processes — that's infrastructure."*

---

## From Processes to Pods

On a Linux server, everything comes down to processes. You start them, monitor them, restart them when they crash, and schedule them to run at specific times. You've got `systemd` for long-running services, `cron` for scheduled tasks, and `at` for one-off jobs. The whole system is a carefully orchestrated collection of processes.

Kubernetes has the same needs — but at a much larger scale. Instead of managing processes on one machine, you're managing workloads across a cluster. And just like Linux has different tools for different kinds of process management (`systemd` for daemons, `cron` for schedules, `systemctl` for service health), Kubernetes has different resource types for different kinds of workloads.

This chapter is the taxonomy of those workload resources. Think of it as the "systemd deep dive" chapter — once you understand how each workload controller works, you'll know which one to reach for in every situation.

---

## Pods: The Atomic Unit

A **Pod** is the smallest deployable unit in Kubernetes. It's not a container — it's a wrapper around one or more containers that share networking and storage.

If Linux processes are atoms, a Pod is a molecule: one or more tightly coupled processes (containers) that need to live and die together.

### Why Not Just Run Containers?

You might wonder: if Docker already runs containers, why do we need Pods?

Because some workloads need multiple processes that share resources:

- **Shared network namespace** — All containers in a Pod share the same IP address and port space. They communicate over `localhost`, just like processes on the same Linux machine.
- **Shared storage volumes** — Containers in a Pod can mount the same volumes, enabling them to share files without network overhead.
- **Co-scheduling** — All containers in a Pod are guaranteed to run on the same node. No split-brain scenarios.

### Single-Container Pods (The Common Case)

Most Pods run a single container. This is the standard pattern for web servers, APIs, workers, and most microservices:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    ports:
    - containerPort: 80
```

### Multi-Container Pods

When you need tightly coupled processes, multi-container Pods make sense. The most common patterns:

**Sidecar containers** — A helper that runs alongside the main application. Examples: log shippers, service mesh proxies, TLS termination. Native sidecar containers (using `initContainers` with `restartPolicy: Always`) were introduced as alpha in v1.28, beta in v1.29, and graduated to stable (GA) in v1.31. They ensure the sidecar starts before and stops after the main container.

**Init containers** — Run to completion *before* the main containers start. Used for setup tasks: waiting for a database to be ready, populating a shared volume, running migrations.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
  - name: init-db-check
    image: busybox:1.36
    command: ['sh', '-c', 'until nslookup db-service; do echo waiting for db; sleep 2; done']
  containers:
  - name: app
    image: nginx:1.27
    ports:
    - containerPort: 80
```

### Pod Lifecycle

A Pod goes through several phases:

| Phase | Description |
|-------|-------------|
| `Pending` | Accepted by the cluster, but containers not yet running (image pulling, scheduling) |
| `Running` | At least one container is running |
| `Succeeded` | All containers terminated successfully (exit code 0) |
| `Failed` | At least one container terminated with an error |
| `Unknown` | Pod state cannot be determined (usually a node communication issue) |

---

## Deployments and ReplicaSets

You almost never create Pods directly in production. Instead, you use **Deployments**, which manage Pods through **ReplicaSets**.

Think of it as a hierarchy:

```
Deployment → manages → ReplicaSet → manages → Pods
```

### ReplicaSets: Ensuring N Replicas

A ReplicaSet's job is simple: make sure exactly N copies of a Pod are running at all times. If a Pod dies, the ReplicaSet creates a new one. If there are too many, it kills the extras.

This is the Kubernetes equivalent of `Restart=always` in systemd — but instead of restarting one process, it maintains a fleet.

You rarely create ReplicaSets directly. Deployments create and manage them for you.

### Deployments: Rolling Updates on Top of ReplicaSets

A **Deployment** adds update management on top of ReplicaSets. When you change the Pod template (new image, new config), the Deployment creates a *new* ReplicaSet and gradually shifts traffic from the old one to the new one.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
```

Key fields:
- **`replicas: 3`** — Maintain 3 Pods at all times
- **`selector.matchLabels`** — How the Deployment finds its Pods (must match the Pod template labels)
- **`template`** — The Pod specification used for every replica

---

## DaemonSets: One Pod Per Node

A **DaemonSet** ensures that every node (or a subset of nodes) runs exactly one copy of a Pod. When a new node joins the cluster, the DaemonSet automatically schedules a Pod on it. When a node is removed, the Pod is garbage collected.

This is the Kubernetes equivalent of a system daemon — something like `syslog`, `node_exporter`, or `filebeat` that you install on every server.

Common use cases:
- **Log collection** — Fluentd or Filebeat on every node
- **Monitoring** — Prometheus node exporter on every node
- **Storage** — CSI node plugins
- **Networking** — CNI plugins, kube-proxy itself runs as a DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  labels:
    app: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      containers:
      - name: log-collector
        image: busybox:1.36
        command: ['sh', '-c', 'tail -f /var/log/syslog || tail -f /var/log/messages || sleep infinity']
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

---

## StatefulSets: Stable Identity for Stateful Workloads

Some workloads can't be treated as interchangeable cattle. Databases, distributed systems (ZooKeeper, Kafka, etcd), and any application that needs **stable identity** requires a **StatefulSet**.

StatefulSets provide guarantees that Deployments don't:

| Guarantee | What It Means |
|-----------|---------------|
| **Stable hostname** | Pods get predictable names: `web-0`, `web-1`, `web-2` (not random suffixes) |
| **Stable persistent storage** | Each Pod gets its own PersistentVolumeClaim that persists across rescheduling |
| **Ordered deployment** | Pods are created in order (0, 1, 2) and terminated in reverse order (2, 1, 0) |
| **Ordered rolling updates** | Updates proceed one Pod at a time, in reverse ordinal order |

Think of this like named database servers on Linux: you don't have "some database server" — you have `db-primary`, `db-replica-1`, `db-replica-2`, each with its own data directory on a dedicated disk.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db-headless
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: db
        image: postgres:16
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

The `volumeClaimTemplates` field is unique to StatefulSets — it creates a separate PersistentVolumeClaim for each Pod (`data-db-0`, `data-db-1`, `data-db-2`). Even if a Pod is deleted and recreated, it reattaches to the same volume.

---

## Jobs and CronJobs: Run-to-Completion Workloads

Not every workload runs forever. Sometimes you need to run a task once, or on a schedule.

### Jobs: One-Time Execution

A **Job** creates one or more Pods and ensures they run to successful completion. If a Pod fails, the Job controller creates a new one (up to a configurable `backoffLimit`).

This is the Kubernetes equivalent of the `at` command — run something once and report success or failure.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  template:
    spec:
      containers:
      - name: hello
        image: busybox:1.36
        command: ["echo", "Hello from Job"]
      restartPolicy: Never
  backoffLimit: 4
```

Key points:
- **`restartPolicy: Never`** or **`OnFailure`** — Jobs can't use `Always` (that's what Deployments are for)
- **`backoffLimit`** — Maximum retries before marking the Job as failed
- **Parallelism** — Jobs can run multiple Pods in parallel with `spec.parallelism` and `spec.completions`

### CronJobs: Scheduled Execution

A **CronJob** creates Jobs on a repeating schedule, using the same cron syntax you already know from Linux:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.36
            command:
            - /bin/sh
            - -c
            - date; echo "Hello from CronJob"
          restartPolicy: OnFailure
```

The `schedule` field uses standard cron format: `minute hour day-of-month month day-of-week`. The example above runs every minute.

---

## Probes: Health Checks for Containers

On Linux, systemd can check if a service is healthy using `ExecStartPre`, watchdog timers, or custom health check scripts. Kubernetes has a more granular system with three types of probes:

| Probe | Purpose | What Happens on Failure |
|-------|---------|------------------------|
| **Liveness** | Is the container alive? | Container is **restarted** |
| **Readiness** | Is the container ready to serve traffic? | Pod is **removed from Service endpoints** (no traffic) |
| **Startup** | Has the container finished starting? | Liveness/readiness probes are **paused** until startup succeeds |

### Probe Mechanisms

Each probe can use one of four mechanisms:

- **`httpGet`** — Makes an HTTP GET request. Success = 200-399 response code
- **`tcpSocket`** — Opens a TCP connection. Success = port is open
- **`exec`** — Runs a command inside the container. Success = exit code 0
- **`grpc`** — Performs a gRPC health check (Kubernetes v1.27+, stable)

### Example: HTTP Probes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-probed
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 2
      periodSeconds: 5
    startupProbe:
      httpGet:
        path: /
        port: 80
      failureThreshold: 30
      periodSeconds: 2
```

Key timing parameters:
- **`initialDelaySeconds`** — Wait before first probe
- **`periodSeconds`** — How often to probe
- **`failureThreshold`** — How many consecutive failures before taking action
- **`successThreshold`** — How many consecutive successes to be considered healthy (default 1; for readiness probes, can be set higher)

---

## Rolling Updates and Rollbacks

Deployments support **zero-downtime updates** through rolling updates. When you change the Pod template (new image, new env var, new config), the Deployment creates a new ReplicaSet and gradually shifts Pods:

```
Old ReplicaSet (3 pods) ──────→ New ReplicaSet (3 pods)
   ↓ scale down                    ↑ scale up
   3 → 2 → 1 → 0                  0 → 1 → 2 → 3
```

### Controlling the Rollout

Two key parameters in the Deployment's `strategy` section:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

- **`maxSurge`** — How many extra Pods can exist above the desired count during an update (absolute number or percentage). `maxSurge: 1` means you can temporarily have 4 Pods when `replicas: 3`.
- **`maxUnavailable`** — How many Pods can be unavailable during the update. `maxUnavailable: 0` means all 3 Pods must always be running — the safest option, but the slowest rollout.

### Rollback

Every time you update a Deployment, Kubernetes keeps the old ReplicaSet around (scaled to 0). This is your rollback history. You can undo an update instantly:

```bash
# Undo the last update
kubectl rollout undo deployment/nginx

# Undo to a specific revision
kubectl rollout undo deployment/nginx --to-revision=2

# Check rollout history
kubectl rollout history deployment/nginx
```

---

## Pod Disruption Budgets (PDB)

Rolling updates are voluntary disruptions — you chose to do them. But disruptions also happen during node drains (for maintenance or upgrades), cluster autoscaling, and other administrative operations.

A **PodDisruptionBudget** tells Kubernetes: "During voluntary disruptions, never let fewer than N Pods be available."

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: nginx
```

With this PDB and 3 replicas, Kubernetes will only drain one Pod at a time from the nginx Deployment. If draining a node would violate the budget, the drain operation blocks until it's safe.

You can specify either:
- **`minAvailable`** — Minimum Pods that must remain running (absolute number or percentage)
- **`maxUnavailable`** — Maximum Pods that can be down at once (absolute number or percentage)

---

## Linux ↔ Kubernetes Comparison

| Linux Concept | K8s Workload | Use Case |
|---|---|---|
| Process / process group | Pod | Atomic execution unit — one or more containers sharing network and storage |
| systemd service (`Restart=always`) | Deployment | Long-running, scalable applications with rolling updates |
| System daemon (`/usr/lib/systemd/system/`) | DaemonSet | Per-node agents (monitoring, logging, networking) |
| Named services (`db-primary`, `db-replica`) | StatefulSet | Databases, distributed systems needing stable identity |
| cron job (`crontab -e`) | CronJob | Scheduled batch tasks |
| `at` command | Job | One-time batch execution |
| systemd health checks (`ExecStartPre`, watchdog) | Probes | Liveness, readiness, and startup detection |

---

> ### ⚠️ Where the Linux Analogy Breaks
>
> **Pods are ephemeral — truly ephemeral.** On Linux, a process keeps its PID and memory until it's killed or the machine reboots. A Kubernetes Pod can be destroyed and recreated at any time — on a different node, with a different IP, with a fresh filesystem. Pods are cattle, not pets. Never store state inside a Pod and expect it to survive.
>
> **A Deployment rollback doesn't "downgrade" anything.** On Linux, `yum downgrade nginx` replaces the binary in-place. In Kubernetes, `kubectl rollout undo` creates entirely new Pods running the old image. The old Pods are gone forever — there's no in-place downgrade. It's more like "deploy the previous version" than "undo the update."
>
> **StatefulSets give stable hostnames, NOT stable IPs.** A Pod named `db-0` will always be reachable at `db-0.db-headless.default.svc.cluster.local` — but its IP address changes every time it's rescheduled. The stable identity is DNS, not network address. This is deliberate: DNS-based discovery is more resilient than hardcoded IPs.

---

## Diagnostic Lab: Working with Workloads

### Prerequisites

Make sure your Kind cluster from Chapter 5 is running:

```bash
kind get clusters
```

If not, create one:

```bash
kind create cluster --name lab-cluster --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF
```

---

### Lab 1: Create a Pod Manually

**Step 1 — Create a Pod YAML file:**

```bash
cat <<'EOF' > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    ports:
    - containerPort: 80
EOF
```

**Step 2 — Apply and inspect:**

```bash
kubectl apply -f pod.yaml
```

```bash
kubectl get pods
```

Expected output:

```
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          10s
```

**Step 3 — Describe the Pod to see events and details:**

```bash
kubectl describe pod nginx
```

Look for the `Events` section at the bottom — you'll see the scheduler assigning the Pod to a node, the kubelet pulling the image, and the container starting.

**Step 4 — Exec into the Pod:**

```bash
kubectl exec -it nginx -- /bin/sh
```

Inside the container, try:

```bash
hostname
cat /etc/os-release
curl localhost:80
exit
```

**Step 5 — Clean up:**

```bash
kubectl delete pod nginx
```

---

### Lab 2: Deployments with Rolling Updates

**Step 1 — Create a Deployment with nginx:1.26:**

```bash
cat <<'EOF' > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.26
        ports:
        - containerPort: 80
EOF
```

```bash
kubectl apply -f deployment.yaml
```

```bash
kubectl get pods -l app=nginx
```

You should see 3 Pods running.

**Step 2 — Update the image to trigger a rolling update:**

```bash
kubectl set image deployment/nginx nginx=nginx:1.27
```

**Step 3 — Watch the rollout:**

```bash
kubectl rollout status deployment/nginx
```

Expected output:

```
Waiting for deployment "nginx" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
deployment "nginx" successfully rolled out
```

**Step 4 — Check rollout history:**

```bash
kubectl rollout history deployment/nginx
```

**Step 5 — Rollback to the previous version:**

```bash
kubectl rollout undo deployment/nginx
```

Verify the rollback:

```bash
kubectl describe deployment nginx | grep Image
```

You should see `nginx:1.26` again.

**Step 6 — Clean up:**

```bash
kubectl delete deployment nginx
```

---

### Lab 3: DaemonSet — One Pod Per Node

**Step 1 — Create a DaemonSet:**

```bash
cat <<'EOF' > daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: collector
        image: busybox:1.36
        command: ['sh', '-c', 'while true; do echo "$(date) - collecting logs from $(hostname)"; sleep 60; done']
EOF
```

```bash
kubectl apply -f daemonset.yaml
```

**Step 2 — Verify one Pod per node:**

```bash
kubectl get pods -l app=log-collector -o wide
```

You should see one Pod on each node (including the control plane, since we added a toleration for it):

```
NAME                  READY   STATUS    RESTARTS   AGE   IP           NODE
log-collector-abc12   1/1     Running   0          10s   10.244.0.5   lab-cluster-control-plane
log-collector-def34   1/1     Running   0          10s   10.244.1.3   lab-cluster-worker
log-collector-ghi56   1/1     Running   0          10s   10.244.2.4   lab-cluster-worker2
```

**Step 3 — Check logs from one of the Pods:**

```bash
kubectl logs -l app=log-collector --tail=5
```

**Step 4 — Clean up:**

```bash
kubectl delete daemonset log-collector
```

---

### Lab 4: Job and CronJob

**Step 1 — Create a Job:**

```bash
cat <<'EOF' > job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  template:
    spec:
      containers:
      - name: hello
        image: busybox:1.36
        command: ["echo", "Hello from Job"]
      restartPolicy: Never
  backoffLimit: 4
EOF
```

```bash
kubectl apply -f job.yaml
```

**Step 2 — Watch the Job complete:**

```bash
kubectl get jobs
```

Expected output:

```
NAME        COMPLETIONS   DURATION   AGE
hello-job   1/1           3s         10s
```

Check the output:

```bash
kubectl logs job/hello-job
```

```
Hello from Job
```

**Step 3 — Create a CronJob:**

```bash
cat <<'EOF' > cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.36
            command:
            - /bin/sh
            - -c
            - date; echo "Hello from CronJob"
          restartPolicy: OnFailure
EOF
```

```bash
kubectl apply -f cronjob.yaml
```

**Step 4 — Wait about 60-90 seconds, then check:**

```bash
kubectl get cronjobs
kubectl get jobs --watch
```

You should see a new Job created every minute. Check the output of the latest Job:

```bash
kubectl logs job/$(kubectl get jobs --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
```

**Step 5 — Clean up:**

```bash
kubectl delete cronjob hello-cron
kubectl delete job hello-job
```

---

### Lab 5: Probes in Action

**Step 1 — Create a Pod with a liveness probe that will fail:**

This Pod creates a `/tmp/healthy` file on start, then removes it after 30 seconds — simulating an application that becomes unhealthy:

```bash
cat <<'EOF' > probe-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-test
  labels:
    test: liveness
spec:
  containers:
  - name: liveness
    image: busybox:1.36
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
EOF
```

```bash
kubectl apply -f probe-test.yaml
```

**Step 2 — Watch the Pod:**

```bash
kubectl get pod liveness-test --watch
```

For the first ~30 seconds, the Pod is healthy. After the file is removed, the liveness probe fails. After 3 consecutive failures (15 seconds), the kubelet restarts the container.

You'll see `RESTARTS` increment:

```
NAME            READY   STATUS    RESTARTS   AGE
liveness-test   1/1     Running   0          30s
liveness-test   1/1     Running   1 (2s ago) 52s
```

Press `Ctrl+C` to stop watching.

**Step 3 — Inspect the events:**

```bash
kubectl describe pod liveness-test
```

In the Events section, you'll see:

```
Warning  Unhealthy  ...  Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
Normal   Killing    ...  Container liveness failed liveness probe, will be restarted
```

**Step 4 — Create a Deployment with readiness probes:**

```bash
cat <<'EOF' > readiness-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-probed
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-probed
  template:
    metadata:
      labels:
        app: nginx-probed
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
          initialDelaySeconds: 2
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
EOF
```

```bash
kubectl apply -f readiness-deploy.yaml
```

```bash
kubectl get pods -l app=nginx-probed
```

Both Pods should show `1/1 READY` — meaning the readiness probe is passing and they're receiving traffic.

**Step 5 — Clean up:**

```bash
kubectl delete pod liveness-test
kubectl delete deployment nginx-probed
```

Clean up lab YAML files:

```bash
rm -f pod.yaml deployment.yaml daemonset.yaml job.yaml cronjob.yaml probe-test.yaml readiness-deploy.yaml
```

---

## Key Takeaways

1. **A Pod is one or more containers sharing network and storage — it's the atomic unit of Kubernetes.** Most Pods run a single container, but multi-container patterns (sidecar, init) exist for tightly coupled processes.

2. **Never create bare Pods in production.** Always use a controller (Deployment, DaemonSet, StatefulSet, Job) that manages Pod lifecycle — bare Pods won't be rescheduled if a node fails.

3. **Deployments are your default choice for stateless workloads.** They handle replication, rolling updates, and rollbacks. Under the hood, they manage ReplicaSets.

4. **DaemonSets run one Pod per node — perfect for infrastructure agents.** Log collectors, monitoring agents, and node-level plugins belong here.

5. **StatefulSets are for workloads that need stable identity.** Predictable hostnames (`pod-0`, `pod-1`), dedicated persistent storage, and ordered startup/shutdown. Use them for databases and distributed systems.

6. **Probes are your three-layer health check system.** Liveness restarts broken containers. Readiness controls traffic flow. Startup gives slow applications time to initialize. Configure them for every production workload.

7. **Pod Disruption Budgets protect your applications during planned maintenance.** Without a PDB, a `kubectl drain` can take down all your replicas simultaneously.

---

## Further Reading

- [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [DaemonSets](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
- [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Disruptions (Pod Disruption Budgets)](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)

---

**Previous:** [Chapter 5 — Your First Cluster](05-your-first-cluster.md)
**Next:** [Chapter 7 — Networking](07-networking.md)
