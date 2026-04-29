# Chapter 4: How Kubernetes Thinks

*"You don't manage servers anymore. You describe intentions, and controllers make them real."*

---

## The Biggest Mental Shift You'll Make

If there's one chapter in this book you read twice, make it this one.

On a Linux server, you're used to being in control. You type a command, something happens, and you see the result. Install nginx? `apt install nginx`. Start it? `systemctl start nginx`. It crashes? Set `Restart=always` in the systemd unit. You are the operator, and the machine does what you say, when you say it.

Kubernetes flips this on its head. You don't *do* things — you *declare* things. You write a document (a YAML manifest) that says "I want three nginx replicas running, each with 256MB of memory, exposed on port 80." Then you hand that document to the API server and walk away. Kubernetes takes it from there.

This isn't just a syntactic difference. It's a fundamentally different relationship between you and the system. On Linux, you're the hands-on mechanic under the hood. In Kubernetes, you're the architect who draws the blueprint and trusts the construction crew (controllers) to build it — and to *keep* it built, forever.

Let's break down exactly how Kubernetes thinks.

---

## Desired State vs. Actual State

Every object in Kubernetes has two states:

- **Desired state** — what you asked for (defined in your YAML manifest, stored in etcd)
- **Actual state** — what's really happening right now on the cluster

The entire purpose of Kubernetes is to continuously make the actual state match the desired state. This is the single most important concept in the entire system.

### A Linux Comparison

Think about a systemd unit with `Restart=always`:

```ini
[Service]
ExecStart=/usr/bin/nginx
Restart=always
RestartSec=5
```

If nginx crashes, systemd notices and restarts it. The "desired state" is "nginx should be running." The "actual state" might temporarily be "nginx is dead." Systemd reconciles the difference.

Kubernetes does this for *everything*, not just process restarts:

- Want 3 replicas? A controller ensures there are always exactly 3.
- Want a Service routing to healthy pods? A controller updates endpoints when pods come and go.
- Want a volume mounted? A controller attaches it to the right node.
- Want a node to be drained? A controller evicts pods and respects disruption budgets.

### Declaring Desired State

In practice, desired state looks like this:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-frontend
  template:
    metadata:
      labels:
        app: web-frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
