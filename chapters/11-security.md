# Chapter 11: Security

*"The only truly secure system is one that is powered off, cast in a block of concrete, and sealed in a lead-lined room with armed guards — and even then I have my doubts."*
— Gene Spafford

---

## From chmod to RBAC: Same Goal, Different Scale

On a Linux server, security is physical and intimate. You know who has root access because you manage `/etc/passwd` and `/etc/shadow`. You control what processes can do with file permissions (`chmod 640`), what users can execute with `sudoers`, and what network traffic is allowed with `iptables`. Every security decision comes down to the same three questions: **who** is requesting access, **what** are they trying to do, and **to which resource**?

Kubernetes asks the exact same three questions — but at cluster scale.

Instead of Linux users, you have **ServiceAccounts** — identities that pods use to authenticate against the API server. Instead of `/etc/sudoers`, you have **Roles** and **ClusterRoles** — YAML manifests that define which API verbs (get, list, create, delete) are permitted on which resources (pods, services, secrets). Instead of adding a user to the `sudo` group, you create a **RoleBinding** that attaches a Role to a ServiceAccount.

The mental model is the same: *who can do what to which resources.* The mechanism is different because Kubernetes is an API-driven distributed system, not a single machine with a filesystem. Every action flows through the API server, and every request is authenticated, authorized, and admitted (or rejected) before anything happens.

For a Linux admin, the good news is that your security instincts — least privilege, defense in depth, explicit deny — all apply. You just need to learn the Kubernetes vocabulary.

---

## RBAC: Role-Based Access Control

RBAC is the authorization backbone of Kubernetes. It was promoted to GA in Kubernetes 1.8 and is enabled by default on every modern cluster. If you understand Linux users, groups, and sudoers, you already understand the conceptual foundation.

### ServiceAccounts — Identity for Pods

Every pod in Kubernetes runs with a ServiceAccount. If you don't specify one, it uses the `default` ServiceAccount in the namespace. Think of ServiceAccounts as system users in Linux — they're identities for processes, not people.

```bash
# Every namespace gets a 'default' ServiceAccount automatically
kubectl get serviceaccounts
```

The critical thing: the `default` ServiceAccount often has more permissions than you'd expect, especially in older clusters. The first rule of Kubernetes security is **don't use the default ServiceAccount for production workloads.** Create dedicated ServiceAccounts with only the permissions each workload needs.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: default
```

### Roles and ClusterRoles — Permission Definitions

A **Role** defines what actions are permitted on which resources, scoped to a single namespace. A **ClusterRole** does the same but across the entire cluster.

```yaml
# A Role that permits read-only access to pods in a namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]            # "" = core API group
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

Think of this as a sudoers rule: "This identity can run `get`, `list`, and `watch` on pods in the default namespace." No create, no delete, no update.

A **ClusterRole** works the same way but without namespace scoping:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
```

### RoleBindings and ClusterRoleBindings — Attaching Permissions

Roles define permissions. Bindings attach those permissions to identities. A **RoleBinding** grants a Role within a specific namespace. A **ClusterRoleBinding** grants a ClusterRole across all namespaces.

```yaml
# Bind the pod-reader Role to the my-app ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: my-app
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

The `subjects` list is *who* gets the permission. The `roleRef` is *which* permission set. You can bind the same Role to multiple ServiceAccounts, or bind multiple Roles to the same ServiceAccount via separate RoleBindings.

### Built-in Roles

Kubernetes ships with several default ClusterRoles that cover common use cases:

| ClusterRole | Permissions | Linux Equivalent |
|-------------|------------|------------------|
| `cluster-admin` | Everything, everywhere | `root` |
| `admin` | Full access within a namespace (including RBAC) | Namespace-level `sudo` |
| `edit` | Read/write most resources, but no RBAC | Power user |
| `view` | Read-only access to most resources | Audit / read-only user |

**The principle of least privilege applies.** Don't hand out `cluster-admin` when `view` will do. Every over-permissioned ServiceAccount is a potential blast radius expansion.

---

## Pod Security Admission (PSA)

If RBAC controls *who can create workloads*, Pod Security Admission controls *what kind of workloads can run*. It's the Kubernetes equivalent of AppArmor or SELinux profiles — restrictions on what a process (pod) is allowed to do at runtime.

### A Brief History

