# Chapter 12: Scaling and Observability

*"You can't improve what you can't measure. And you can't scale what you don't understand."*

---

## From top to kubectl top: Monitoring at Cluster Scale

On a Linux server, resource management is hands-on. You open `top` and see every process, its CPU percentage, its memory usage. You tweak cgroups to limit a runaway container. You set up `sar` to track historical trends. When traffic spikes, you SSH in, check the load, and maybe spin up another VM behind a load balancer. It's manual, reactive, and tied to machines you can name.

Kubernetes takes these same concerns — *how much CPU is this thing using? How much memory does it have? What happens when demand exceeds capacity?* — and turns them into declarative, automated systems. Instead of checking `top` and manually starting new processes, you define resource **requests** and **limits** in your pod spec, and Kubernetes handles scheduling, enforcement, and even automatic scaling.

The concepts are the same. CPU is still CPU. Memory is still memory. OOM kills still happen. But the mechanism shifts from "admin watches a dashboard and reacts" to "the system watches metrics and acts." Your job isn't to babysit servers anymore — it's to define the rules and let Kubernetes enforce them.

For a Linux admin, this chapter is where your `cgroups`, `top`, `sar`, and `ulimit` knowledge becomes directly useful — because Kubernetes uses those exact kernel features under the hood. It just wraps them in YAML.

---

## Resource Management: Requests and Limits

Every container in Kubernetes can declare how much CPU and memory it needs (requests) and how much it's allowed to use (limits). This is how Kubernetes does capacity planning and enforcement — and it maps directly to Linux cgroups.

### CPU: Measured in Millicores

Kubernetes measures CPU in **millicores** (also called millicpu). 1000m = 1 full CPU core.

- `100m` = 10% of one core
- `500m` = half a core
- `2000m` or `2` = two full cores

Under the hood, this configures the container's cgroup `cpu.max` value. A limit of `500m` means the container gets throttled if it tries to use more than half a core.

### Memory: Measured in Bytes

Memory is specified in bytes, typically using suffixes:

- `64Mi` = 64 mebibytes (67,108,864 bytes)
- `256Mi` = 256 mebibytes
- `1Gi` = 1 gibibyte

Under the hood, this configures the container's cgroup `memory.max`. If a container exceeds its memory limit, the Linux OOM killer terminates it immediately — no warning, no graceful shutdown.

### Requests vs. Limits

| | Requests | Limits |
|---|---------|--------|
| **What it means** | Guaranteed minimum | Maximum allowed |
| **Who uses it** | Scheduler (for pod placement) | Kubelet/kernel (for enforcement) |
| **CPU behavior** | Node must have this much CPU available to schedule the pod | Container is throttled beyond this |
| **Memory behavior** | Node must have this much memory available to schedule the pod | Container is OOMKilled beyond this |
| **Linux cgroup equivalent** | `cpu.weight` / scheduling guarantee | `cpu.max` / `memory.max` |

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: app
    image: nginx:1.27
    resources:
      requests:
        cpu: "100m"        # Guarantee: 10% of a core
        memory: "64Mi"     # Guarantee: 64 MiB
      limits:
        cpu: "200m"        # Ceiling: 20% of a core
        memory: "128Mi"    # Ceiling: 128 MiB (OOMKill if exceeded)
```

**Critical rule:** Always set requests. Always set limits. Pods without resource specifications are the first to be evicted under pressure, and they make capacity planning impossible.

---

## QoS Classes: Eviction Priority

Based on how you set requests and limits, Kubernetes assigns each pod a **Quality of Service (QoS) class** that determines its eviction priority when the node runs low on resources. This is the Kubernetes equivalent of Linux's OOM killer priorities (`oom_score_adj`).

| QoS Class | Condition | Eviction Priority | Linux Analogy |
|-----------|-----------|-------------------|---------------|
| **Guaranteed** | `requests == limits` for all containers | Last to be evicted | `oom_score_adj = -997` |
| **Burstable** | `requests < limits` for at least one container | Medium priority | `oom_score_adj` varies |
| **BestEffort** | No requests or limits set at all | **First to be evicted** | `oom_score_adj = 1000` |

```yaml
# Guaranteed QoS — requests equal limits
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