```

You apply this with `kubectl apply -f deployment.yaml`, and you're done. You don't tell Kubernetes *which* node to run it on, *how* to pull the image, or *what* to do if a pod crashes. You just say what you want. The system handles the rest.

---

## The Reconciliation Loop

Every controller in Kubernetes runs the same basic algorithm, forever:

```
1. WATCH — Observe the current state of the objects you're responsible for
2. DIFF  — Compare current state to desired state
3. ACT   — Take action to close the gap
4. REPEAT
```

This is called the **reconciliation loop** (or control loop), and it's the heartbeat of the system.

### The Thermostat Analogy

The Kubernetes documentation itself uses this analogy: think of a thermostat. You set it to 22°C (desired state). The thermostat continuously reads the room temperature (actual state). If it's too cold, it turns on the heater. If it's too warm, it turns on the AC. It never stops checking.

Every Kubernetes controller is a thermostat for its resource type.

### What Happens When You Create a Deployment

Let's trace what happens when you `kubectl apply` the deployment above:

1. `kubectl` sends the manifest to the **API server**
2. The API server validates it and stores it in **etcd**
3. The **Deployment controller** notices a new Deployment object
4. It creates a **ReplicaSet** to manage the desired 3 replicas
5. The **ReplicaSet controller** notices the new ReplicaSet
6. It creates 3 **Pod** objects (desired state: these pods should exist)
7. The **Scheduler** notices 3 unscheduled Pods
8. It assigns each Pod to a node based on available resources
9. The **kubelet** on each selected node notices new Pods assigned to it
10. Each kubelet pulls the container image and starts the container

That's at least five different controllers, each doing one small job, chained together by the reconciliation pattern. None of them know about the others. They just watch for changes to the objects they care about and react.

### What Happens When a Pod Dies

Now suppose a node crashes and takes one of the three pods with it:

1. The **Node controller** detects the node is unresponsive (via missed heartbeats)
2. It marks the node as `NotReady`
3. The **ReplicaSet controller** notices there are now only 2 running pods, but the desired count is 3
4. It creates a new Pod object
5. The **Scheduler** places it on a healthy node
6. The **kubelet** starts it

You didn't do anything. You didn't get paged. The system healed itself because every controller kept checking: "Does reality match the spec?"

---

## Controllers and the Controller Pattern

A **controller** is a process that watches the state of your cluster through the API server, then makes changes attempting to move the current state towards the desired state.

### Built-in Controllers

Kubernetes ships with dozens of controllers, all running inside the `kube-controller-manager` process. Some key ones:

| Controller | Watches | Ensures |
|------------|---------|---------|
| Deployment controller | Deployments | Correct ReplicaSets exist for each Deployment |
| ReplicaSet controller | ReplicaSets | The right number of Pods are running |
| Node controller | Nodes | Unhealthy nodes are detected and handled |
| Job controller | Jobs | Pods run to completion for batch workloads |
| EndpointSlice controller | Services, Pods | Service endpoints point to healthy Pod IPs |
| ServiceAccount controller | Namespaces | Default ServiceAccount exists in each namespace |

### The Single-Responsibility Principle

Notice how each controller does *one thing*. The Deployment controller doesn't create Pods directly — it creates ReplicaSets, and then the ReplicaSet controller creates Pods. This chain of responsibility is deliberate. It means:

- Each controller is simple and testable
- You can swap out or extend individual pieces
- The system is resilient — if one controller is slow, others keep working

This is the same design principle you see in Unix philosophy: do one thing well. `grep` finds text, `sort` orders it, `uniq` deduplicates it. Each tool is simple; the power comes from composition.

---

## Labels, Selectors, and Annotations

If controllers are the muscles of Kubernetes, labels and selectors are the nervous system. They're how objects find each other.

### Labels

Labels are key-value pairs attached to any Kubernetes object. They're how you organize, categorize, and select resources.

```yaml
metadata:
  labels:
    app: web-frontend
    env: production
    team: platform
    version: v2.1.0
```

Labels are *freeform* — you define whatever keys and values make sense for your organization. Kubernetes doesn't enforce any schema beyond basic syntax rules (63 characters max for the name segment, alphanumeric characters, dashes, underscores, and dots).

### Selectors

Selectors are queries against labels. They're how controllers, Services, and you (via kubectl) find the objects they care about.

**Equality-based selectors:**

```bash
# Find all pods in production
kubectl get pods -l env=production

# Find all pods NOT in production
kubectl get pods -l env!=production
```

**Set-based selectors:**

```bash
# Find pods in either staging or production
kubectl get pods -l 'env in (staging, production)'

# Find pods with a "team" label (any value)
kubectl get pods -l team

# Find pods without a "deprecated" label
kubectl get pods -l '!deprecated'
```

### How Services Find Pods

This is where labels become essential. A Service doesn't know about specific Pods — it uses a selector to find them dynamically:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-frontend
spec:
  selector:
    app: web-frontend    # "find all Pods with this label"
  ports:
  - port: 80
    targetPort: 80
```

When a new Pod with `app: web-frontend` appears, the Service automatically includes it. When a Pod with that label is deleted, it's automatically removed. No manual registration. No configuration reload.

Think of it like this: on Linux, you might configure a load balancer with a static list of backend IPs. In Kubernetes, the load balancer (Service) says "send traffic to anything labeled `app: web-frontend`," and the system keeps the list updated in real time.

### Annotations

Annotations are also key-value metadata, but with a different purpose. While labels are for *identification and selection*, annotations are for *attaching arbitrary non-identifying metadata*.

```yaml
metadata:
  annotations:
    description: "Main customer-facing frontend"
    git-commit: "a1b2c3d4"
    config.kubernetes.io/managed-by: "kustomize"
```

