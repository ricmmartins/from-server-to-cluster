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
  tag: "1.30.6-alpine"
service:
  port: 80
  type: ClusterIP
```

When you run `helm install`, Helm renders the templates by substituting values, producing plain Kubernetes YAML that gets applied to the cluster. You can override any value at install time with `--set` or `--values`.

### Helm Repositories and Artifact Hub

Charts are distributed through repositories — HTTP servers (or OCI registries) hosting packaged charts. You add a repository, update the index, and search for charts just like you would with `apt`:

| APT Command | Helm Equivalent |
|-------------|----------------|
| `add-apt-repository ppa:...` | `helm repo add traefik https://traefik.github.io/charts` |
| `apt update` | `helm repo update` |
| `apt search <package>` | `helm search repo <chart>` |
| `apt show <package>` | `helm show chart <repository>/<chart> --version <chart-version>` |
| `apt install <package>` | `helm install <release> <repository>/<chart> --version <chart-version>` |

[Artifact Hub](https://artifacthub.io/) is the public directory for finding Helm charts — think of it as the Kubernetes equivalent of `packages.ubuntu.com` or `hub.docker.com` for charts. It indexes charts from multiple repositories and provides search, documentation, and version history.

Repository entries are not guarantees that a chart's container images remain maintained or publicly accessible. Inspect both chart and image provenance, pin their versions, and review support policies. The labs below deliberately use a self-contained local chart and an official NGINX image instead of depending on an external application catalog.

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
| `labels` | Add labels; `includeSelectors: true` also updates selectors and Pod templates |
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

A GitOps controller (like [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) or [Flux](https://fluxcd.io/)) runs inside the cluster and reconciles declared sources with cluster state according to its configured policy. Detecting drift does not necessarily mean automatically applying it: sync may be manual, suspended, or subject to approval. For Argo CD, **automated sync** applies new desired revisions, **`selfHeal: true`** enables automated correction of live drift, and **`prune: true`** permits automatic deletion of tracked resources removed from Git. Self-healing and pruning are not unconditional defaults. Review permissions, exclusions, sync windows, and deletion safeguards before enabling them.

Think of it as `ansible-pull` or `puppet agent` for Kubernetes — the system pulls its configuration from a central source instead of having an admin push it.

### Why GitOps Matters

| Benefit | How |
|---------|-----|
| **Audit trail** | Every change is a Git commit with author, timestamp, and message |
| **Reproducibility** | The cluster state is defined in code, not in someone's terminal history |
| **Rollback** | `git revert` changes desired state; an authorized sync applies it (it does not restore database data) |
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
> - **GitOps inverts the deployment model.** Traditional Linux deployment is push-based: you SSH in, run commands, and the system changes. GitOps is pull-based: a controller detects changes and reconciles according to policy. A manual `kubectl edit` may be reported as drift rather than immediately reverted; automatic correction requires the appropriate self-healing/reconciliation policy. Resource deletion likewise depends on pruning policy.

---

## Diagnostic Lab: Packaging Hands-On

### Prerequisites

Use **Helm v3.19.0** for this lab. Download the archive for your operating system and architecture from the [official v3.19.0 release](https://github.com/helm/helm/releases/tag/v3.19.0), verify its published checksum, and extract the binary. Do not use an unpinned “latest” installer.

```bash
# Confirm the pinned lab version
helm version
# Expected version: v3.19.0
```

Use Kind **v0.31.0**, kubectl **v1.35.0**, and the Kubernetes **v1.35.0** teaching snapshot. These `.0` pins are for reproducible labs; production should use the latest supported patch of its chosen minor. Create a dedicated cluster and select its context:

```bash
kind create cluster --name packaging-lab --image kindest/node:v1.35.0@sha256:452d707d4862f52530247495d180205e029056831160e22870e37e3f6c1ac31f
kubectl config use-context kind-packaging-lab
kubectl wait --for=condition=Ready nodes --all --timeout=120s
```

### Lab 1: Helm Basics

**Step 1 — Create a local, versioned chart:**

The chart **`web-demo` version `1.0.0`** is fully defined here, using the maintained official image **`nginx:1.30.6-alpine`** (see the [official image source](https://github.com/nginx/docker-nginx/tree/master/stable/alpine)). No external chart or catalog-specific image is required. This is a local chart, so `helm search repo` is not needed; it demonstrates the same release lifecycle as a downloaded chart.

```bash
mkdir -p web-demo/templates
cat <<'EOF' > web-demo/Chart.yaml
apiVersion: v2
name: web-demo
description: Minimal web server for the packaging lab
type: application
version: 1.0.0
appVersion: "1.30.6"
EOF

cat <<'EOF' > web-demo/values.yaml
replicaCount: 1
image:
  repository: nginx
  tag: "1.30.6-alpine"
service:
  type: ClusterIP
  port: 80
EOF

cat <<'EOF' > web-demo/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-web-demo
  labels:
    app.kubernetes.io/instance: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/instance: {{ .Release.Name }}
      app.kubernetes.io/name: web-demo
  template:
    metadata:
      labels:
        app.kubernetes.io/instance: {{ .Release.Name }}
        app.kubernetes.io/name: web-demo
    spec:
      containers:
      - name: web
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - name: http
          containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: http
        resources:
          requests:
            cpu: 50m
            memory: 32Mi
          limits:
            cpu: 200m
            memory: 128Mi
EOF

cat <<'EOF' > web-demo/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-web-demo
  labels:
    app.kubernetes.io/instance: {{ .Release.Name }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/name: web-demo
  ports:
  - port: {{ .Values.service.port }}
    targetPort: http
EOF
```

**Step 2 — Validate and render before installing:**

```bash
helm lint ./web-demo
helm template my-nginx ./web-demo
```

**Step 3 — Inspect metadata and values:**

```bash
# Show chart metadata
helm show chart ./web-demo

# Show the default values (this is what you can configure)
helm show values ./web-demo
```

**Step 4 — Install the chart:**

```bash
helm install my-nginx ./web-demo --wait --timeout 180s
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
cat <<'EOF' > web-demo-upgrade.yaml
replicaCount: 3
image:
  repository: nginx
  tag: "1.30.6-alpine"
service:
  type: ClusterIP
  port: 80
EOF
helm upgrade my-nginx ./web-demo \
  --values web-demo-upgrade.yaml --wait --timeout 180s

# Check that we now have 3 replicas
kubectl get pods -l app.kubernetes.io/instance=my-nginx

# View release history
helm history my-nginx
```

**Step 7 — Rollback:**

```bash
# Rollback to revision 1 (the original install)
helm rollback my-nginx 1 --wait --timeout 180s

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

The upgrade changes **values**, not the chart version: chart `1.0.0` and the image tag stay pinned, while Helm records a new release revision. `--wait` checks readiness for install, upgrade, and rollback. Readiness failures are failures to investigate, not evidence of a successful deployment.

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
        image: nginx:1.30.6
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
labels:
- pairs:
    environment: dev
  includeSelectors: true
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
labels:
- pairs:
    environment: production
  includeSelectors: true
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
  newTag: "1.30.6-alpine"
EOF
```

**Step 4 — Compare the outputs:**

The `labels` transformer uses `includeSelectors: true` to label resources, Deployment selectors, Pod templates, and Service selectors consistently. This preserves the behavior of the older `commonLabels` field without its deprecation warning. Do not change these selector labels on an existing Deployment; selectors are immutable. See the [official labels transformer reference](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/labels/).

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

helm template my-nginx ./web-demo \
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
kubectl rollout status deployment/my-nginx-web-demo --timeout=180s

# Verify the annotations are present
kubectl get deployment -o yaml | grep -A2 "annotations"

# Clean up
kubectl delete -k helm-kustomize-demo/base
```

**What just happened:** Helm rendered local chart `web-demo` version `1.0.0` into plain YAML. Kustomize added annotations that the chart didn't expose as values. This demonstrates packaging plus patching without an external image catalog. Because the result was applied with kubectl rather than `helm install`, it is not a Helm-managed release: clean it up with `kubectl delete -k`, not `helm uninstall`.

### Clean Up

```bash
# Delete the Kind cluster
kind delete cluster --name packaging-lab

# Remove the demo directories
rm -rf kustomize-demo helm-kustomize-demo web-demo
rm -f web-demo-upgrade.yaml
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
- [Argo CD — automated sync, self-healing, and pruning](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/)
- [Official NGINX container image source](https://github.com/nginx/docker-nginx)
- [Flux Documentation](https://fluxcd.io/flux/)

---

**Previous:** [Chapter 9 — Configuration and Secrets](09-configuration-and-secrets.md)
**Next:** [Chapter 11 — Security](11-security.md)