```yaml
# Burstable QoS — requests less than limits
resources:
  requests:
    cpu: "100m"
    memory: "64Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

```yaml
# BestEffort QoS — no requests or limits
# (Don't do this in production!)
resources: {}
```

When a node runs out of memory, the kubelet evicts pods in this order: BestEffort first, then Burstable (starting with the ones using the most memory relative to their request), then Guaranteed last. It's triage — just like the Linux OOM killer, but with Kubernetes-level awareness of pod priorities.

---

## LimitRange: Namespace-Level Defaults

What happens when a developer deploys a pod without specifying resources? Without guardrails, it becomes a BestEffort pod that can consume unlimited resources and is the first to be evicted. **LimitRange** solves this by setting default resource values and constraints per namespace.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: default
spec:
  limits:
  - type: Container
    default:              # Applied if no limits are specified
      cpu: "200m"
      memory: "128Mi"
    defaultRequest:       # Applied if no requests are specified
      cpu: "100m"
      memory: "64Mi"
    max:                  # Maximum limits a container can request
      cpu: "1"
      memory: "512Mi"
    min:                  # Minimum requests a container must specify
      cpu: "50m"
      memory: "32Mi"
```

Think of LimitRange as `ulimit` for a Kubernetes namespace — it sets default and maximum resource constraints so no single pod can be a bad neighbor.

---

## Horizontal Pod Autoscaler (HPA)

On a Linux server, scaling is manual: you watch `top`, decide the server is overloaded, spin up another VM, configure the load balancer, and deploy the app again. It takes minutes to hours.

The Horizontal Pod Autoscaler (HPA) makes this automatic. It watches metrics (CPU utilization, memory usage, or custom metrics), compares them to targets you define, and adjusts the number of replicas up or down.

### How HPA Works

The HPA controller runs a control loop (every 15 seconds by default) that:

1. Fetches current metrics from the Metrics Server (or a custom metrics API)
2. Compares current utilization to the target
3. Calculates the desired number of replicas using this formula:

```
desiredReplicas = ceil(currentReplicas × (currentMetricValue / targetMetricValue))
```

For example: if you have 2 replicas at 80% CPU utilization and your target is 50%, the HPA calculates:

```
desiredReplicas = ceil(2 × (80 / 50)) = ceil(3.2) = 4
```

It scales from 2 to 4 replicas to bring average utilization back toward 50%.

### Creating an HPA

```bash
# Imperative (quick)
kubectl autoscale deployment my-app --cpu-percent=50 --min=1 --max=10

# Or declarative (recommended)
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### Scaling Behavior and Cooldowns

HPA includes configurable scaling policies to prevent thrashing (rapid scale-up/scale-down cycles):

- **Scale-up:** By default, HPA can scale up quickly — it responds within 15–30 seconds of detecting increased load.
- **Scale-down:** By default, HPA waits 5 minutes (the stabilization window) before scaling down. This prevents premature scale-down if a load spike is temporary.

You can customize this behavior:

```yaml
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60       # Wait 60s before scaling up further
      policies:
      - type: Percent
        value: 100                          # Can double replicas in one step
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300      # Wait 5 min before scaling down
      policies:
      - type: Percent
        value: 10                           # Remove at most 10% of replicas per step
        periodSeconds: 60