Kubernetes previously used PodSecurityPolicies (PSPs) for this purpose. PSPs were deprecated in Kubernetes 1.21 and **removed entirely in Kubernetes 1.25**. The replacement is the built-in Pod Security Admission controller, which is simpler, namespace-scoped, and uses predefined security profiles.

### Three Security Levels

Pod Security Admission defines three progressively restrictive profiles, called **Pod Security Standards**:

| Level | What It Allows | Use Case |
|-------|---------------|----------|
| **Privileged** | Anything goes — no restrictions | System-level infrastructure (CNI, logging agents) |
| **Baseline** | Blocks known privilege escalations (hostNetwork, hostPID, privileged containers) | Most application workloads |
| **Restricted** | Requires non-root, drops all capabilities, read-only root filesystem, seccomp profiles | Security-sensitive workloads |

### Three Enforcement Modes

For each namespace, you can configure Pod Security Admission in three modes:

| Mode | Behavior |
|------|----------|
| **enforce** | Rejects pods that violate the policy |
| **audit** | Allows the pod but logs a violation in the audit log |
| **warn** | Allows the pod but displays a warning to the user |

You apply these via namespace labels:

```bash
# Enforce the restricted profile on a namespace
kubectl label namespace secure-ns \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest

# Warn about baseline violations (but don't block)
kubectl label namespace dev-ns \
  pod-security.kubernetes.io/warn=baseline \
  pod-security.kubernetes.io/warn-version=latest
```

The recommended pattern for migrating existing workloads: start with `warn` to see what would break, then `audit` to track violations without disruption, then `enforce` when you're confident.

---

## NetworkPolicy — Firewall Rules for Pods

By default, Kubernetes networking is **wide open**. Every pod can talk to every other pod in the cluster, regardless of namespace. There's no firewall, no segmentation, no isolation. It's like running a Linux server with `iptables -P INPUT ACCEPT` and no rules at all.

**NetworkPolicy** objects are the Kubernetes firewall. They define which pods can talk to which other pods, on which ports, in which direction.

### How NetworkPolicies Work

1. **No NetworkPolicy = no restrictions.** Pods accept traffic from everywhere.
2. **Once you apply a NetworkPolicy selecting a pod, that pod becomes isolated.** Only traffic explicitly allowed by a policy is permitted.
3. **Policies are additive (whitelist model).** You can't write a "deny this specific connection" rule. You write "allow these connections" and everything else is implicitly denied.

### Anatomy of a NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend            # This policy applies to pods with app=backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend       # Allow traffic FROM pods with app=frontend
    ports:
    - protocol: TCP
      port: 80                # Only on port 80
```

This reads: "Pods labeled `app: backend` accept incoming TCP traffic on port 80, but only from pods labeled `app: frontend`. All other ingress to `backend` pods is denied."

### Default Deny Policies

The most important NetworkPolicy pattern is the **default deny** — block all traffic, then selectively whitelist:

```yaml
# Default deny ALL ingress in the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}             # Empty selector = applies to ALL pods
  policyTypes:
  - Ingress                   # Block all incoming traffic
```

```yaml
# Default deny ALL egress in the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress                    # Block all outgoing traffic
```

> **Important:** NetworkPolicies require a CNI plugin that supports them. Not all CNIs do. **kindnet** (Kind's default CNI) has limited NetworkPolicy support. For production or full-featured testing, use **Calico** or **Cilium**.

---

## Secrets Protection

Chapter 9 introduced Secrets as a Kubernetes resource. But storing a Secret in the cluster is only half the battle — you also need to protect it.

### The Problem with Default Secrets

By default, Kubernetes Secrets are:

1. **Base64-encoded, not encrypted.** Anyone who can read the Secret can decode it instantly with `base64 -d`.
2. **Stored unencrypted in etcd.** If someone gains access to the etcd data directory, they can read every Secret in the cluster.
3. **Accessible to anyone with `get secret` RBAC permissions.** A misconfigured RoleBinding can expose every password and API key in a namespace.

### Encryption at Rest

Kubernetes supports encrypting Secret data in etcd via an `EncryptionConfiguration` resource. This ensures that even if someone accesses the etcd storage directly, the Secret values are encrypted:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}       # Fallback: read unencrypted data written before encryption was enabled
```

This is configured on the API server via the `--encryption-provider-config` flag. In managed Kubernetes services (AKS, EKS, GKE), encryption at rest is typically enabled by default.

### External Secret Management

For production environments, the best practice is to store secrets *outside* the cluster entirely:

