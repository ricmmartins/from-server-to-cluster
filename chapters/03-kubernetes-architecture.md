# Chapter 3: Kubernetes Architecture

*"In Linux, you manage a server. In Kubernetes, you manage the intent — and controllers manage the servers for you."*

---

## The Cluster Is a Distributed Operating System

In Chapter 2, you learned that containers are just Linux processes with namespaces and cgroups. Now imagine you need to run hundreds of those containers across dozens of machines, keep them healthy, route traffic between them, and handle failures automatically.

That's the problem Kubernetes solves — and it solves it by borrowing architectural patterns directly from Linux.

Think about how a single Linux server works:

- **systemd** (PID 1) ensures your services are running and restarts them if they crash
- **/etc/** stores configuration files that define how services behave
- **iptables/nftables** manages network routing rules
- The **CPU scheduler** decides which process gets CPU time and on which core
- **D-Bus / systemctl** is the interface you use to communicate with the system

Kubernetes has an equivalent for every one of these — but instead of managing processes on one machine, it manages containers across a fleet of machines. The architecture splits into two parts: the **control plane** (the brain) and **worker nodes** (the muscle).

## Control Plane Components

The control plane runs the decision-making and state-management components. In a production cluster, it typically runs on dedicated machines (or multiple machines for high availability). In Kind, everything runs on a single Docker container.

### kube-apiserver — The Central Command Interface

**Linux analogy: systemctl / D-Bus**

Every interaction with a Kubernetes cluster goes through the API server. When you run `kubectl get pods`, you're making an HTTPS REST call to the API server. When the scheduler needs to assign a Pod to a node, it talks to the API server. When the kubelet reports a node's status, it talks to the API server.

The API server is:
- **The single point of entry** — no component talks directly to another; everything goes through the API
- **An authentication and authorization gateway** — every request is authenticated (certificates, tokens) and authorized (RBAC)
- **An admission controller** — requests can be validated, mutated, or rejected by admission webhooks before being persisted
- **A REST API** — every Kubernetes resource (Pods, Services, Deployments) is a REST endpoint you can query with `curl`

On a Linux server, `systemctl` is your interface to manage services, and D-Bus is the communication backbone between system components. The API server serves both roles for the entire cluster.

### etcd — The Distributed Configuration Store

**Linux analogy: /etc/ directory (but distributed and versioned)**

etcd is a distributed key-value store that holds the entire cluster state: every Pod, every Service, every ConfigMap, every Secret. When the API server receives a request to create a Pod, the Pod definition is persisted to etcd. When a controller needs to know what Deployments exist, it reads from etcd (via the API server — nothing talks to etcd directly except the API server).

Key characteristics:
- **Distributed consensus** using the Raft protocol — data is replicated across multiple etcd nodes (in production, typically 3 or 5)
- **Strongly consistent** — reads always return the latest committed value
- **Versioned** — every key has a revision number; Kubernetes uses this for optimistic concurrency control (the `resourceVersion` field on every object)
- **Watch support** — components can subscribe to changes on specific keys, enabling the reactive controller pattern that makes Kubernetes tick

If `/etc/` on a Linux server is the source of truth for that machine's configuration, etcd is the source of truth for the entire cluster's state.

### kube-scheduler — The Workload Placer

**Linux analogy: Linux CPU scheduler**

When you create a Pod, it initially has no node assigned. The scheduler watches for unscheduled Pods, evaluates which nodes can run them (filtering), scores the remaining candidates (ranking), and assigns the Pod to the best node.

The scheduler considers:
- **Resource availability** — does the node have enough CPU and memory?
- **Affinity and anti-affinity rules** — does the Pod need to be near (or far from) other Pods?
- **Taints and tolerations** — is the node marked as unsuitable for general workloads?
- **Topology constraints** — should Pods be spread across availability zones?

On a Linux server, the CPU scheduler decides which process runs on which core based on priority and fair sharing. The Kubernetes scheduler does the same thing, but at the machine level — deciding which node runs which Pod.

### kube-controller-manager — The Reconciliation Engine

**Linux analogy: systemd service managers (combined)**

The controller manager runs a collection of **controllers** — each is a control loop that watches the cluster state (via the API server) and works to make reality match the desired state.

Key controllers include:
- **Node controller** — monitors node health, marks nodes as NotReady if they stop reporting
- **Replication controller** — ensures the correct number of Pod replicas are running
- **Job controller** — manages batch workloads (run-to-completion)
- **ServiceAccount controller** — creates default ServiceAccounts for new namespaces
- **Endpoint controller** — populates the Endpoints object (which Pods back a Service)

Think of systemd on Linux: it reads unit files (desired state) and ensures the corresponding services are running (actual state). If a service crashes, systemd restarts it. Kubernetes controllers do the same, but at scale — if a Pod dies, the ReplicaSet controller creates a replacement. If a node goes down, the node controller marks it as unhealthy and controllers reschedule the affected Pods elsewhere.

### cloud-controller-manager — The Cloud Integration Layer

**Linux analogy: hardware-specific kernel drivers**

This optional component integrates Kubernetes with cloud provider APIs. It handles:
- **Node management** — detecting when a cloud VM is deleted and removing the corresponding Node object
- **Load balancers** — provisioning cloud load balancers when you create a Service of type `LoadBalancer`
- **Storage** — interacting with cloud storage APIs for volume provisioning

Just as Linux has kernel drivers that abstract hardware differences (you use the same `mount` command regardless of the disk type), the cloud controller manager abstracts cloud differences so Kubernetes works the same everywhere.

In Kind (and on bare-metal clusters), this component doesn't exist — there's no cloud to integrate with.

## Worker Node Components

Worker nodes are where your actual workloads run. Each node runs a small set of components that receive instructions from the control plane and ensure containers are running correctly.

### kubelet — The Node Agent

**Linux analogy: systemd (PID 1) on each machine**

The kubelet is an agent running on every node. Its job is simple and critical: **ensure that the containers described in PodSpecs are running and healthy.**

The kubelet:
- **Receives Pod assignments** from the API server (via watch)
- **Tells the container runtime** (containerd) to pull images and start containers
- **Runs health checks** (liveness probes, readiness probes) and reports status back to the API server
- **Manages Pod lifecycle** — starts, stops, and restarts containers as needed
- **Reports node status** — CPU, memory, disk, and running Pod information

On a Linux server, systemd reads unit files and ensures the right processes are running. The kubelet reads PodSpecs and ensures the right containers are running. The parallel is direct — but the kubelet gets its instructions from the API server, not from local files (mostly — static Pods are the exception).

### kube-proxy — The Network Rules Manager

**Linux analogy: iptables / nftables rule manager**

kube-proxy runs on every node and maintains network rules that enable the Kubernetes Service abstraction. When you create a Service, kube-proxy programs rules so that traffic to the Service's ClusterIP gets forwarded to one of the backing Pods.

kube-proxy can operate in different modes:
- **iptables mode** (common) — programs iptables rules for each Service; kernel-space packet forwarding
- **IPVS mode** — uses the kernel's IP Virtual Server for better performance at scale
- **nftables mode** (newer, alpha from Kubernetes v1.29) — uses nftables as the backend, replacing iptables rules

On a Linux server, you write iptables rules to route traffic. In Kubernetes, kube-proxy writes those rules for you based on the Services you define. The mechanism is identical — the automation layer is what's new.

### Container Runtime — The Container Engine

**Linux analogy: package manager + process launcher**

The container runtime pulls images and starts containers. In modern Kubernetes (v1.24+), the standard runtime is **containerd**, which communicates with the kubelet through the Container Runtime Interface (CRI).

The stack on each node is:

```
kubelet → (CRI) → containerd → runc → Linux kernel
```

This is the same stack from Chapter 2, minus Docker. Kubernetes doesn't need Docker's CLI or daemon — it talks directly to containerd.

## How It All Fits Together

Here's the flow when you run `kubectl run nginx --image=nginx`:

1. **kubectl** sends an HTTPS POST to the **API server** with the Pod specification
2. **API server** authenticates you, checks RBAC authorization, runs admission controllers, and persists the Pod to **etcd** (with status "Pending")
3. **Scheduler** notices the new unassigned Pod (via a watch on the API server), evaluates nodes, and updates the Pod with the selected node name
4. **kubelet** on the chosen node notices it has a new Pod assigned (via its watch), tells **containerd** to pull the nginx image and start the container
5. **kubelet** reports the Pod status as "Running" to the API server, which persists it to etcd
6. **kube-proxy** on each node updates network rules so the Pod is reachable via Services

Every step goes through the API server. The API server is the hub; everything else is a spoke. This is by design — it means there's a single, auditable, authenticated chokepoint for all cluster operations.

## Linux ↔ Kubernetes Architecture Comparison Table

| Linux Component | K8s Equivalent | Role |
|----------------|----------------|------|
| `/etc/` directory | etcd | Configuration and state store (but distributed, versioned, and consensus-backed) |
| systemd (PID 1) | kubelet | Ensures processes/containers are running as declared |
| Init system (boot process) | kube-scheduler | Decides what runs where based on available resources |
| D-Bus / `systemctl` | kube-apiserver | Central command and communication interface |
| iptables / nftables | kube-proxy | Network routing rules management |
| Package manager (apt, yum) | Container runtime (containerd) | Pulls and runs software packages/images |
| Hardware drivers | cloud-controller-manager | Abstracts underlying infrastructure differences |
| systemd unit files | PodSpecs (YAML manifests) | Declarative descriptions of what should run |
| `/var/log/` + journald | API server audit logs + Pod logs | Centralized activity and event logging |

> **Where the Linux Analogy Breaks**
>
> - **etcd is not just `/etc/`.** It's a distributed consensus system using the Raft protocol. Losing a majority of etcd nodes means losing the ability to write cluster state. Losing all etcd data without a backup means losing the cluster entirely. On a Linux server, corrupting `/etc/` is bad but recoverable — you can boot from a rescue disk. etcd failure at scale is a much bigger disaster because it affects every node and workload in the cluster.
>
> - **The API server isn't just `systemctl`.** It's simultaneously an authentication gateway, an authorization engine (RBAC), an admission controller pipeline, a REST API, and the only component that talks to etcd. Nothing happens in the cluster without going through it. On Linux, you can bypass systemd and run processes manually. In Kubernetes, bypassing the API server means the cluster doesn't know the workload exists.
>
> - **The flow of control is reversed.** On a Linux server, you SSH in and run commands — you push instructions to the machine. In Kubernetes, the kubelet **pulls** instructions from the API server. You never SSH into a node to start a container. You submit a manifest to the API server, and the kubelet on the appropriate node picks it up. This pull-based model is what enables Kubernetes to manage thousands of nodes without needing SSH access to any of them.

## Diagnostic Lab: Exploring Kubernetes Architecture

This lab uses a Kind cluster to explore the actual components that make up a Kubernetes cluster. You'll see that every control plane component is a real Linux process running inside the Kind container.

### Prerequisites

- Docker installed and running
- Kind and kubectl installed (see Chapter 1 lab)

### Exercise 1: Create a Kind Cluster

```bash
kind create cluster --name arch-lab
```

Wait for the cluster to be ready:

```bash
kubectl cluster-info --context kind-arch-lab
```

> **Source:** [Kind — Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)

### Exercise 2: Explore Control Plane Components

In Kind, all control plane components run as static Pods in the `kube-system` namespace:

```bash
kubectl get pods -n kube-system
```

You should see something like:

```
NAME                                         READY   STATUS    RESTARTS   AGE
coredns-...                                  1/1     Running   0          2m
coredns-...                                  1/1     Running   0          2m
etcd-arch-lab-control-plane                  1/1     Running   0          2m
kindnet-...                                  1/1     Running   0          2m
kube-apiserver-arch-lab-control-plane        1/1     Running   0          2m
kube-controller-manager-arch-lab-control-plane  1/1  Running   0          2m
kube-proxy-...                               1/1     Running   0          2m
kube-scheduler-arch-lab-control-plane        1/1     Running   0          2m
```

Every component we discussed is right there — running as a container (which is just a Linux process, as you learned in Chapter 2).

### Exercise 3: Check Cluster Health

The modern way to check API server health uses the `/readyz` and `/livez` endpoints:

```bash
# Readiness check (is the API server ready to serve traffic?)
kubectl get --raw='/readyz?verbose'
```

You'll see a detailed list of health checks, each reporting `ok` or a failure reason. This tells you which subsystems are healthy.

```bash
# Liveness check (is the API server alive?)
kubectl get --raw='/livez?verbose'
```

> **Note:** You may see `kubectl get componentstatuses` referenced in older tutorials. This command and its underlying API have been deprecated since Kubernetes v1.19 and may produce incomplete or misleading results. Use the `/readyz` and `/livez` endpoints instead.
>
> **Source:** [Kubernetes API Health Endpoints](https://kubernetes.io/docs/reference/using-api/health-checks/)

### Exercise 4: Inspect kubelet Logs

The kubelet runs as a systemd service inside the Kind container (which is itself a Docker container). Let's look at its logs:

```bash
docker exec arch-lab-control-plane journalctl -u kubelet --no-pager | tail -20
```

You'll see the kubelet starting up, registering the node, and syncing Pod states — exactly like reading `journalctl -u nginx` on a regular Linux server.

For a broader view of kubelet activity:

```bash
docker exec arch-lab-control-plane journalctl -u kubelet --no-pager | grep -i "started" | tail -10
```

### Exercise 5: Peek Inside etcd

All cluster state lives in etcd. Let's prove it by querying etcd directly from inside the etcd Pod:

```bash
# List some keys stored in etcd — these ARE your cluster objects
kubectl exec -n kube-system etcd-arch-lab-control-plane -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get / --prefix --keys-only --limit=20
```

You'll see keys like:
```
/registry/configmaps/kube-system/coredns
/registry/namespaces/default
/registry/pods/kube-system/etcd-arch-lab-control-plane
/registry/services/specs/default/kubernetes
```

Every Kubernetes object — Pods, Services, ConfigMaps, Secrets — is stored as a key in etcd under the `/registry/` prefix. This is the cluster's source of truth.

> **Source:** [etcd documentation](https://etcd.io/docs/) | [Kubernetes — Operating etcd clusters](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)

### Exercise 6: Trace the API Calls

When you run a kubectl command, it's making REST API calls to the API server. Let's see them:

```bash
kubectl get pods -n kube-system -v=6
```

At verbosity level 6, you'll see output like:

```
I0101 12:00:00.000000  loader.go:395] Config loaded from file: /home/user/.kube/config
I0101 12:00:00.100000  round_trippers.go:553] GET https://127.0.0.1:PORT/api/v1/namespaces/kube-system/pods?limit=500 200 OK in 15 milliseconds
```

This reveals the actual HTTP request: `GET /api/v1/namespaces/kube-system/pods`. Kubernetes is just a REST API — `kubectl` is a fancy HTTP client. You could make the same request with `curl`:

```bash
# Get the API server URL and use kubectl's credentials to make a raw request
kubectl get --raw /api/v1/namespaces/kube-system/pods | python3 -m json.tool | head -20
```

Understanding that Kubernetes is a REST API makes debugging much more intuitive — you're just dealing with HTTP endpoints that return JSON.

> **Source:** [Kubernetes API Overview](https://kubernetes.io/docs/concepts/overview/kubernetes-api/) | [kubectl Verbosity and Debugging](https://kubernetes.io/docs/reference/kubectl/quick-reference/#kubectl-output-verbosity-and-debugging)

### Clean Up

```bash
kind delete cluster --name arch-lab
```

## Key Takeaways

1. **Kubernetes architecture splits into control plane (brain) and worker nodes (muscle).** The control plane makes decisions; worker nodes execute them.
2. **The API server is the hub of everything.** All communication between components goes through it — there's no direct component-to-component chatter. This makes the system auditable and secure.
3. **etcd stores all cluster state** as a distributed, strongly consistent key-value store. Losing etcd data means losing the cluster. Back it up.
4. **The scheduler assigns Pods to nodes** based on resource availability, constraints, and policies — just like the Linux CPU scheduler assigns processes to cores.
5. **The kubelet is systemd for containers.** It runs on every node, receives PodSpecs from the API server, and ensures the right containers are running and healthy.
6. **kube-proxy manages network rules** (iptables, IPVS, or nftables) to implement the Service abstraction — same kernel features you already know, automated at cluster scale.
7. **Everything is a REST API call.** When you run `kubectl get pods`, you're making an HTTP GET request. Understanding this makes debugging transparent — add `-v=6` to any kubectl command to see the actual API calls.

## Further Reading

- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)
- [API Health Endpoints](https://kubernetes.io/docs/reference/using-api/health-checks/)
- [Operating etcd Clusters](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
- [Kind — Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [Kubernetes API Concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/)

---

**Previous:** [Chapter 2 — Containers Demystified](02-containers-demystified.md)

**Next:** [Chapter 4 — How Kubernetes Thinks](04-how-kubernetes-thinks.md) — Desired state, reconciliation loops, controllers, labels, selectors — the mental model that makes everything click.