```

---

## Vertical Pod Autoscaler (VPA)

While HPA scales *out* (more replicas), the Vertical Pod Autoscaler scales *up* (more resources per pod). VPA watches actual resource usage over time and recommends — or automatically applies — adjusted CPU and memory requests/limits.

### HPA vs. VPA

| | HPA | VPA |
|---|-----|-----|
| **Scales** | Number of replicas (horizontal) | Resource requests/limits (vertical) |
| **When to use** | Stateless workloads that can run multiple instances | Workloads where the right resource size is unclear |
| **Requires** | Metrics Server | VPA controller (separate install) |
| **Production readiness** | GA, widely used | Mature, but auto-update mode restarts pods |

> **Note:** VPA is a separate project, not built into Kubernetes core. It requires installing the VPA controller components. For this book's labs, we focus on HPA — but VPA is an important concept to understand for production right-sizing.

---

## Metrics Server: The Foundation

Both HPA and `kubectl top` depend on the **Metrics Server** — a lightweight, in-memory aggregator that collects CPU and memory metrics from the kubelets on each node.

### What Metrics Server Provides

- `kubectl top nodes` — CPU and memory usage per node
- `kubectl top pods` — CPU and memory usage per pod
- Metrics API (`metrics.k8s.io`) consumed by HPA

### What Metrics Server Doesn't Provide

- Historical data (it only shows current values)
- Custom metrics (application-level metrics)
- Disk, network, or other resource metrics

Think of Metrics Server as `top` for your cluster: real-time, lightweight, and limited to CPU and memory. For anything more, you need a full monitoring stack.

---

## Prometheus + Grafana: Production Monitoring

If Metrics Server is `top`, then **Prometheus** is `sar` + `collectd` + `nagios` rolled into one — a full-featured metrics collection, storage, and alerting system. Combined with **Grafana** for visualization, it's the de facto monitoring stack for Kubernetes.

### How Prometheus Works

Prometheus uses a **pull model**: it scrapes metrics endpoints at regular intervals (typically every 15–30 seconds). Kubernetes pods expose metrics at `/metrics` endpoints in a standard format, and Prometheus discovers targets via Kubernetes service discovery.

Key concepts:

| Concept | Description |
|---------|------------|
| **Scraping** | Prometheus periodically pulls metrics from targets |
| **PromQL** | Query language for slicing and aggregating metric data |
| **Alertmanager** | Routes alerts to Slack, PagerDuty, email, etc. |
| **Time series** | Every metric is stored with timestamps for historical analysis |

### Key Metrics to Monitor

| Metric | What It Tells You |
|--------|-------------------|
| `container_cpu_usage_seconds_total` | CPU consumption per container |
| `container_memory_working_set_bytes` | Actual memory in use (what the OOM killer watches) |
| `kube_pod_status_phase` | How many pods are in each phase (Running, Pending, Failed) |
| `kube_deployment_status_replicas_available` | Whether your deployments have the expected replica count |
| `node_cpu_seconds_total` | Node-level CPU utilization |
| `node_memory_MemAvailable_bytes` | Available memory per node |

### The kube-prometheus-stack

The easiest way to deploy Prometheus and Grafana on Kubernetes is the **kube-prometheus-stack** Helm chart. It installs Prometheus, Grafana, Alertmanager, node-exporter, kube-state-metrics, and a set of pre-built dashboards and alert rules — all in one Helm install.

---

## Linux ↔ Kubernetes Comparison

| Linux Concept | Kubernetes Equivalent | Notes |
|---------------|----------------------|-------|
| cgroups `cpu.max` | `resources.limits.cpu` | Hard CPU ceiling |
| cgroups `memory.max` | `resources.limits.memory` | OOMKill if exceeded |
| `nice`/`renice` priority | QoS classes (Guaranteed, Burstable, BestEffort) | Eviction priority under pressure |
| OOM killer (`oom_score_adj`) | Pod eviction | Lowest QoS evicted first |
| `top`, `htop` | `kubectl top`, Metrics Server | Real-time resource usage |
| `sar`, `collectd` | Prometheus | Historical metrics collection and alerting |
| Grafana/Cacti | Grafana | Visualization and dashboards |
| Manual scaling (add VMs) | HPA | Automatic replica scaling based on metrics |
| `ulimit` | LimitRange | Default/max resource constraints per namespace |

---

> ### 📊 Where the Linux Analogy Breaks
>
> - **Linux scaling is vertical or manual. Kubernetes scaling is automatic and horizontal.** On Linux, when a server is overloaded, you either add more RAM/CPU (vertical scaling) or manually provision new servers and configure load balancers (horizontal scaling). Both require human intervention and take minutes to hours. HPA reacts in seconds and handles everything — replica creation, scheduling, load distribution — without any manual steps. The infrastructure handles placement automatically.
>
> - **`kubectl top` shows now, not then.** There's no built-in `sar` or `vmstat` equivalent that tracks resource usage over time. Metrics Server is purely real-time — a snapshot of this moment. If you need "what happened at 3 AM when the alerts fired?" you need Prometheus or a similar time-series database. Kubernetes doesn't ship with historical monitoring.
>
> - **Resource limits in Kubernetes are hard — there is no safety net.** On Linux, if a process exceeds available physical memory, the system can swap to disk, giving you time to react. In Kubernetes, there is no swap by default. Exceeding a memory limit triggers an immediate OOMKill — the container is terminated instantly with no graceful shutdown for the limit violation. Set your limits too low, and your pods die. Set them too high, and you waste cluster capacity. Getting resource limits right is one of the hardest operational challenges in Kubernetes.

---

## Diagnostic Lab: Scaling and Observability in Practice

### Prerequisites

Create a Kind cluster:

```bash
kind create cluster --name scaling-lab
```

### Lab 1: Resource Requests, Limits, and QoS Classes

**Step 1 — Deploy pods with different resource configurations:**

```bash
# Guaranteed QoS (requests == limits)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
  - name: app
    image: nginx:1.27
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "100m"
        memory: "64Mi"
