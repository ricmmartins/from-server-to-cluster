# Chapter 5: Your First Cluster

*"The best way to learn a new system is to break things in it on purpose."*

---

## From Theory to Terminal

You've spent four chapters building a mental model — what containers are, how Kubernetes is architected, and how it thinks in terms of desired state and reconciliation. Now it's time to get your hands dirty.

This chapter is your "get comfortable" chapter. If the previous chapters were studying the map, this is the hike. We'll build a multi-node Kind cluster, learn to speak fluent kubectl, and explore every corner of a running cluster the way a Linux admin explores a new server — poking at processes, checking logs, looking at network connections, and testing what happens when things break.

On a Linux server, the first thing you probably do after SSH'ing in is run `w`, `top`, `df -h`, and maybe `systemctl list-units`. We're going to do the Kubernetes equivalent of all that, and by the end of this chapter, `kubectl` should feel as natural as `systemctl`.

---

## Kind in Depth

In Chapter 1, you created a single-node Kind cluster. That was fine for a "hello world," but real Kubernetes clusters have multiple nodes — a control plane and one or more workers. Kind lets you simulate this on your laptop.

### Multi-Node Cluster Configuration

Kind uses a YAML configuration file to define your cluster topology:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

This gives you a 3-node cluster: one control plane and two workers. Each "node" is actually a Docker container running a full Kubernetes node (kubelet, kube-proxy, container runtime).

Save this as `kind-config.yaml` and create the cluster:

```bash
kind create cluster --name first-cluster --config kind-config.yaml
```

### Port Mappings

If you want to access Services from your host machine (say, to hit an nginx server in the cluster from your browser), you need to map ports from the Docker container to your host:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30080
    hostPort: 8080
    protocol: TCP
- role: worker
- role: worker
```

This maps port 30080 inside the cluster (commonly used with NodePort Services) to port 8080 on your local machine. We'll use this in later chapters.

### What Kind Creates Under the Hood

After creating the cluster, peek at what Docker is actually running:

```bash
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
```

You'll see three containers — these are your "nodes." Each one runs a full kubelet, kube-proxy, and containerd runtime. The control plane node additionally runs etcd, kube-apiserver, kube-scheduler, and kube-controller-manager.

---

## Kubeconfig Explained

When you created the Kind cluster, it configured kubectl to talk to it. But *how*? The answer is the kubeconfig file, which lives at `~/.kube/config` by default.

### The SSH Config Analogy

If you've ever used `~/.ssh/config`, kubeconfig will feel familiar:

| SSH Config Concept | Kubeconfig Equivalent |
|--------------------|----------------------|
| Host entry | Cluster entry |
| Hostname + Port | Server URL |
| IdentityFile (private key) | Client certificate / token |
| User | User entry |
| A full Host block | Context (cluster + user + namespace) |

### Kubeconfig Structure

A kubeconfig file has three main sections:

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: https://127.0.0.1:PORT
    certificate-authority-data: <base64-encoded CA cert>
  name: kind-first-cluster
users:
- name: kind-first-cluster
  user:
    client-certificate-data: <base64-encoded client cert>
    client-key-data: <base64-encoded client key>
contexts:
- context:
    cluster: kind-first-cluster
    user: kind-first-cluster
  name: kind-first-cluster
current-context: kind-first-cluster
```

- **Clusters** — where to connect (server URL + CA cert to trust)
- **Users** — how to authenticate (certs, tokens, or auth plugins)
- **Contexts** — a shorthand that binds a cluster + user + optional default namespace

### Working with Contexts

If you manage multiple clusters (dev, staging, production — or multiple Kind clusters for testing), contexts let you switch between them:

```bash
# See all available contexts
kubectl config get-contexts

# See which context is currently active
kubectl config current-context

# Switch to a different context
kubectl config use-context kind-first-cluster

# View the full kubeconfig (redacting sensitive data)
kubectl config view
```

This is exactly like SSH aliases. Instead of typing `ssh -i ~/.ssh/prod-key -p 2222 admin@10.0.1.5`, you define a Host block and just type `ssh prod`. Instead of passing certificates and URLs to kubectl, you define a context and just type `kubectl --context prod get pods`.

---

## kubectl: Your New CLI

`kubectl` is the Swiss army knife for Kubernetes — your `systemctl`, `journalctl`, `ssh`, `top`, and `ss` all rolled into one.

### The Verb-Resource Pattern

Almost every kubectl command follows this pattern:

