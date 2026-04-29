# Chapter 1: From Server to Cluster

*"Everything you know about Linux still applies. You just need to see it from a higher altitude."*

---

## You Already Know More Than You Think

If you've spent years managing Linux servers — configuring services with systemd, debugging networking with ss and tcpdump, managing users and permissions, writing shell scripts that keep production running — you might look at Kubernetes and feel like you're starting from zero.

You're not.

Kubernetes didn't appear out of nowhere. It was built by people who managed Linux servers at massive scale (Google's Borg system ran on Linux). Every core concept in Kubernetes has a direct ancestor in Linux:

| What You Know (Linux) | What You'll Learn (Kubernetes) |
|------------------------|-------------------------------|
| Processes | Pods |
| systemd units | Deployments |
| iptables / nftables | Services and kube-proxy |
| /etc/fstab, mount | Volumes, PersistentVolumeClaims |
| /etc/ config files | ConfigMaps and Secrets |
| users, groups, chmod | RBAC, ServiceAccounts |
| cron | CronJobs |
| systemctl restart | Rolling updates |
| top, htop | kubectl top, Metrics Server |
| journalctl | kubectl logs |
| namespaces (unshare) | Kubernetes Namespaces |
| cgroups | Resource requests and limits |

This table isn't a gimmick — it's the thesis of this entire book. You're not learning something alien. You're learning how familiar concepts scale beyond a single machine.

## Why Kubernetes?

Let's be honest: if you're managing 3 servers, you don't need Kubernetes. A good set of Ansible playbooks and systemd units will serve you well.

But the moment you need to:

- **Run the same application on 50 machines** without manually configuring each one
- **Handle failures automatically** — if a process dies, it restarts; if a machine dies, the workload moves
- **Scale up and down** based on actual demand, not on your best guess at 2 AM
- **Deploy new versions** without downtime and roll back if something breaks
- **Give different teams** isolated environments on shared infrastructure

...that's when managing individual servers stops scaling and you start thinking in terms of clusters.

Kubernetes is what happens when you take the best ideas from Linux system administration — process management, networking, storage, security — and apply them across a fleet of machines instead of one.

### A Brief History (For Context, Not Trivia)

Kubernetes wasn't the first attempt at container orchestration, but it was the one that stuck:

- **2003-2013**: Google runs Borg internally — managing millions of containers across their infrastructure, all on Linux
- **2013**: Docker makes containers accessible to everyone (before Docker, you needed deep Linux knowledge to use namespaces and cgroups directly)
- **2014**: Google open-sources Kubernetes, distilling lessons from Borg into a community project
- **2015**: Kubernetes 1.0 released; Cloud Native Computing Foundation (CNCF) is formed
- **2018-present**: Kubernetes becomes the de facto standard for container orchestration; all major cloud providers offer managed Kubernetes (AKS, EKS, GKE)

The key insight: Kubernetes was born from Linux. The people who built it were Linux system administrators and kernel engineers. When you learn Kubernetes, you're learning the next evolution of skills you already have.

## The Mental Shift: From Server to Cluster

The hardest part of learning Kubernetes isn't the technology — it's the mindset change.

### On a Linux Server, You Think:

- "I need to install nginx on **this machine**"
- "I need to open port 80 on **this machine's** firewall"
- "I need to mount **/dev/sdb1** to **/var/www**"
- "If the service crashes, I'll set up **a systemd restart policy**"

### In Kubernetes, You Think:

- "I need **3 copies** of nginx running **somewhere** in the cluster"
- "I need a **Service** that routes traffic to any healthy nginx Pod"
- "I need **persistent storage** that survives Pod restarts, regardless of which node it runs on"
- "If a Pod crashes, the **Deployment controller** will replace it automatically"

Notice the shift: you stop caring about *which specific machine* runs your workload. You declare *what you want* (3 nginx replicas, persistent storage, a network endpoint), and Kubernetes figures out the *where* and *how*.

This is the fundamental change: **from imperative management of individual servers to declarative management of a cluster.**