- **External Secrets Operator (ESO):** Syncs secrets from external stores (AWS Secrets Manager, Azure Key Vault, HashiCorp Vault, GCP Secret Manager) into Kubernetes Secrets automatically.
- **Sealed Secrets:** Encrypts Secrets client-side so the encrypted version can be safely stored in Git. Only the cluster can decrypt them.

The golden rule: **never store plaintext secrets in Git.** Not in manifests, not in Helm values files, not in environment variable definitions. Use external secret management or encrypted-at-rest solutions.

---

## Security Context — Per-Pod and Per-Container Permissions

A SecurityContext defines privilege and access control settings for a pod or individual container. This is where Kubernetes maps most directly to Linux security primitives — because it literally configures the Linux security features of the container runtime.

### Key Fields

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:                    # Pod-level settings
    runAsNonRoot: true                # Refuse to start if image runs as root
    runAsUser: 1000                   # Run as UID 1000
    runAsGroup: 3000                  # Run as GID 3000
    fsGroup: 2000                     # Supplemental group for volume mounts
    seccompProfile:
      type: RuntimeDefault            # Apply the container runtime's default seccomp filter
  containers:
  - name: app
    image: nginx:1.27
    securityContext:                   # Container-level settings (override pod-level)
      allowPrivilegeEscalation: false # Block setuid/setgid bits
      readOnlyRootFilesystem: true    # Mount root filesystem as read-only
      capabilities:
        drop:
        - ALL                         # Drop all Linux capabilities
        add:
        - NET_BIND_SERVICE            # Add back only what's needed
```

### Linux Capabilities in Kubernetes

Linux capabilities are a direct mapping. The `capabilities` field in a SecurityContext is the exact same mechanism as `setcap` and `capsh` on Linux:

| Capability | What It Allows |
|-----------|---------------|
| `NET_BIND_SERVICE` | Bind to ports below 1024 |
| `SYS_PTRACE` | Trace/debug other processes |
| `SYS_ADMIN` | A grab-bag of privileged operations (mount, namespace ops) |
| `NET_RAW` | Use raw sockets (needed for `ping`) |
| `CHOWN` | Change file ownership |

The best practice: **drop ALL capabilities, then add back only what the container actually needs.** This is the capability equivalent of least privilege.

### Seccomp Profiles

Seccomp (Secure Computing Mode) is a Linux kernel feature that restricts which system calls a process can make. Kubernetes exposes this via the `seccompProfile` field:

- `RuntimeDefault` — Uses the container runtime's default seccomp profile (recommended baseline)
- `Localhost` — References a custom seccomp profile on the node
- `Unconfined` — No seccomp filtering (not recommended)

---

## Linux ↔ Kubernetes Comparison

| Linux Concept | Kubernetes Equivalent | Notes |
|---------------|----------------------|-------|
| Users and groups | ServiceAccounts | Identity for processes/pods |
| `/etc/sudoers` | Role / ClusterRole | What actions are permitted |
| `sudo` group membership | RoleBinding / ClusterRoleBinding | Granting permissions to an identity |
| `chmod`, `chown` | SecurityContext | Per-pod/container permissions |
| `iptables` INPUT/OUTPUT chains | NetworkPolicy ingress/egress | Traffic filtering |
| AppArmor/SELinux profiles | Pod Security Admission | Restrict what pods can do |
| Capabilities (`capsh`, `setcap`) | securityContext.capabilities | Fine-grained privilege control |
| `/etc/shadow` permissions | Secrets + RBAC | Restricting access to sensitive data |

---

> ### 🔒 Where the Linux Analogy Breaks
>
> - **Linux security is machine-scoped. Kubernetes RBAC is cluster-scoped.** On Linux, one misconfigured user affects one machine. In Kubernetes, a misconfigured ClusterRoleBinding can grant permissions across *every* namespace in the cluster. A single `cluster-admin` binding given to the wrong ServiceAccount is equivalent to giving `root` on every server simultaneously. The blast radius is the entire cluster.
>
> - **There's no `su` or `sudo` in Kubernetes.** You can't temporarily elevate privileges. Your ServiceAccount either has permission or it doesn't — there's no "enter the password to run this one command as admin." Kubernetes does support impersonation (`--as=` flag), but using it requires explicit RBAC grants for the impersonation verbs. It's a deliberate design choice: no ambient authority, no privilege escalation paths.
>
> - **NetworkPolicies are additive (whitelist model).** On Linux, `iptables` default is often "accept all, block specific traffic" — you add DROP rules for things you want to block. In Kubernetes, once you apply a default deny policy, **nothing** can communicate until you explicitly allow it. You build up from zero, not down from everything. This catches people off guard: apply a default deny, and suddenly your entire namespace goes silent.

---

## Diagnostic Lab: Kubernetes Security in Practice

### Prerequisites

Create a Kind cluster with Calico for full NetworkPolicy support:

```bash
cat <<'EOF' > kind-security-lab.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: "192.168.0.0/16"
nodes:
- role: control-plane
- role: worker
EOF