Key differences from labels:

- Annotations **cannot** be used in selectors
- Annotation values can be much larger (up to 256KB total per object)
- They're typically used by tools, controllers, and humans for informational purposes
- Common uses: build info, git commit hashes, tool configuration, change reasons

Think of labels as the filing system you use to find documents, and annotations as the sticky notes on the documents with extra context.

---

## Namespaces

Kubernetes namespaces provide a way to divide cluster resources into logically isolated groups. They're one of the first things that trips up Linux professionals, because the name collides with a very different Linux concept.

### Not Linux Kernel Namespaces

Let's be crystal clear: **Kubernetes namespaces are NOT the same as Linux kernel namespaces.**

| Aspect | Linux Kernel Namespaces | Kubernetes Namespaces |
|--------|------------------------|-----------------------|
| Purpose | Process-level isolation (PID, network, mount, etc.) | Logical grouping and access control |
| Scope | Single machine | Entire cluster |
| Isolation | Hard isolation (processes can't see each other) | Soft isolation (RBAC, resource quotas, network policies) |
| Mechanism | Kernel-level | API server-level |

Linux kernel namespaces make containers possible — they isolate process trees, network stacks, and filesystems at the kernel level. Kubernetes namespaces are a *logical partitioning* mechanism — more like directories in a filesystem than kernel isolation boundaries.

### What Namespaces Actually Do

Namespaces provide:

1. **Name scoping** — You can have a Pod named `nginx` in the `staging` namespace and another Pod named `nginx` in the `production` namespace. No conflict.
2. **Access control boundaries** — RBAC policies can grant permissions per namespace. "Alice can deploy to staging but not production."
3. **Resource quotas** — You can limit how much CPU, memory, and how many objects a namespace can consume.
4. **Default settings** — LimitRanges can set default resource requests/limits for all Pods in a namespace.

### Default Namespaces

Every cluster starts with four namespaces:

| Namespace | Purpose |
|-----------|---------|
| `default` | Where your objects go if you don't specify a namespace |
| `kube-system` | Kubernetes system components (API server, scheduler, CoreDNS, etc.) |
| `kube-public` | Publicly readable resources (rarely used directly) |
| `kube-node-lease` | Node heartbeat leases for failure detection |

### Working with Namespaces

```bash
# List all namespaces
kubectl get namespaces

# Create a namespace
kubectl create namespace staging

# Deploy to a specific namespace
kubectl apply -f deployment.yaml -n staging

# List pods in a specific namespace
kubectl get pods -n staging

# List pods in ALL namespaces
kubectl get pods -A

# Set your default namespace (so you don't have to type -n every time)
kubectl config set-context --current --namespace=staging
```

### When to Use Namespaces

Good use cases:

- **Per-environment**: `staging`, `production`, `dev`
- **Per-team**: `team-platform`, `team-payments`
- **Per-application**: `app-frontend`, `app-backend` (in larger organizations)

You do *not* need namespaces for every small difference. Different versions of the same app? Use labels, not namespaces. The official documentation is explicit: "For clusters with a few to tens of users, you should not need to create or think about namespaces at all."

---

## Scheduling Fundamentals

The **kube-scheduler** is the component that decides which node a Pod runs on. Every time a new Pod is created without a node assignment, the scheduler picks one.

On Linux, you might use `taskset` to pin a process to specific CPUs, or `nice` to adjust priority. Kubernetes scheduling is the cluster-level equivalent — but instead of CPUs on one machine, you're choosing among nodes in a cluster.

### How the Scheduler Decides

The scheduler works in two phases:

1. **Filtering** — Eliminate nodes that can't run the Pod (not enough resources, wrong labels, taints that aren't tolerated)
2. **Scoring** — Rank the remaining nodes and pick the best one (most resources available, best spread, etc.)

### Resource Requests and Limits