```
kubectl <verb> <resource-type> [name] [flags]
```

The most common verbs:

| Verb | What It Does | Linux Equivalent |
|------|-------------|------------------|
| `get` | List resources | `ls`, `systemctl list-units` |
| `describe` | Show detailed info about a resource | `systemctl status`, `ip addr show` |
| `apply` | Create or update from a file (declarative) | `ansible-playbook site.yml` |
| `create` | Create a resource (imperative) | `systemctl start`, `useradd` |
| `delete` | Remove a resource | `systemctl stop`, `rm` |
| `exec` | Run a command inside a container | `ssh`, `nsenter` |
| `logs` | View container output | `journalctl -u service` |
| `port-forward` | Forward a local port to a Pod/Service | `ssh -L` tunnel |
| `edit` | Open a resource in your editor | `systemctl edit` |
| `scale` | Change replica count | — (no direct equivalent) |
| `top` | Show resource usage | `top`, `htop` |

### Common Resource Types

| Resource | Short Name | What It Is |
|----------|-----------|------------|
| `pods` | `po` | The smallest deployable unit (one or more containers) |
| `deployments` | `deploy` | Manages ReplicaSets and rolling updates |
| `services` | `svc` | Stable network endpoint for a set of Pods |
| `configmaps` | `cm` | Configuration data as key-value pairs |
| `secrets` | — | Sensitive data (base64-encoded, not encrypted by default) |
| `namespaces` | `ns` | Logical cluster partitions |
| `nodes` | `no` | The machines (physical or virtual) in the cluster |
| `replicasets` | `rs` | Ensures a specified number of pod replicas |
| `events` | `ev` | Cluster events (scheduling, errors, warnings) |

Short names save typing: `kubectl get po` is the same as `kubectl get pods`.

### Output Formats

kubectl can output in multiple formats — this is incredibly powerful for scripting and debugging:

```bash
# Default table view
kubectl get pods

# Wide view — more columns (node name, IP, etc.)
kubectl get pods -o wide

# Full YAML — the complete object as stored in etcd
kubectl get pod <name> -o yaml

# Full JSON
kubectl get pod <name> -o json

# JSONPath — extract specific fields
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Custom columns — build your own table
kubectl get pods -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName,STATUS:.status.phase'

# Sort by a field
kubectl get pods --sort-by=.status.startTime

# Get just the names (useful for scripting)
kubectl get pods -o name
```

The `-o yaml` output is particularly valuable. When you need to create a YAML manifest for something you built imperatively, just export it:

```bash
kubectl get deployment nginx -o yaml > nginx-deployment.yaml
```

---

## Imperative vs. Declarative

There are two ways to work with Kubernetes: imperative and declarative. Understanding both — and knowing when to use each — is essential.

### Imperative: Run Commands Directly

```bash
# Create a deployment
kubectl create deployment nginx --image=nginx:1.27 --replicas=3

# Expose it as a service
kubectl expose deployment nginx --port=80 --type=NodePort

# Scale it
kubectl scale deployment nginx --replicas=5

# Delete it
kubectl delete deployment nginx
```

This feels natural to Linux admins — it's like running `systemctl start`, `useradd`, or `iptables -A`. You tell the system what to do, step by step.

**Good for:** Quick experiments, debugging, one-off tasks during labs.

### Declarative: Apply YAML Manifests

```bash
# Apply a manifest (creates or updates)
kubectl apply -f nginx-deployment.yaml

# Apply all manifests in a directory
kubectl apply -f ./manifests/

# Delete everything defined in a manifest
kubectl delete -f nginx-deployment.yaml
```

This is the Kubernetes-native way. You define the desired state in a file, store it in version control, and `apply` it. If you change the file and apply again, Kubernetes figures out the diff and makes the minimal changes needed.

**Good for:** Production, anything you want to reproduce, team collaboration, GitOps.

### Dry-Run and Diff

Before applying changes to a live cluster, you can preview them:

```bash
# Dry-run: show what WOULD be sent to the server (without actually sending it)
kubectl apply -f nginx-deployment.yaml --dry-run=client -o yaml

# Server-side dry-run: validate against the API server without persisting
kubectl apply -f nginx-deployment.yaml --dry-run=server

# Diff: compare a local manifest against what's running in the cluster
kubectl diff -f nginx-deployment.yaml
```

The `--dry-run=client` flag is also great for *generating* YAML manifests:

```bash
# Generate a Deployment YAML without creating it
kubectl create deployment nginx --image=nginx:1.27 --replicas=3 --dry-run=client -o yaml > nginx-deployment.yaml
```