kind create cluster --name security-lab --config kind-security-lab.yaml
```

Install Calico CNI:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.3/manifests/calico.yaml
```

Wait for Calico pods to become ready:

```bash
kubectl wait --for=condition=Ready pods -l k8s-app=calico-node -n kube-system --timeout=120s
kubectl get nodes
```

All nodes should show `Ready` status.

### Lab 1: RBAC — ServiceAccounts, Roles, and RoleBindings

**Step 1 — Create a ServiceAccount:**

```bash
kubectl create serviceaccount dev-user
```

**Step 2 — Create a Role that only allows reading pods:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
EOF
```

**Step 3 — Create a RoleBinding to attach the Role to the ServiceAccount:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-user-pod-reader
  namespace: default
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

**Step 4 — Test the permissions:**

```bash
# Should return "yes" — dev-user can list pods
kubectl auth can-i list pods --as=system:serviceaccount:default:dev-user

# Should return "yes" — dev-user can get pods
kubectl auth can-i get pods --as=system:serviceaccount:default:dev-user

# Should return "no" — dev-user cannot delete pods
kubectl auth can-i delete pods --as=system:serviceaccount:default:dev-user

# Should return "no" — dev-user cannot create deployments
kubectl auth can-i create deployments --as=system:serviceaccount:default:dev-user

# Should return "no" — dev-user cannot get secrets
kubectl auth can-i get secrets --as=system:serviceaccount:default:dev-user
```

This is least privilege in action: `dev-user` can read pods and nothing else.

**Step 5 — Explore what the ServiceAccount can do across the board:**

```bash
kubectl auth can-i --list --as=system:serviceaccount:default:dev-user
```

This outputs a complete list of permissions — useful for auditing.

### Lab 2: Pod Security Admission

**Step 1 — Create a namespace with the restricted PSA profile:**

```bash
kubectl create namespace secure-ns

kubectl label namespace secure-ns \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest
```

**Step 2 — Try deploying a privileged pod (should be REJECTED):**

```bash
cat <<'EOF' | kubectl apply -n secure-ns -f -
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    securityContext:
      privileged: true
EOF
```

You should see an error like:
```
Error from server (Forbidden): error when creating "STDIN": pods "privileged-pod"
is forbidden: violates PodSecurity "restricted:latest"...
```

**Step 3 — Deploy a compliant pod (should SUCCEED):**

```bash
cat <<'EOF' | kubectl apply -n secure-ns -f -
apiVersion: v1
kind: Pod
metadata:
  name: compliant-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: busybox:1.37
    command: ["sleep", "3600"]
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
EOF
```

**Step 4 — Verify the compliant pod is running:**

```bash
kubectl get pods -n secure-ns
kubectl exec -n secure-ns compliant-pod -- id
kubectl exec -n secure-ns compliant-pod -- whoami
```

The pod runs as UID 1000 — a non-root user.

### Lab 3: NetworkPolicy

**Step 1 — Deploy two applications (frontend and backend):**

```bash
# Deploy backend
kubectl run backend --image=nginx:1.27 --labels="app=backend" --port=80
kubectl expose pod backend --port=80 --target-port=80

# Deploy frontend (a long-lived pod for testing connectivity)
kubectl run frontend --image=busybox:1.37 --labels="app=frontend" -- sleep 3600

# Wait for pods to be ready
kubectl wait --for=condition=Ready pod/backend --timeout=60s
kubectl wait --for=condition=Ready pod/frontend --timeout=60s
```

**Step 2 — Verify they can communicate (no policies yet):**

```bash
kubectl exec frontend -- wget -qO- --timeout=5 http://backend
```

You should see the nginx welcome page HTML. The network is wide open.

**Step 3 — Apply a default-deny-all ingress policy:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF
```