When you define a Pod, you can specify how much CPU and memory it needs:

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "250m"      # 250 millicores = 0.25 CPU
  limits:
    memory: "256Mi"
    cpu: "500m"
```

- **Requests** are what the scheduler uses for placement. "This Pod needs at least 128Mi of memory and 250m of CPU." The scheduler only places the Pod on a node that has enough *unrequested* resources.
- **Limits** are the maximum the Pod is allowed to consume. If a container exceeds its memory limit, it gets OOM-killed. If it exceeds its CPU limit, it gets throttled.

This maps directly to Linux cgroups. Under the hood, the kubelet translates requests and limits into cgroup settings on the node. Requests become cgroup reservations; limits become cgroup hard caps.

### Node Affinity

Node affinity lets you constrain which nodes a Pod can be scheduled on, based on node labels.

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
```

This says: "Only schedule this Pod on nodes labeled `disktype=ssd`." It's the Kubernetes equivalent of Linux's `taskset`, but instead of pinning to CPUs, you're pinning to nodes with specific characteristics.

There's also `preferredDuringSchedulingIgnoredDuringExecution`, which is a soft preference rather than a hard requirement — the scheduler will *try* to honor it but won't fail if it can't.

### Taints and Tolerations

Taints are the *opposite* of affinity — they let a node *repel* Pods.

```bash
# Taint a node: "this node is for GPU workloads only"
kubectl taint nodes gpu-node-1 workload=gpu:NoSchedule
```

Now, only Pods with a matching toleration can be scheduled there:

```yaml
spec:
  tolerations:
  - key: "workload"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
```

Think of it like a VIP section at a venue. The taint is the velvet rope (keeps everyone out by default), and the toleration is the VIP pass (lets specific Pods through).

Taint effects include:

| Effect | Behavior |
|--------|----------|
| `NoSchedule` | New Pods without a matching toleration won't be scheduled on this node |
| `PreferNoSchedule` | Scheduler will *try* to avoid this node, but it's not guaranteed |
| `NoExecute` | Existing Pods without a matching toleration are evicted; new ones aren't scheduled |

### Topology Spread Constraints