EOF

# Burstable QoS (requests < limits)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
spec:
  containers:
  - name: app
    image: nginx:1.27
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "200m"
        memory: "128Mi"
EOF

# BestEffort QoS (no requests or limits)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
spec:
  containers:
  - name: app
    image: nginx:1.27
EOF
```

**Step 2 — Check the QoS class assigned to each pod:**

```bash
kubectl get pod guaranteed-pod -o jsonpath='{.status.qosClass}'
echo ""
kubectl get pod burstable-pod -o jsonpath='{.status.qosClass}'
echo ""
kubectl get pod besteffort-pod -o jsonpath='{.status.qosClass}'
echo ""
```

Expected output: `Guaranteed`, `Burstable`, `BestEffort`.

**Step 3 — Inspect the resource allocations:**

```bash
kubectl describe pod guaranteed-pod | grep -A6 "Limits:"
kubectl describe pod burstable-pod | grep -A6 "Limits:"
kubectl describe pod besteffort-pod | grep -A6 "Limits:"
```

**Step 4 — Clean up the QoS demo pods:**

```bash
kubectl delete pod guaranteed-pod burstable-pod besteffort-pod
```

### Lab 2: Install Metrics Server on Kind

Kind doesn't ship with Metrics Server. Let's install it.

**Step 1 — Apply the Metrics Server manifest:**

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**Step 2 — Patch the deployment to add `--kubelet-insecure-tls`:**

Kind uses self-signed certificates, so Metrics Server needs to skip TLS verification:

```bash
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
```

**Step 3 — Wait for Metrics Server to be ready:**

```bash
kubectl wait --for=condition=Available deployment/metrics-server -n kube-system --timeout=120s
```

**Step 4 — Verify it's collecting metrics (may take 30–60 seconds):**

```bash
# Node-level metrics
kubectl top nodes

# Pod-level metrics (run a pod first if needed)
kubectl run metrics-test --image=nginx:1.27 --restart=Never
sleep 30
kubectl top pods
```

If `kubectl top` returns "Metrics not available yet," wait another 30 seconds — Metrics Server needs a couple of collection cycles.

**Step 5 — Clean up the test pod:**

```bash
kubectl delete pod metrics-test
```

### Lab 3: HPA in Action

This lab follows the official Kubernetes HPA walkthrough using the `php-apache` example.

**Step 1 — Deploy the php-apache application:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      run: php-apache
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "200m"
          limits:
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache
EOF
```

Wait for the deployment to be ready:

```bash
kubectl wait --for=condition=Available deployment/php-apache --timeout=120s
```

**Step 2 — Create an HPA targeting 50% CPU utilization:**

```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

**Step 3 — Verify the HPA is active:**

```bash
kubectl get hpa
```

You should see the HPA with `TARGETS` showing current CPU usage vs. the 50% target. If it shows `<unknown>/50%`, wait for Metrics Server to collect data.

**Step 4 — Generate load:**

Open a separate terminal (or run in the background) to generate continuous HTTP requests:

```bash
kubectl run load-generator --image=busybox:1.37 --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
```

**Step 5 — Watch the HPA scale up:**

```bash
kubectl get hpa php-apache --watch
```

Within 1–2 minutes, you should see:
- CPU utilization climbing above 50%
- `REPLICAS` increasing as HPA adds more pods
- Eventually, utilization stabilizing around the 50% target

Press `Ctrl+C` to stop watching, then check the pods:

```bash
kubectl get pods -l run=php-apache
```

You should see multiple replicas running.

**Step 6 — Stop the load and watch scale-down:**

```bash
kubectl delete pod load-generator
```

Now watch the HPA again:

```bash
kubectl get hpa php-apache --watch
```

After approximately 5 minutes (the default stabilization window), replicas will start scaling back down toward 1. This delay prevents premature scale-down in case of intermittent load.

Press `Ctrl+C` to stop watching.

**Step 7 — Inspect HPA events:**

```bash
kubectl describe hpa php-apache
```

The events section shows every scaling decision — when it scaled up, why, and when it scaled down.

### Lab 4: Quick Prometheus and Grafana Setup (Optional)

This lab requires Helm. If you completed Chapter 10, you already have it installed.

**Step 1 — Add the Prometheus community Helm repository:**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

**Step 2 — Install the kube-prometheus-stack:**

```bash
helm install kube-prom prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.resources.requests.memory=256Mi \
  --set prometheus.prometheusSpec.resources.requests.cpu=100m