**Step 4 — Verify frontend can NO LONGER reach backend:**

```bash
kubectl exec frontend -- wget -qO- --timeout=5 http://backend
```

This should timeout — the default deny policy blocks all ingress traffic.

**Step 5 — Add a policy that allows frontend → backend on port 80:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
EOF
```

**Step 6 — Verify communication is restored:**

```bash
kubectl exec frontend -- wget -qO- --timeout=5 http://backend
```

You should see the nginx welcome page again. The whitelist policy selectively allows frontend → backend while everything else remains blocked.

**Step 7 — Verify that other pods still can't reach backend:**

```bash
# Create a pod without the frontend label
kubectl run outsider --image=busybox:1.37 --labels="app=outsider" -- sleep 3600
kubectl wait --for=condition=Ready pod/outsider --timeout=60s

# Try to reach backend — should timeout
kubectl exec outsider -- wget -qO- --timeout=5 http://backend
```

The outsider pod is blocked because it doesn't match the `app: frontend` label selector.

### Lab 4: Security Context

**Step 1 — Deploy a pod with strict security settings:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: busybox:1.37
    command: ["sleep", "3600"]
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
EOF
```

**Step 2 — Verify the security constraints are active:**

```bash
# Check UID and GID — should be uid=1000, gid=3000
kubectl exec hardened-pod -- id

# Try to write to the root filesystem — should fail (read-only)
kubectl exec hardened-pod -- touch /test

# Try to write to /tmp — also fails (entire root FS is read-only)
kubectl exec hardened-pod -- touch /tmp/test
```

The `touch` commands should fail with "Read-only file system." The container can only write to explicitly mounted writable volumes (emptyDir, PVC).

**Step 3 — Check that the pod has the strictest QoS settings reflected:**

```bash
kubectl get pod hardened-pod -o jsonpath='{.spec.securityContext}' | python -m json.tool
```

### Clean Up

```bash
# Delete all lab resources
kubectl delete pod frontend backend outsider hardened-pod --ignore-not-found
kubectl delete svc backend --ignore-not-found
kubectl delete networkpolicy default-deny-ingress allow-frontend-to-backend --ignore-not-found
kubectl delete rolebinding dev-user-pod-reader --ignore-not-found
kubectl delete role pod-reader --ignore-not-found
kubectl delete serviceaccount dev-user --ignore-not-found
kubectl delete namespace secure-ns --ignore-not-found

# Delete the Kind cluster
kind delete cluster --name security-lab

# Remove the cluster config file
rm -f kind-security-lab.yaml
```

---

## Key Takeaways

1. **RBAC is the foundation of Kubernetes security.** Every pod runs with a ServiceAccount, every action requires authorization, and the principle of least privilege should guide every Role and RoleBinding you create. Never use `cluster-admin` when a namespace-scoped Role will do.

2. **Pod Security Admission replaces the deprecated PodSecurityPolicies.** Use the three predefined levels (Privileged, Baseline, Restricted) with namespace labels. Start with `warn` mode, graduate to `enforce` once you've verified compliance.

3. **The default Kubernetes network is wide open.** Without NetworkPolicies, every pod can talk to every other pod. Apply default-deny policies first, then selectively whitelist required communication paths — just like configuring a firewall.

4. **NetworkPolicies require a supporting CNI.** Kind's default `kindnet` has limited support. For production or thorough testing, use Calico or Cilium.

5. **Security Context maps directly to Linux security primitives.** `runAsNonRoot`, `capabilities`, `readOnlyRootFilesystem`, and `seccompProfile` are all configuring Linux kernel features via Kubernetes API objects. Your Linux security knowledge applies directly here.

6. **Secrets are not secure by default.** They're base64-encoded (not encrypted), stored unencrypted in etcd unless you enable encryption at rest, and accessible to anyone with the right RBAC permissions. Treat Secrets as the starting point, not the finish line — use external secret management for production.

7. **Defense in depth is the only strategy.** RBAC controls who can do what. PSA controls what workloads can run. NetworkPolicy controls what can talk to what. SecurityContext controls what a container can do at the OS level. Use all of them together — each layer catches what the others miss.

---

## Further Reading

- [RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Encrypting Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- [Declare Network Policy](https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/)

---

**Previous:** [Chapter 10 — Packaging and Delivery](10-packaging-and-delivery.md)
**Next:** [Chapter 12 — Scaling and Observability](12-scaling-and-observability.md)
