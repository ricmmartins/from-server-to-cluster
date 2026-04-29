# Chapter 10: Packaging and Delivery

*"Nobody wants to manage a thousand YAML files by hand. That's not DevOps — that's suffering."*

---

## From Artisanal YAML to Repeatable Deployments

On a Linux server, you don't install software by manually copying binaries and writing config files from scratch. You use a package manager — `apt install nginx`, `dnf install postgresql`, `brew install redis`. The package manager handles dependencies, versioning, upgrades, rollbacks, and uninstalls. It's civilized. It's repeatable. It's how professionals work.

Now look at what we've been doing in Kubernetes for the last nine chapters: writing individual YAML files by hand, applying them one at a time with `kubectl apply -f`. For a single service with a Deployment, a Service, a ConfigMap, and a Secret, that's four YAML files. Add an Ingress, a PVC, a ServiceAccount, and RBAC rules, and you're up to eight. Multiply by three environments (dev, staging, production) and you have twenty-four files to manage. For one service.

In a real microservices architecture with dozens of services, you're looking at hundreds of YAML files. Different environments need different replica counts, different image tags, different resource limits, different ConfigMap values. Copying and editing YAML files across directories is the Kubernetes equivalent of managing servers by SSH'ing in and editing config files by hand — it works until it doesn't, and then it fails spectacularly.

This chapter introduces the tools that solve this problem: **Helm** (the package manager), **Kustomize** (the patch engine), and the concept of **GitOps** (the deployment model). Together, they transform Kubernetes from "artisanal YAML craft" into a repeatable, auditable, production-grade delivery pipeline.

---

## Helm — The Package Manager for Kubernetes

Helm is to Kubernetes what `apt` is to Debian or `dnf` is to Fedora. It packages multiple Kubernetes resources into a single, versioned, configurable unit called a **chart**.

### Core Concepts

| Helm Term | Linux Equivalent | What It Is |
|-----------|-----------------|------------|
| **Chart** | `.deb` or `.rpm` package | A bundle of templated Kubernetes manifests |
| **Release** | An installed package | A specific instance of a chart running in a cluster |
| **Values** | `debconf` answers / dpkg-reconfigure | Configuration parameters passed at install time |
| **Repository** | APT source list (`/etc/apt/sources.list`) | A server that hosts charts |
| **Template** | Jinja2 / ERB templates | Go templates that generate YAML from values |
| **Revision** | Package version history | Each `helm upgrade` creates a new revision |

### How Helm Works

A Helm chart is a directory with a specific structure:

```
my-chart/
├── Chart.yaml          # Metadata (name, version, description)
├── values.yaml         # Default configuration values
├── templates/          # Go template files that generate K8s manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── _helpers.tpl    # Template helper functions
└── charts/             # Dependencies (sub-charts)
```

The `templates/` directory contains Kubernetes manifests with Go template placeholders. For example, `templates/deployment.yaml` might look like:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.port }}
```

And `values.yaml` provides the defaults:

```yaml
replicaCount: 1
image:
  repository: nginx
  tag: "1.27"
service:
  port: 80
  type: ClusterIP
```

When you run `helm install`, Helm renders the templates by substituting values, producing plain Kubernetes YAML that gets applied to the cluster. You can override any value at install time with `--set` or `--values`.

### Helm Repositories and Artifact Hub

Charts are distributed through repositories — HTTP servers (or OCI registries) hosting packaged charts. You add a repository, update the index, and search for charts just like you would with `apt`:

| APT Command | Helm Equivalent |
|-------------|----------------|
| `add-apt-repository ppa:...` | `helm repo add bitnami https://charts.bitnami.com/bitnami` |
| `apt update` | `helm repo update` |
| `apt search nginx` | `helm search repo nginx` |
| `apt show nginx` | `helm show chart bitnami/nginx` |
| `apt install nginx` | `helm install my-nginx bitnami/nginx` |