This is a common workflow: create imperatively with dry-run to get the YAML template, then customize it and apply declaratively.

---

## Linux ↔ K8s Comparison Table

| Linux Task | kubectl Equivalent | Notes |
|------------|-------------------|-------|
| `ssh user@server` | `kubectl exec -it <pod> -- /bin/sh` | Not a persistent session — Pod may be replaced anytime |
| `systemctl status nginx` | `kubectl get pods -l app=nginx` | Shows status of all Pods matching the label |
| `journalctl -u nginx -f` | `kubectl logs -f deployment/nginx` | Follows stdout/stderr from the container |
| `scp file server:/path` | `kubectl cp file <pod>:/path` | Copies between local filesystem and container |
| `ss -tlnp` | `kubectl get svc` | Lists Services and their port mappings |
| `cat /etc/systemd/system/nginx.service` | `kubectl get deployment nginx -o yaml` | Shows the full desired-state spec |
| `top` | `kubectl top pods` | Requires Metrics Server (installed in lab below) |
| `systemctl restart nginx` | `kubectl rollout restart deployment/nginx` | Triggers a rolling restart of all Pods |
| `systemctl list-units` | `kubectl get all` | Lists common resource types in a namespace |
| `hostnamectl` | `kubectl get nodes -o wide` | Shows node names, IPs, OS, and kernel version |

---

> **Where the Linux Analogy Breaks**
>
> **`kubectl exec` is not SSH.** It looks like SSH, but the Pod might be replaced at any time by a controller. Never make production changes via exec — change the manifest instead. Anything you do inside a container via exec is *ephemeral* and will be lost when the Pod is recreated.
>
> **`kubectl logs` shows container stdout/stderr, not system logs.** There's no `journalctl` equivalent for the cluster itself from kubectl. Kubelet logs, API server logs, and other system-level logs live on the nodes and require node-level access to inspect (or a log aggregation solution like Fluentd or Loki).
>
> **You never "log into a node" for normal operations.** Unlike SSH where you connect to a specific server, kubectl connects to the API server, which proxies everything. The API server is the single entry point — you interact with the cluster, not with individual machines. This is by design: nodes are cattle, not pets.

---

## Diagnostic Lab: Getting Comfortable with kubectl and Kind

This is the hands-on chapter. Take your time with this lab — the goal is for kubectl to feel natural by the end.

### Step 1: Create a Multi-Node Kind Cluster

Create the cluster configuration file:

```bash
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF
```

Create the cluster:

```bash
kind create cluster --name first-cluster --config kind-config.yaml
```

---

### Step 2: Explore the Cluster

Get familiar with what you're working with:

```bash
# See all nodes and their details
kubectl get nodes -o wide

# What's running across the entire cluster?
kubectl get pods -A

# Cluster endpoint info
kubectl cluster-info

# What resource types does this cluster support?
kubectl api-resources | head -20

# How many resource types are available?
kubectl api-resources --no-headers | wc -l
```

Take a moment with `kubectl get pods -A`. These are the system components you learned about in Chapter 3 — etcd, kube-apiserver, kube-scheduler, kube-controller-manager, CoreDNS, kube-proxy — all running as Pods in the `kube-system` namespace.

---

### Step 3: Deploy an Application Imperatively

Create a Deployment and expose it:

```bash
# Create a Deployment with 3 replicas
kubectl create deployment nginx --image=nginx:1.27 --replicas=3

# Expose it as a Service
kubectl expose deployment nginx --port=80 --type=NodePort
```

Verify everything was created:

```bash
kubectl get all
```

You should see the Deployment, ReplicaSet, 3 Pods, and a Service.

---

### Step 4: Inspect Everything

Now let's explore like a Linux admin would:

```bash
# Detailed view of the Deployment
kubectl describe deployment nginx

# See which node each Pod is running on
kubectl get pods -o wide

# Check the events timeline (your system log for the cluster)
kubectl get events --sort-by=.metadata.creationTimestamp

# View the full Deployment spec
kubectl get deployment nginx -o yaml

# Extract just the container image with JSONPath
kubectl get deployment nginx -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Look at the `kubectl get pods -o wide` output. Notice how the 3 Pods are distributed across the worker nodes? That's the scheduler doing its job.

---

### Step 5: Debug Like a Linux Admin

**Exec into a Pod (your SSH replacement):**

```bash
# Get a shell inside a running container
kubectl exec -it $(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}') -- /bin/sh
```

Once inside:

```sh
# Check the process list (you're inside a container)
ps aux