```

This installs Prometheus, Grafana, Alertmanager, node-exporter, and kube-state-metrics. It takes a couple of minutes.

**Step 3 — Wait for all pods to be ready:**

```bash
kubectl wait --for=condition=Ready pods --all -n monitoring --timeout=300s
kubectl get pods -n monitoring
```

**Step 4 — Get the Grafana admin password:**

```bash
kubectl get secret kube-prom-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

The default username is `admin`.

**Step 5 — Port-forward to access Grafana:**

```bash
kubectl port-forward -n monitoring svc/kube-prom-grafana 3000:80
```

Open your browser to [http://localhost:3000](http://localhost:3000) and log in with `admin` and the password from Step 4.

**Step 6 — Explore the pre-built dashboards:**

Navigate to **Dashboards** in the left sidebar. The kube-prometheus-stack comes with dozens of pre-configured dashboards:

- **Kubernetes / Compute Resources / Cluster** — Cluster-wide CPU and memory usage
- **Kubernetes / Compute Resources / Namespace (Pods)** — Per-pod resource usage
- **Kubernetes / Compute Resources / Node (Pods)** — Node-level breakdown
- **Node Exporter / Nodes** — OS-level node metrics (disk, network, CPU)

These dashboards give you the historical visibility that `kubectl top` can't — you can see trends, spikes, and anomalies over time.

Press `Ctrl+C` to stop the port-forward when done.

### Clean Up

```bash
# Delete HPA resources
kubectl delete hpa php-apache --ignore-not-found
kubectl delete deployment php-apache --ignore-not-found
kubectl delete svc php-apache --ignore-not-found
kubectl delete pod load-generator --ignore-not-found

# Delete Prometheus/Grafana (if installed)
helm uninstall kube-prom -n monitoring 2>/dev/null
kubectl delete namespace monitoring --ignore-not-found

# Delete the Kind cluster
kind delete cluster --name scaling-lab
```

---

## Key Takeaways

1. **Always set resource requests and limits.** Pods without resource specifications become BestEffort — first to be evicted under pressure and invisible to the scheduler for capacity planning. Requests guarantee minimum resources; limits enforce maximum usage.

2. **QoS classes determine eviction order.** Guaranteed pods (requests == limits) are evicted last. BestEffort pods (no resources specified) are evicted first. For critical workloads, use Guaranteed QoS.

3. **HPA automates horizontal scaling.** It watches metrics, calculates desired replicas using a simple ratio formula, and adjusts replica count automatically. No manual intervention needed — just define the target utilization and bounds.

4. **HPA requires Metrics Server.** Without Metrics Server, HPA has no data to work with. On Kind, you need to install it separately and add the `--kubelet-insecure-tls` flag.

5. **`kubectl top` is real-time only.** For historical metrics, trending, and alerting, you need Prometheus (or a similar monitoring system). The kube-prometheus-stack Helm chart is the fastest path to production-grade monitoring.

6. **Memory limits are hard limits — OOMKill is immediate.** There's no swap safety net in Kubernetes. If a container exceeds its memory limit, it's terminated instantly. Set memory limits based on actual usage patterns, not guesses.

7. **LimitRange protects namespaces from resource anarchy.** Set default requests, limits, and maximums per namespace so that even pods deployed without resource specifications get sensible defaults. It's the `ulimit` of Kubernetes.

---

## Further Reading

- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Configure Quality of Service for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
- [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [HorizontalPodAutoscaler Walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
- [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)
- [kube-prometheus-stack Helm Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/)

---

**Previous:** [Chapter 11 — Security](11-security.md)
**Next:** [Chapter 13 — Troubleshooting](13-troubleshooting.md)