[Artifact Hub](https://artifacthub.io/) is the public directory for finding Helm charts — think of it as the Kubernetes equivalent of `packages.ubuntu.com` or `hub.docker.com` for charts. It indexes charts from multiple repositories and provides search, documentation, and version history.

### Helm Lifecycle Commands

The core workflow:

```bash
# Install a chart (creates a release)
helm install <release-name> <chart> [--set key=value] [--values custom.yaml]

# List installed releases
helm list

# Show current values of a release
helm get values <release-name>

# Upgrade a release (new revision)
helm upgrade <release-name> <chart> [--set key=value]

# Rollback to a previous revision
helm rollback <release-name> <revision-number>

# View release history
helm history <release-name>

# Uninstall a release
helm uninstall <release-name>

# Render templates locally without installing (great for debugging)
helm template <release-name> <chart> [--set key=value]
```

The `helm template` command is particularly useful — it renders the chart to plain YAML without touching the cluster. This is how you debug template issues or feed Helm output into other tools (like Kustomize, which we'll cover next).

---

## Kustomize — Patch Without Templates

Kustomize takes a fundamentally different approach from Helm. Instead of templates with placeholders, Kustomize works with plain, valid Kubernetes YAML and applies patches on top. There's no templating language, no special syntax — just YAML transformations.

### The Base + Overlays Model

Kustomize uses a layered structure:

```
app/
├── base/                    # The base manifests (valid YAML)
│   ├── kustomization.yaml   # Declares which resources to include
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/                 # Dev-specific patches
    │   └── kustomization.yaml
    └── prod/                # Prod-specific patches
        └── kustomization.yaml
```

The **base** contains your standard Kubernetes manifests — no placeholders, no special syntax. These are the same YAML files you'd apply with `kubectl apply -f`.

**Overlays** contain patches that modify the base for a specific environment. The dev overlay might reduce replicas to 1 and use a `latest` image tag. The prod overlay might increase replicas to 5, set resource limits, and add pod anti-affinity rules.

This is like Linux alternatives and symlinks — the base software is the same, but the system-specific configuration (which binary to use, which config file to load) varies.

### How Kustomize Works

The `kustomization.yaml` file in each directory tells Kustomize what resources to include and what transformations to apply:

**base/kustomization.yaml:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
```

**overlays/dev/kustomization.yaml:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
patches:
- patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-app
    spec:
      replicas: 1
namePrefix: dev-
namespace: dev
```

**overlays/prod/kustomization.yaml:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
patches:
- patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-app
    spec:
      replicas: 5
namePrefix: prod-
namespace: production
```

Build and apply:

```bash
# Preview what dev overlay produces
kubectl kustomize overlays/dev

# Preview what prod overlay produces
kubectl kustomize overlays/prod

# Apply the dev overlay
kubectl apply -k overlays/dev

# Apply the prod overlay
kubectl apply -k overlays/prod
```

The `-k` flag is built into kubectl — Kustomize is integrated directly, no separate installation needed. The `kubectl kustomize` command renders the output without applying, which is useful for review and debugging.

### What Kustomize Can Do

| Transformation | Description |
|----------------|-------------|
| `namePrefix` / `nameSuffix` | Add prefix/suffix to all resource names |
| `namespace` | Override the namespace for all resources |
| `commonLabels` | Add labels to all resources and selectors |
| `commonAnnotations` | Add annotations to all resources |
| `images` | Override image names and tags without patching |
| `patches` | Strategic merge patches or JSON patches |
| `configMapGenerator` | Generate ConfigMaps from files or literals |
| `secretGenerator` | Generate Secrets from files or literals |

The `images` transformation is particularly clean — you can change image tags across all deployments without writing a patch:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
images:
- name: my-app
  newTag: v2.1.0
```

---

## Helm vs. Kustomize — When to Use Which

Both tools solve the "managing YAML across environments" problem, but from different angles:

| Aspect | Helm | Kustomize |
|--------|------|-----------|
| **Approach** | Templates with values substitution | Patches on top of plain YAML |
| **Distribution** | Charts are packages you share | Bases are just directories you copy or reference |
| **Lifecycle** | Full lifecycle: install, upgrade, rollback, uninstall | Build tool only — uses `kubectl apply/delete` |
| **Complexity** | Higher — Go templates, helpers, hooks, dependencies | Lower — pure YAML, no templating language |
| **Best for** | Third-party apps, complex parameterization | Internal apps, environment-specific customization |
| **State tracking** | Helm tracks release history (revisions) | No state — relies on kubectl and Git |

**Use Helm when:**
- You're installing third-party software (databases, monitoring stacks, ingress controllers)
- You need to distribute reusable packages to other teams
- You need lifecycle management (rollback to revision 3, track who installed what)
- The application has many configurable parameters

**Use Kustomize when:**
- You're customizing your own team's manifests across environments
- You want to avoid learning a templating language
- You need to make small, targeted changes to existing YAML
- You want to keep your base manifests as valid, readable Kubernetes YAML

**Use both together when:**
- You install a Helm chart but need environment-specific tweaks that aren't exposed as Helm values
- You render a chart with `helm template` and apply Kustomize patches on top

---

## GitOps — A Brief Introduction

We've covered how to package (Helm) and customize (Kustomize) your Kubernetes manifests. But how do you get them into the cluster? So far, we've been running `kubectl apply` from our laptops. In production, this is problematic:

- Who applied what, and when? There's no audit trail.
- Did someone make a manual change with `kubectl edit` that nobody else knows about?
- How do you reproduce the exact state of the cluster?

**GitOps** solves this by making Git the single source of truth for your cluster's desired state. The workflow flips from "push" to "pull":

**Traditional (push):**
```
Developer → kubectl apply → Cluster
```

**GitOps (pull):**
```
Developer → git push → Git repo ← GitOps controller → Cluster
```

A GitOps controller (like [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) or [Flux](https://fluxcd.io/)) runs inside the cluster and continuously watches a Git repository. When it detects a change (new commit, updated values), it syncs the cluster to match the Git state. If someone makes a manual change with `kubectl`, the controller reverts it — Git always wins.

Think of it as `ansible-pull` or `puppet agent` for Kubernetes — the system pulls its configuration from a central source instead of having an admin push it.

### Why GitOps Matters

| Benefit | How |
|---------|-----|
| **Audit trail** | Every change is a Git commit with author, timestamp, and message |
| **Reproducibility** | The cluster state is defined in code, not in someone's terminal history |
| **Rollback** | `git revert` rolls back the cluster, not just the code |
| **Security** | Developers push to Git; they don't need `kubectl` access to the cluster |
| **Drift detection** | The controller alerts when the cluster state differs from Git |

We won't build a full GitOps pipeline in this chapter — that's a topic that deserves its own deep dive. But understanding the concept is essential because Helm and Kustomize are the *building blocks* of GitOps. ArgoCD and Flux natively understand both Helm charts and Kustomize overlays, making everything we've covered in this chapter directly applicable to a GitOps workflow.

---

## Linux ↔ Kubernetes Comparison

| Linux Concept | K8s Equivalent | Notes |
|---------------|----------------|-------|
| `apt` / `yum` / `dnf` | Helm | Package manager with repos, install, upgrade, rollback |
| `.deb` / `.rpm` package | Helm Chart | Reusable, versioned bundle of resources |
| APT source list (`sources.list`) | Helm Repository | Where to find charts |
| `debconf` / `dpkg-reconfigure` | `values.yaml` / `--set` | Installation-time configuration |
| `/etc/alternatives`, symlinks | Kustomize overlays | Environment-specific variants of the same base |
| Puppet/Ansible templates | Helm templates (Go) | Parameterized configuration generation |
| `git` + `ansible-pull` | GitOps (ArgoCD / Flux) | Pull-based declarative deployment |
| `dpkg --list` | `helm list` | See what's installed |
| `apt rollback` (via snapshots) | `helm rollback` | Revert to a previous version |

---

> ### 🔀 Where the Linux Analogy Breaks
>
> - **Helm is more than a package manager.** `apt` installs packages and tracks what's installed. Helm does that *plus* manages release state — it knows every revision of every release, what values were used, and can roll back to any previous revision. It's a package manager, a deployment tracker, and a rollback engine combined. On Linux, you'd need `apt` + `etckeeper` + `timeshift` to get comparable functionality.
>
> - **Kustomize doesn't "install" anything.** There's no `kustomize install` or `kustomize uninstall`. Kustomize is a YAML build tool — it transforms manifests and outputs plain YAML. You still use `kubectl apply` to create resources and `kubectl delete` to remove them. It has no concept of releases, revisions, or rollback. Think of it as a preprocessor, not a package manager.
>
> - **GitOps inverts the deployment model.** Traditional Linux deployment is push-based: you SSH in, run commands, and the system changes. GitOps is pull-based: you push to Git, and a controller inside the cluster detects the change and applies it. If you make a manual change with `kubectl edit`, the GitOps controller will revert it to match Git. This is a fundamentally different relationship between the operator and the system — the controller is the operator, not you.

---

## Diagnostic Lab: Packaging Hands-On

### Prerequisites

Make sure you have Helm installed:

```bash
# Check if Helm is installed
helm version

# If not installed, see https://helm.sh/docs/intro/install/
# On macOS: brew install helm
# On Linux: curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
# On Windows: choco install kubernetes-helm
```

Create a Kind cluster:

```bash
kind create cluster --name packaging-lab
```

### Lab 1: Helm Basics

**Step 1 — Add a chart repository:**

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

**Step 2 — Search for charts:**

```bash
helm search repo bitnami/nginx
```

You'll see available versions of the Bitnami nginx chart.

**Step 3 — Inspect the chart before installing:**

```bash
# Show chart metadata
helm show chart bitnami/nginx

# Show the default values (this is what you can configure)
helm show values bitnami/nginx | head -50
```

**Step 4 — Install the chart:**

```bash
helm install my-nginx bitnami/nginx --set service.type=ClusterIP
```

**Step 5 — Explore the release:**

```bash
# List installed releases
helm list

# Show the values used for this release
helm get values my-nginx

# Show all resources created by the release
helm get manifest my-nginx | head -40

# Check the pods and services
kubectl get pods -l app.kubernetes.io/instance=my-nginx
kubectl get svc -l app.kubernetes.io/instance=my-nginx
```

**Step 6 — Upgrade the release:**

```bash
helm upgrade my-nginx bitnami/nginx \
  --set service.type=ClusterIP \
  --set replicaCount=3

# Check that we now have 3 replicas
kubectl get pods -l app.kubernetes.io/instance=my-nginx

# View release history
helm history my-nginx
```

**Step 7 — Rollback:**

```bash
# Rollback to revision 1 (the original install)
helm rollback my-nginx 1

# Verify we're back to 1 replica
kubectl get pods -l app.kubernetes.io/instance=my-nginx

# History now shows the rollback
helm history my-nginx
```

**Step 8 — Uninstall:**

```bash
helm uninstall my-nginx

# Verify everything is cleaned up
kubectl get pods -l app.kubernetes.io/instance=my-nginx
helm list
```

### Lab 2: Kustomize Basics

**Step 1 — Create the base manifests:**

```bash
mkdir -p kustomize-demo/base kustomize-demo/overlays/dev kustomize-demo/overlays/prod
```

Create the base Deployment. Save as `kustomize-demo/base/deployment.yaml`:

```bash
cat <<'EOF' > kustomize-demo/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: nginx:1.27
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
EOF
```

Create the base Service. Save as `kustomize-demo/base/service.yaml`:

```bash
cat <<'EOF' > kustomize-demo/base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF
```

Create the base `kustomization.yaml`:

```bash
cat <<'EOF' > kustomize-demo/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
EOF
```

**Step 2 — Create the dev overlay:**

```bash
cat <<'EOF' > kustomize-demo/overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
namePrefix: dev-
commonLabels:
  environment: dev
patches:
- patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-app
    spec:
      replicas: 1
EOF
```

**Step 3 — Create the prod overlay:**

```bash
cat <<'EOF' > kustomize-demo/overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
namePrefix: prod-
commonLabels:
  environment: production
patches:
- patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-app
    spec:
      replicas: 5
images:
- name: nginx
  newTag: "1.27-alpine"
EOF
```

**Step 4 — Compare the outputs:**

```bash
echo "=== DEV OVERLAY ==="
kubectl kustomize kustomize-demo/overlays/dev
echo ""
echo "=== PROD OVERLAY ==="
kubectl kustomize kustomize-demo/overlays/prod
```

Notice the differences: dev has 1 replica with `dev-` prefix; prod has 5 replicas with `prod-` prefix and the alpine image tag. Same base, different outcomes.

**Step 5 — Apply the dev overlay:**

```bash
# Create the dev namespace
kubectl create namespace dev

# Apply the dev overlay
kubectl apply -k kustomize-demo/overlays/dev -n dev

# Verify
kubectl get deployment -n dev
kubectl get pods -n dev
kubectl get svc -n dev
```

**Step 6 — Apply the prod overlay:**

```bash
# Create the production namespace
kubectl create namespace production

# Apply the prod overlay
kubectl apply -k kustomize-demo/overlays/prod -n production

# Verify — notice the difference in replicas and image tag
kubectl get deployment -n production
kubectl get pods -n production
kubectl describe deployment prod-my-app -n production | grep Image
```

**Step 7 — Clean up the Kustomize resources:**

```bash
kubectl delete -k kustomize-demo/overlays/dev -n dev
kubectl delete -k kustomize-demo/overlays/prod -n production
kubectl delete namespace dev production
```

### Lab 3: Combining Helm and Kustomize

Sometimes you want to install a Helm chart but apply additional customizations that the chart's values don't expose. The pattern: render the chart with `helm template`, then patch with Kustomize.

**Step 1 — Render a Helm chart to YAML:**

```bash
mkdir -p helm-kustomize-demo/base

helm template my-nginx bitnami/nginx \
  --set service.type=ClusterIP \
  > helm-kustomize-demo/base/nginx.yaml
```

**Step 2 — Create a Kustomize layer on top:**

```bash
cat <<'EOF' > helm-kustomize-demo/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- nginx.yaml
commonAnnotations:
  team: platform-engineering
  managed-by: helm-plus-kustomize
EOF
```

**Step 3 — Preview and apply:**

```bash
# Preview — you'll see all the Helm-generated resources now have our custom annotations
kubectl kustomize helm-kustomize-demo/base | head -30

# Apply
kubectl apply -k helm-kustomize-demo/base

# Verify the annotations are present
kubectl get deployment -o yaml | grep -A2 "annotations"

# Clean up
kubectl delete -k helm-kustomize-demo/base
```

**What just happened:** Helm rendered the nginx chart into plain YAML. Kustomize added annotations that the original chart didn't support. This pattern lets you use Helm for what it's good at (packaging third-party apps) and Kustomize for what it's good at (environment-specific tweaks).

### Clean Up

```bash
# Delete the Kind cluster
kind delete cluster --name packaging-lab

# Remove the demo directories
rm -rf kustomize-demo helm-kustomize-demo
```

---

## Key Takeaways

1. **Managing raw YAML at scale doesn't work.** As your cluster grows, you need templating (Helm), patching (Kustomize), or both to keep manifests maintainable across environments.

2. **Helm is a full package manager.** Charts bundle resources, values parameterize them, and Helm tracks releases with install, upgrade, rollback, and uninstall lifecycle management.

3. **Kustomize patches plain YAML without templates.** The base + overlays model lets you maintain a single set of valid manifests and customize them per environment using patches, prefixes, image overrides, and label injection.

4. **Helm and Kustomize are complementary, not competing.** Use Helm to install third-party charts. Use Kustomize to customize your own manifests. Use both together when you need Helm packaging with Kustomize fine-tuning.

5. **`helm template` is your debugging tool.** It renders charts to plain YAML without installing, letting you inspect exactly what would be applied to the cluster.

6. **Kustomize is built into kubectl.** No separate installation needed — `kubectl apply -k` and `kubectl kustomize` work out of the box.

7. **GitOps is the production deployment model.** Push to Git, let a controller (ArgoCD, Flux) sync the cluster. Helm charts and Kustomize overlays are the building blocks that GitOps controllers consume.

---

## Further Reading

- [Helm — Using Helm](https://helm.sh/docs/intro/using_helm/)
- [Helm Install](https://helm.sh/docs/helm/helm_install/)
- [Helm — Getting Started](https://helm.sh/docs/chart_template_guide/getting_started/)
- [Kustomize — Managing Kubernetes Objects](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [kubectl kustomize](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_kustomize/)
- [Artifact Hub — Find Helm Charts](https://artifacthub.io/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/en/stable/)
- [Flux Documentation](https://fluxcd.io/flux/)

---

**Previous:** [Chapter 9 — Configuration and Secrets](09-configuration-and-secrets.md)
**Next:** [Chapter 11 — Security](11-security.md)