# Verify nginx is running
curl -s localhost

# Check the filesystem
ls /etc/nginx/

# Exit the shell
exit
```

**View container logs (your journalctl replacement):**

```bash
# Logs from a specific Pod
kubectl logs $(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')

# Follow logs in real time (like journalctl -f)
kubectl logs -f deployment/nginx

# Press Ctrl+C to stop following
```

**Port-forward to access the Service (your SSH tunnel replacement):**

```bash
# Forward local port 8080 to the Service's port 80
kubectl port-forward svc/nginx 8080:80 &

# Test it
curl -s localhost:8080

# Stop the port-forward
kill %1
```

---

### Step 6: Switch to Declarative

**Export the existing Deployment to YAML:**

```bash
kubectl get deployment nginx -o yaml > nginx-deployment.yaml
```

**Edit the file — change replicas from 3 to 5:**

```bash
sed -i 's/replicas: 3/replicas: 5/' nginx-deployment.yaml
```

> **Note:** On macOS with BSD sed, use `sed -i '' 's/replicas: 3/replicas: 5/' nginx-deployment.yaml` instead.

**Apply the change declaratively:**

```bash
kubectl apply -f nginx-deployment.yaml
```

**Verify the scaling:**

```bash
kubectl get pods -l app=nginx
```

You should see 5 Pods.

**Run diff — there should be no difference since we just applied:**

```bash
kubectl diff -f nginx-deployment.yaml
```

If there's no output (or just metadata differences like `resourceVersion`), the cluster state matches the file. This is the declarative workflow: the file is truth, and `apply` makes the cluster match.

**Generate a clean manifest from scratch (the recommended approach):**

Instead of exporting from a running resource (which includes lots of runtime fields), use dry-run to generate clean YAML:

```bash
kubectl create deployment nginx-clean --image=nginx:1.27 --replicas=3 --dry-run=client -o yaml > nginx-clean.yaml
```

Look at the difference:

```bash
wc -l nginx-deployment.yaml nginx-clean.yaml
```

The dry-run version is much shorter — it only has what you actually need.

Clean up the test manifest:

```bash
rm -f nginx-clean.yaml
```

---

### Step 7: Clean Up

```bash
# Delete using the manifest (declarative)
kubectl delete -f nginx-deployment.yaml

# Delete the Service
kubectl delete svc nginx

# Clean up the local files
rm -f nginx-deployment.yaml kind-config.yaml

# Delete the Kind cluster
kind delete cluster --name first-cluster
```

---

## Key Takeaways

1. **Kind lets you run multi-node clusters locally.** Control plane + workers, port mappings, custom configurations — all in Docker containers on your machine.

2. **Kubeconfig is your SSH config for clusters.** It defines where to connect (clusters), how to authenticate (users), and shortcuts to switch between them (contexts).

3. **kubectl follows a consistent verb-resource pattern.** Learn the verbs (`get`, `describe`, `apply`, `delete`, `exec`, `logs`) and the resource types (`pods`, `deployments`, `services`), and you can navigate any cluster.

4. **Output formats are powerful.** `-o wide` for quick debugging, `-o yaml` for full specs, `-o jsonpath` for scripting. Master these and you'll never feel lost.

5. **Imperative for exploration, declarative for production.** Use `kubectl create` to experiment fast. Use `kubectl apply -f` with version-controlled YAML for anything that matters.

6. **`--dry-run=client -o yaml` is your manifest generator.** Don't write YAML from scratch — let kubectl generate the template, then customize.

7. **`kubectl exec` and `kubectl logs` are your SSH and journalctl.** They're good enough for debugging, but remember: the Pod is ephemeral. The manifest is the source of truth, not the running container.

---

## Further Reading

- [kubectl Overview](https://kubernetes.io/docs/reference/kubectl/)
- [kubectl Quick Reference (Cheat Sheet)](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- [Organizing Cluster Access Using kubeconfig Files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
- [Kind — Configuration](https://kind.sigs.k8s.io/docs/user/configuration/)
- [Kind — Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [Managing Resources with kubectl](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/)
- [Declarative Management of Kubernetes Objects Using Configuration Files](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/)

---

**Previous:** [Chapter 4 — How Kubernetes Thinks](04-how-kubernetes-thinks.md)
**Next:** [Chapter 6 — Pods and Workloads](06-pods-and-workloads.md)