> **Where the Linux Analogy Breaks**
>
> On a Linux server, you SSH in, run commands, and see immediate results. It's imperative: you tell the system exactly what to do, step by step.
>
> Kubernetes is declarative: you describe the desired end state, and a set of controllers continuously work to make reality match your description. There is no "SSH into the cluster and install nginx." Instead, you submit a YAML manifest that says "I want 3 nginx Pods," and Kubernetes creates them, monitors them, and replaces them if they fail.
>
> This shift from "I do things" to "I describe what I want" is the single biggest mental adjustment. Everything else flows from it.

## What This Book Covers

This book is organized as a progressive journey, starting from what you know and building toward production-ready Kubernetes:

**Part I — Foundations (Chapters 1-5)**

You'll bridge from Linux to containers, understand Kubernetes architecture, learn how Kubernetes "thinks" differently from a single server, and get your first cluster running.

**Part II — Core Concepts (Chapters 6-10)**

The meat of Kubernetes: workloads, networking, storage, configuration, and packaging. Each chapter maps directly to something you already do on Linux servers.

**Part III — Operations (Chapters 11-15)**

Security, scaling, observability, troubleshooting, and production readiness. This is where Linux operational instincts really pay off — and where the differences matter most.

Every chapter includes:
- **Linux-to-K8s comparison tables** — quick reference for concept mapping
- **"Where the Analogy Breaks" boxes** — honest about the limits of Linux analogies
- **Diagnostic labs** — hands-on exercises using Kind (local Kubernetes)
- **Key takeaways** — what to remember from each chapter

## Lab: Verify Your Prerequisites

Before we dive in, let's make sure your environment is ready. You'll need Docker and Kind installed — both run on Linux (or WSL2 on Windows).

### Step 1: Verify Docker

```bash
docker version
```

You should see both Client and Server versions. If Docker isn't installed, follow the [official installation guide](https://docs.docker.com/engine/install/).

### Step 2: Install Kind

Kind (Kubernetes IN Docker) runs a full Kubernetes cluster inside Docker containers. It's the fastest way to get a local cluster:

```bash
# For Linux (amd64)
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.31.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Verify the installation:

```bash
kind version
```

> **Source:** [Kind Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)

### Step 3: Install kubectl

kubectl is the command-line tool for interacting with Kubernetes clusters — think of it as your `systemctl` for the cluster:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
```

Verify:

```bash
kubectl version --client
```

> **Source:** [Install kubectl on Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

### Step 4: Create Your First Cluster

```bash
kind create cluster --name from-server-to-cluster
```

This creates a single-node Kubernetes cluster running inside a Docker container. You should see output ending with:

```
Set kubectl context to "kind-from-server-to-cluster"
```

### Step 5: Verify the Cluster

```bash
kubectl cluster-info --context kind-from-server-to-cluster
```

Expected output:

```
Kubernetes control plane is running at https://127.0.0.1:<port>
CoreDNS is running at https://127.0.0.1:<port>/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

Now explore what's running — this is a full Kubernetes cluster, and its components are actual Linux processes:

```bash
kubectl get nodes
kubectl get pods -A
```

You'll see familiar friends: `etcd`, `kube-apiserver`, `kube-scheduler`, `kube-controller-manager`, `coredns`, `kube-proxy` — all running as containers, which are just Linux processes with namespaces and cgroups.

### Clean Up (Optional)

If you want to delete the cluster and recreate it later:

```bash
kind delete cluster --name from-server-to-cluster
```

## Key Takeaways

1. **Kubernetes is an evolution of Linux skills**, not a replacement. Every core concept maps to something you already know.
2. **The fundamental shift** is from imperative (SSH and run commands) to declarative (describe desired state in YAML).
3. **You stop thinking about individual servers** and start thinking about clusters — what you want running, not where it runs.
4. **Kind** gives you a full Kubernetes cluster locally for learning — no cloud account needed.
5. **This book's approach**: start with what you know in Linux, map it to Kubernetes, and honestly flag where the analogy breaks.

## Further Reading

- [Kubernetes Documentation — Overview](https://kubernetes.io/docs/concepts/overview/)
- [Kind — Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [kubectl Installation — Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
- [The History of Kubernetes (CNCF)](https://www.cncf.io/blog/2018/07/19/the-history-of-kubernetes-the-community-behind-it/)

---

**Next:** [Chapter 2 — Containers Demystified](02-containers-demystified.md) — The Linux foundations (namespaces, cgroups) that make containers possible, and why understanding them gives you an edge.