Topology spread constraints ensure your Pods are distributed evenly across failure domains (nodes, zones, regions):

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web-frontend
```

This says: "The difference in the number of `web-frontend` Pods between any two zones should be at most 1." If zone A has 3 Pods and zone B has 2, the next Pod goes to zone B.

There's no direct Linux equivalent for this — it's a cluster-wide concern that doesn't exist on a single machine.

---

## Linux ↔ K8s Comparison Table

| Linux Concept | K8s Equivalent | Key Difference |
|---------------|----------------|----------------|
| `systemd Restart=always` | Controller reconciliation loop | K8s controllers manage across nodes, not just locally on one machine |
| `/etc/systemd/system/*.service` | YAML manifests | Declarative desired state stored in etcd, versioned and auditable |
| File labels (SELinux contexts) | Labels + Selectors | Labels are freeform key-value pairs; selectors query them dynamically |
| Linux kernel namespaces | K8s Namespaces | K8s namespaces are logical/RBAC boundaries, not kernel-level process isolation |
| `nice`/`renice`, cgroups | Resource requests/limits | Scheduler uses requests for node placement decisions; kubelet enforces limits via cgroups |
| `/etc/hosts`, config file comments | Annotations | Metadata attached to objects; structured key-value but not queryable for selection |
| CPU affinity (`taskset`) | Node affinity / nodeSelector | Constrains where workloads run based on node labels, not CPU IDs |

---

> **Where the Linux Analogy Breaks**
>
> **Eventual consistency, not immediate execution.** Linux is immediate: you run a command, it happens now. Kubernetes is eventually consistent — you submit a desired state, and controllers work toward it asynchronously. You might wait seconds or minutes for the system to converge. This isn't a bug; it's the design. Distributed systems *must* be eventually consistent.
>
> **The manifest is the source of truth, not the running state.** There's no "SSH in and fix it" escape hatch for normal operations. If you manually modify a running Pod (say, installing a package via `kubectl exec`), the controller will overwrite your changes the next time it recreates the Pod. The YAML manifest in version control is the only source of truth. If it's not in the manifest, it doesn't exist.
>
> **Labels are flat, not hierarchical.** Labels aren't organized like a filesystem hierarchy — they're flat key-value pairs. The power comes from flexible selector queries (equality-based, set-based), not from nested directory structures. You can't do `labels/env/staging` — you do `env=staging` and query with `-l env=staging`.

---

## Diagnostic Lab: Reconciliation, Labels, Namespaces, and Scheduling

This lab demonstrates the core concepts from this chapter using a Kind cluster.

### Prerequisites

Make sure you have a Kind cluster running:

```bash
kind create cluster --name chapter04
```

Verify it's ready:

```bash
kubectl cluster-info --context kind-chapter04
```

---

### Part 1: Watch the Reconciliation Loop in Action

**Step 1 — Create a Deployment with 3 replicas:**

```bash
kubectl create deployment nginx --image=nginx:1.27 --replicas=3
```

**Step 2 — Verify the Pods are running:**

```bash
kubectl get pods -l app=nginx
```

You should see 3 Pods in `Running` state.

**Step 3 — Delete a Pod and watch it come back:**

```bash
# Pick one of the Pod names from the output above
kubectl delete pod $(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
```

Immediately check:

```bash
kubectl get pods -l app=nginx
```

You'll see a new Pod with a different name being created (or already running). The ReplicaSet controller noticed the count dropped to 2 and created a replacement. This is the reconciliation loop in action.

**Step 4 — Scale to 5 replicas:**

```bash
kubectl scale deployment nginx --replicas=5
```

Watch the new Pods appear:

```bash
kubectl get pods -l app=nginx -w
```

Press `Ctrl+C` once all 5 are Running.

**Step 5 — View the events that tell the reconciliation story:**

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

You'll see events from the Deployment controller (scaling), the ReplicaSet controller (creating Pods), the Scheduler (assigning Pods to nodes), and the kubelet (pulling images, starting containers).

---

### Part 2: Labels and Selectors

**Step 1 — Inspect existing labels:**

```bash
kubectl get pods --show-labels
```

Every Pod created by the Deployment already has `app=nginx` — that's how the ReplicaSet finds its Pods.

**Step 2 — Add custom labels:**

```bash
# Label two pods as staging
POD1=$(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
POD2=$(kubectl get pods -l app=nginx -o jsonpath='{.items[1].metadata.name}')
kubectl label pod $POD1 env=staging
kubectl label pod $POD2 env=staging

# Label the rest as production
kubectl label pod $(kubectl get pods -l app=nginx -o jsonpath='{.items[2].metadata.name}') env=production
kubectl label pod $(kubectl get pods -l app=nginx -o jsonpath='{.items[3].metadata.name}') env=production
kubectl label pod $(kubectl get pods -l app=nginx -o jsonpath='{.items[4].metadata.name}') env=production
```

**Step 3 — Query by label:**

```bash
# All staging pods
kubectl get pods -l env=staging

# All production pods
kubectl get pods -l env=production

# All pods with an "env" label (any value)
kubectl get pods -l env

# Pods that are BOTH nginx AND production
kubectl get pods -l app=nginx,env=production
```

**Step 4 — See how a Service uses selectors:**

Create a Service that selects only production Pods:

```bash
kubectl expose deployment nginx --port=80 --name=nginx-prod --type=ClusterIP --overrides='{"spec":{"selector":{"app":"nginx","env":"production"}}}'
```

Check which Pods the Service selected:

```bash
kubectl describe svc nginx-prod | grep Endpoints
```

You should see only the IPs of the 3 production-labeled Pods.

Clean up the Service:

```bash
kubectl delete svc nginx-prod
```

---

### Part 3: Namespace Isolation

**Step 1 — Create two namespaces:**

```bash
kubectl create namespace team-alpha
kubectl create namespace team-beta
```

**Step 2 — Deploy the same app in both:**

```bash
kubectl create deployment web --image=nginx:1.27 -n team-alpha
kubectl create deployment web --image=nginx:1.27 -n team-beta
```

**Step 3 — Show they're independent:**

```bash
# Each namespace has its own "web" deployment
kubectl get deployments -n team-alpha
kubectl get deployments -n team-beta

# Pods have different names and IPs
kubectl get pods -o wide -n team-alpha
kubectl get pods -o wide -n team-beta
```

**Step 4 — Demonstrate name scoping:**

```bash
# This works — "web" exists in both namespaces independently
kubectl get deployment web -n team-alpha
kubectl get deployment web -n team-beta

# But in the default namespace, there's no "web"
kubectl get deployment web 2>&1 || true
```

**Step 5 — View all namespaces at once:**

```bash
kubectl get pods -A | grep web
```

Clean up:

```bash
kubectl delete namespace team-alpha
kubectl delete namespace team-beta
```

---

### Part 4: Scheduling Decisions

**Step 1 — Describe a pod to see scheduler events:**

```bash
kubectl describe pod $(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
```

Look at the `Events` section at the bottom. You'll see entries like:

```
Successfully assigned default/nginx-xxx to chapter04-control-plane
```

This tells you the Scheduler made a placement decision and assigned the Pod to a node.

**Step 2 — Create a Pod with an unsatisfiable nodeSelector:**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: picky-pod
spec:
  nodeSelector:
    disktype: nvme
  containers:
  - name: nginx
    image: nginx:1.27
EOF
```

**Step 3 — Watch the Pod stay Pending:**

```bash
kubectl get pod picky-pod
```

You'll see `STATUS: Pending`. The scheduler can't find a node with `disktype=nvme`.

Check the events:

```bash
kubectl describe pod picky-pod | tail -5
```

You'll see a warning like:

```
0/1 nodes are available: 1 node(s) didn't match Pod's node affinity/selector.
```

**Step 4 — Fix it by labeling the node:**

```bash
# Label the node
kubectl label node chapter04-control-plane disktype=nvme

# Watch the Pod get scheduled
kubectl get pod picky-pod -w
```

The Pod should transition from `Pending` to `ContainerCreating` to `Running`.

Press `Ctrl+C` once it's Running.

---

### Clean Up

```bash
kubectl delete pod picky-pod
kubectl delete deployment nginx
kind delete cluster --name chapter04
```

---

## Key Takeaways

1. **Kubernetes is a desired-state system.** You declare what you want in a manifest, and controllers continuously reconcile reality to match. You don't tell it *how* — you tell it *what*.

2. **The reconciliation loop (Watch → Diff → Act) is the core algorithm.** Every controller runs this loop forever. When you internalize this pattern, all of Kubernetes makes sense.

3. **Controllers follow the single-responsibility principle.** Each controller manages one resource type. Complex behaviors emerge from chains of simple controllers reacting to each other's changes.

4. **Labels and selectors are the glue connecting objects.** Services find Pods via selectors. ReplicaSets track their Pods via selectors. Your queries use selectors. Master labels, and you master Kubernetes organization.

5. **Kubernetes namespaces are NOT Linux kernel namespaces.** They're logical partitions for organization, access control, and resource quotas — not kernel-level process isolation.

6. **The scheduler uses resource requests for placement decisions.** Always set resource requests on your Pods. Without them, the scheduler is flying blind and you'll hit resource contention issues.

7. **Taints repel Pods; tolerations are the exception pass.** Use taints and tolerations to dedicate nodes for specific workloads (GPU nodes, high-memory nodes) while keeping everything else away.

---

## Further Reading

- [Kubernetes Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)
- [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)
- [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Scheduling, Preemption and Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/)
- [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)

---

**Previous:** [Chapter 3 — Kubernetes Architecture](03-kubernetes-architecture.md)
**Next:** [Chapter 5 — Your First Cluster](05-your-first-cluster.md)
