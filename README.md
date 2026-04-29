# From Server to Cluster

**Kubernetes for Linux Professionals**

*By [Ricardo Martins](https://ricardomartins.com.br)*

---

A hands-on book for Linux professionals transitioning to Kubernetes. If you manage servers, understand processes, and think in terms of systemd units and iptables rules — this book translates everything you already know into the Kubernetes world.

## Who This Book Is For

You're a Linux sysadmin, cloud engineer, or DevOps professional who:
- Manages Linux servers daily (or has done so)
- Understands processes, filesystems, networking, and permissions
- Wants to learn Kubernetes without starting from zero
- Prefers understanding *why* things work, not just *how*

**Prerequisite:** Solid Linux fundamentals. If you need a refresher, start with the [Linux Hackathon](https://linuxhackathon.com).

## What Makes This Book Different

Most Kubernetes books assume you're starting fresh. This one assumes you're *already good at something* — Linux — and builds on that foundation.

Every chapter opens with what you already know, maps it to Kubernetes, and honestly tells you where the analogy breaks. You'll find war stories from production, architecture decisions explained, and diagnostic labs (not just tutorials) that build systems thinking.

**This is not a challenge-based hackathon** (that's [k8shackathon.com](https://k8shackathon.com)). This book is the companion reading — the *why* behind the *what*.

## Chapters

| # | Chapter | What You'll Learn |
|---|---------|-------------------|
| 01 | [From Server to Cluster](chapters/01-from-server-to-cluster.md) | Why Kubernetes exists, what Linux pros already know, the learning path ahead |
| 02 | [Containers Demystified](chapters/02-containers-demystified.md) | Linux namespaces, cgroups, images, Dockerfiles, container runtimes |
| 03 | [Kubernetes Architecture](chapters/03-kubernetes-architecture.md) | Control plane, nodes, etcd, API server — through the Linux lens |
| 04 | [How Kubernetes Thinks](chapters/04-how-kubernetes-thinks.md) | Desired state, reconciliation loops, controllers, labels, selectors, scheduling |
| 05 | [Your First Cluster](chapters/05-your-first-cluster.md) | Kind setup, kubectl, kubeconfig, exploring cluster components |
| 06 | [Pods and Workloads](chapters/06-pods-and-workloads.md) | Pods, Deployments, ReplicaSets, DaemonSets, StatefulSets, Jobs, Probes |
| 07 | [Networking](chapters/07-networking.md) | Services, Ingress, Gateway API, DNS — from iptables to kube-proxy |
| 08 | [Storage and Persistence](chapters/08-storage-and-persistence.md) | Volumes, PV, PVC, StorageClass, CSI drivers |
| 09 | [Configuration and Secrets](chapters/09-configuration-and-secrets.md) | ConfigMaps, Secrets, environment variables — how apps consume config |
| 10 | [Packaging and Delivery](chapters/10-packaging-and-delivery.md) | Helm, Kustomize, GitOps introduction |
| 11 | [Security](chapters/11-security.md) | RBAC, Pod Security Admission, NetworkPolicy, Secrets protection, encryption at rest |
| 12 | [Scaling and Observability](chapters/12-scaling-and-observability.md) | HPA, VPA, resource management, QoS classes, Prometheus, Grafana |
| 13 | [Troubleshooting](chapters/13-troubleshooting.md) | 5 real-world scenarios: symptoms, diagnosis, fix — kubectl debug, events, logs |
| 14 | [Production Operations](chapters/14-production-operations.md) | Cluster upgrades, etcd backup/restore, disaster recovery, CRDs and Operators |
| 15 | [Next Steps](chapters/15-next-steps.md) | CKA, CKAD, CKS, KCNA certifications, career paths, community |

### Appendices

| | Appendix | Content |
|---|----------|---------|
| A | [Linux-to-Kubernetes Glossary](chapters/appendix-a-glossary.md) | systemctl to kubectl, iptables to NetworkPolicy, and 50+ mappings |
| B | [kubectl Cheatsheet](chapters/appendix-b-cheatsheet.md) | Essential commands and YAML patterns |
| C | [Cloud Platform Comparison](chapters/appendix-c-cloud-platforms.md) | AKS vs EKS vs GKE — setup, pricing, features |

## The Learning Ecosystem

This book is part of a progressive learning path:

```
linuxhackathon.com          k8shackathon.com           ai4infra.com
Linux FUNdamentals    --->  Kubernetes Hackathon  ---> AI for Infrastructure
(20 challenges)             (20 challenges)            (AI + Cloud)
        \                        |                      /
         \                       |                     /
          `----> This Book <----'                    /
           "From Server to Cluster"                 /
            (The WHY behind the WHAT)   -----------'
```

| Resource | Format | Focus |
|----------|--------|-------|
| [Linux Hackathon](https://linuxhackathon.com) | Hands-on challenges | Linux fundamentals (20 challenges) |
| **This Book** | Narrative + labs | Conceptual bridge: Linux to Kubernetes |
| [K8s Hackathon](https://k8shackathon.com) | Hands-on challenges | Kubernetes practice (20 challenges) |
| [AI for Infra](https://ai4infra.com) | Ebook | AI applied to infrastructure |

## How to Use This Book

1. **Read sequentially** the first time — chapters build on each other
2. **Run the labs** — every chapter has a diagnostic lab using [Kind](https://kind.sigs.k8s.io/)
3. **Check the comparison tables** — quick reference for Linux-to-K8s concept mapping
4. **Read the "Where the Analogy Breaks" boxes** — this is where real understanding happens
5. **Then do the hackathon** — apply what you learned at [k8shackathon.com](https://k8shackathon.com)

## Technical Requirements

- A Linux machine (physical, VM, or WSL2)
- [Docker](https://docs.docker.com/get-docker/) installed
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) installed
- [kubectl](https://kubernetes.io/docs/tasks/tools/) installed
- [Helm](https://helm.sh/docs/intro/install/) (from Chapter 10)
- 8 GB RAM minimum (16 GB recommended)

All labs are tested on **Kubernetes v1.32** with Kind. Cloud-specific examples use AKS, EKS, and GKE as variants.

## Portuguese Content

Looking for content in Portuguese? Visit [ricardomartins.com.br](https://ricardomartins.com.br) for the blog series adaptation.

## About the Author

**Ricardo Martins** is a Principal Solutions Engineer at Microsoft with deep expertise in Linux, cloud infrastructure, and Kubernetes. He is the author of [AI for Infrastructure Professionals](https://ai4infra.com), creator of the [Linux Hackathon](https://linuxhackathon.com) and [Kubernetes Hackathon](https://k8shackathon.com), and has been helping infrastructure professionals evolve their careers for over a decade.

- Blog (EN): [rmmartins.com](https://rmmartins.com)
- Blog (PT-BR): [ricardomartins.com.br](https://ricardomartins.com.br)
- GitHub: [github.com/ricmmartins](https://github.com/ricmmartins)
- LinkedIn: [linkedin.com/in/yourprofile](https://www.linkedin.com/in/yourprofile)

## License

This work is licensed under [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/).

You are free to share and adapt this material for non-commercial purposes, as long as you give appropriate credit and distribute your contributions under the same license.

---

*"You don't need to forget everything you know about Linux to learn Kubernetes. You need to see how it evolves."*
