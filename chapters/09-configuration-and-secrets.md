# Chapter 9: Configuration and Secrets

*"Never hardcode what might change. And everything changes."*

---

## The /etc/ of Kubernetes

Every Linux admin has a deep, personal relationship with `/etc/`. It's where the personality of a server lives — `nginx.conf`, `sshd_config`, `resolv.conf`, `my.cnf`, the `.env` files your applications read, the shell profiles in `/etc/profile.d/`. Change a file in `/etc/`, reload the service, and behavior changes. The application binary stays the same; only the configuration differs between environments.

Kubernetes takes this idea and elevates it to a first-class pattern. Instead of SSH'ing into a server and editing files, you declare configuration as Kubernetes objects — ConfigMaps and Secrets — and inject them into pods as environment variables or mounted files. The configuration lives in the cluster, versioned and managed alongside your workloads. You never log into a container to edit a config file. You change the ConfigMap, and the cluster propagates the update.

This is the Twelve-Factor App principle in action: "Store config in the environment." Kubernetes doesn't just support this pattern — it enforces it. The same container image runs in dev, staging, and production. Only the injected configuration changes.

For a Linux admin, think of it this way: instead of maintaining different `/etc/nginx/nginx.conf` files on different servers, you maintain different ConfigMaps in different namespaces (or clusters). The nginx image is identical everywhere. The config is external.

---

## ConfigMaps — Cluster-Managed Configuration

A ConfigMap holds non-confidential configuration data as key-value pairs or entire files. It's the Kubernetes equivalent of dropping config files into `/etc/` or setting environment variables in `/etc/profile.d/`.

### Creating ConfigMaps

**From literals (key-value pairs):**

```bash
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=DB_HOST=postgres \
  --from-literal=CACHE_TTL=300
```

**From a file:**

```bash
# Create a config file first
cat <<'EOF' > nginx.conf
server {
    listen 80;
    server_name localhost;
    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
    location /health {
        return 200 'OK';
        add_header Content-Type text/plain;
    }
}
EOF

kubectl create configmap nginx-config --from-file=nginx.conf
```

**From a directory (all files become keys):**

```bash
mkdir config-dir
echo "info" > config-dir/log_level
echo "postgres" > config-dir/db_host
kubectl create configmap app-config-dir --from-file=config-dir/
```

**Declaratively (YAML):**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: info
  DB_HOST: postgres
  CACHE_TTL: "300"
  app.properties: |
    database.host=postgres
    database.port=5432
    log.level=info
```

### Consuming ConfigMaps

There are two main ways to inject ConfigMap data into a pod: as environment variables or as mounted files.

**As environment variables:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "env | sort && sleep 3600"]
    env:
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DB_HOST
```

This is like adding `export LOG_LEVEL=info` to `/etc/profile.d/app.sh`.

**Bulk injection with `envFrom`:**

```yaml
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "env | sort && sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-config
```

This injects *every* key in the ConfigMap as an environment variable. Like `source /etc/profile.d/app.sh` where every line is an export.

**As mounted files (volume):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-demo
spec:
  containers:
  - name: web
    image: nginx:1.27
    volumeMounts:
    - name: config-volume
      mountPath: /etc/nginx/conf.d
  volumes:
  - name: config-volume
    configMap:
      name: nginx-config
```

Each key in the ConfigMap becomes a file in the mount path. If the ConfigMap has a key `nginx.conf`, the pod gets a file at `/etc/nginx/conf.d/nginx.conf`. This is exactly like copying config files into `/etc/` — except the cluster manages them.

### The Update Propagation Caveat

Here's something that trips up everyone coming from Linux:

- **Volume-mounted ConfigMaps auto-update** — When you change a ConfigMap, the files in mounted volumes are eventually refreshed (the kubelet sync period is roughly 60–90 seconds by default). Your application still needs to notice the change (re-read the file, watch with inotify, etc.).
- **Environment variables NEVER auto-update** — If you injected a ConfigMap value as an env var, changing the ConfigMap has zero effect on running pods. Environment variables are set at container start time and are immutable for the life of the container. You must restart (or recreate) the pod to pick up the new values.

On Linux, this is like the difference between an app that reads `/etc/app.conf` on every request (it sees changes immediately) versus an app that reads `$LOG_LEVEL` once at startup (it never sees changes until restarted).

---

## Secrets — Sensitive Configuration

Secrets are structurally identical to ConfigMaps but intended for sensitive data: passwords, API keys, TLS certificates, SSH keys. The API, consumption patterns, and YAML structure are nearly the same.

### Creating Secrets

```bash
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=supersecret
```

For TLS certificates:

```bash
kubectl create secret tls my-tls-secret \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem
```

For Docker registry credentials:

```bash
kubectl create secret docker-registry my-registry \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass
```

### The Base64 Warning

Let's be absolutely clear about this: **base64 encoding is NOT encryption.** It's encoding, like URL encoding. Anyone with access to the Secret object can decode it trivially:

```bash
# Create a secret
kubectl create secret generic db-creds --from-literal=password=supersecret

# View the raw YAML
kubectl get secret db-creds -o yaml
```

You'll see something like:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
data:
  password: c3VwZXJzZWNyZXQ=
```

That `c3VwZXJzZWNyZXQ=` is just base64:

```bash
echo 'c3VwZXJzZWNyZXQ=' | base64 --decode
# Output: supersecret
```

Anyone who can `kubectl get secret` can read your passwords. Secrets provide a *separation of concerns* (config vs. sensitive config), not *security*. For actual security, you need:

- **Encryption at rest** — Encrypt etcd storage so Secrets aren't stored in plain text
- **RBAC** — Restrict who can read Secret objects
- **External secret managers** — Tools like HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault, integrated via projects like External Secrets Operator or Sealed Secrets

### Consuming Secrets

Secrets are consumed exactly like ConfigMaps — as env vars or volume mounts:

**As environment variables:**

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-creds
      key: password
```

**As mounted files:**

```yaml
volumeMounts:
- name: secret-volume
  mountPath: /etc/db-creds
  readOnly: true
volumes:
- name: secret-volume
  secret:
    secretName: db-creds
```

When mounted as files, each key becomes a file. The `password` key becomes `/etc/db-creds/password`. This is like putting credentials in a file with restricted permissions — except in Kubernetes, the access control is at the RBAC level, not the filesystem level.

### Secret Types

| Type | Purpose |
|------|---------|
| `Opaque` | Generic key-value data (default) |
| `kubernetes.io/tls` | TLS certificates (must contain `tls.crt` and `tls.key`) |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/basic-auth` | Basic authentication (username and password) |
| `kubernetes.io/ssh-auth` | SSH authentication (ssh-privatekey) |
| `kubernetes.io/service-account-token` | Service account tokens (auto-created) |

---

## Environment Variables — Direct Injection

Besides ConfigMaps and Secrets, you can set environment variables directly in the pod spec:

```yaml
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: APP_MODE
      value: "production"
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: password
```

You can mix direct values, ConfigMap references, and Secret references in the same `env:` block. This gives you maximum flexibility — hardcode what's truly static, reference what's shared, and protect what's sensitive.

---

## The Downward API — Pod Metadata as Configuration

Sometimes your application needs to know about *itself* — its pod name, namespace, IP address, or resource limits. The Downward API exposes this metadata as environment variables or files, without the application needing to call the Kubernetes API.

Think of it as `/proc/self/status` for pods. On Linux, a process can read `/proc/self/status` to learn its PID, memory usage, and capabilities. In Kubernetes, a pod can use the Downward API to learn its name, namespace, labels, and resource allocations.

**As environment variables:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-demo
  labels:
    app: demo
    version: v2
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "env | grep -E 'POD_|NODE_' && sleep 3600"]
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: POD_MEMORY_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: limits.memory
```

Available `fieldRef` fields include:
- `metadata.name` — pod name
- `metadata.namespace` — pod namespace
- `metadata.uid` — pod UID
- `metadata.labels['<KEY>']` — a specific label value
- `metadata.annotations['<KEY>']` — a specific annotation value
- `spec.nodeName` — the node the pod is running on
- `spec.serviceAccountName` — the service account
- `status.podIP` — the pod's IP address

This is incredibly useful for logging (include the pod name in every log line), metrics (tag metrics with namespace and node), and service discovery (know your own IP).

---

## Immutable ConfigMaps and Secrets

Starting with Kubernetes v1.21 (stable), you can mark ConfigMaps and Secrets as immutable:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: static-config
immutable: true
data:
  VERSION: "2.5.0"
  FEATURE_FLAGS: "dark-mode,new-checkout"
```

Once set to `immutable: true`, the data cannot be changed. Any attempt to update it will be rejected by the API server. You must delete and recreate the object to change it.

Why would you want this?

1. **Performance** — The kubelet stops watching immutable ConfigMaps/Secrets for changes, reducing API server load. In clusters with thousands of pods, this matters.
2. **Safety** — Prevents accidental modifications to configuration that shouldn't change during an application's lifetime.

Think of it as mounting a filesystem read-only (`mount -o ro`). You can read it, but you can't accidentally (or intentionally) modify it while it's in use.

---

## Linux ↔ Kubernetes Comparison

| Linux Concept | K8s Equivalent | Notes |
|---------------|----------------|-------|
| `/etc/nginx/nginx.conf` | ConfigMap mounted as file | Cluster-managed configuration |
| `export ENV_VAR=value` | `env:` in pod spec | Environment variable injection |
| `/etc/shadow` (restricted file) | Secret | Sensitive data (but still needs encryption!) |
| `/proc/self/status` | Downward API | Pod metadata exposed to the container |
| `source /etc/profile.d/*.sh` | `envFrom: configMapRef` | Bulk environment injection |
| inotify (file change watch) | ConfigMap volume updates | Auto-refresh when config changes (~60-90s) |
| `mount -o ro` (read-only mount) | `immutable: true` | Performance optimization, prevents changes |
| `chmod 600 /etc/shadow` | RBAC on Secret objects | Access control (namespace-level, not per-key) |

---

> ### 🔀 Where the Linux Analogy Breaks
>
> - **Config file updates don't trigger service reloads.** In Linux, you edit `/etc/nginx/nginx.conf` and run `systemctl reload nginx`. In Kubernetes, you update the ConfigMap, and... the volume mount eventually refreshes (60–90 seconds), but the application may not notice. There's no built-in `reload` signal. If you used env vars instead of a volume mount, the values never update at all — you need a pod restart. Some projects work around this with sidecar reloaders or hash-based rollout triggers, but it's not a native feature.
>
> - **Secrets are NOT secure by default.** They're base64-encoded (trivially reversible) and stored unencrypted in etcd unless you explicitly enable encryption at rest. Anyone with `kubectl get secret` access can read your passwords. Treat Secrets as "slightly better than ConfigMaps" for separation of concerns — real security requires external secret managers and proper RBAC policies.
>
> - **There's no per-key access control.** On Linux, you can `chmod 600 /etc/shadow` and `chmod 644 /etc/passwd` — file-level permissions. In Kubernetes, access control is at the namespace/RBAC level. If someone can read Secrets in a namespace, they can read *all* Secrets in that namespace. There's no equivalent of setting different permissions on individual keys within a ConfigMap or Secret.

---

## Diagnostic Lab: Configuration Hands-On

### Prerequisites

```bash
kind create cluster --name config-lab
```

### Lab 1: Create and Consume a ConfigMap

**Step 1 — Create a ConfigMap from literals:**

```bash
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=DB_HOST=postgres \
  --from-literal=CACHE_TTL=300
```

**Step 2 — Create a ConfigMap from a file:**

```bash
cat <<'EOF' > nginx.conf
server {
    listen 80;
    server_name localhost;
    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
    location /health {
        return 200 'healthy';
        add_header Content-Type text/plain;
    }
}
EOF

kubectl create configmap nginx-config --from-file=nginx.conf
```

**Step 3 — Inspect the ConfigMaps:**

```bash
kubectl get configmap app-config -o yaml
kubectl describe configmap nginx-config
```

**Step 4 — Deploy a pod consuming ConfigMap as environment variables:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: env-consumer
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo LOG_LEVEL=$LOG_LEVEL DB_HOST=$DB_HOST && sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-config
EOF

kubectl wait --for=condition=Ready pod/env-consumer --timeout=60s

# Verify the environment variables
kubectl exec env-consumer -- env | grep -E 'LOG_LEVEL|DB_HOST|CACHE_TTL'
```

**Step 5 — Deploy a pod consuming ConfigMap as a mounted file:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: volume-consumer
spec:
  containers:
  - name: web
    image: nginx:1.27
    volumeMounts:
    - name: nginx-conf
      mountPath: /etc/nginx/conf.d
  volumes:
  - name: nginx-conf
    configMap:
      name: nginx-config
EOF

kubectl wait --for=condition=Ready pod/volume-consumer --timeout=60s

# Verify the config file is mounted
kubectl exec volume-consumer -- cat /etc/nginx/conf.d/nginx.conf

# Test that nginx uses our custom config (the /health endpoint)
kubectl exec volume-consumer -- curl -s http://localhost/health
```

### Lab 2: Create and Consume a Secret

**Step 1 — Create a Secret:**

```bash
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=supersecret
```

**Step 2 — Prove that base64 is not encryption:**

```bash
# View the raw secret
kubectl get secret db-creds -o yaml

# Decode the password (this is why base64 != encryption)
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 --decode
echo  # newline for readability
```

**Step 3 — Deploy a pod consuming the Secret as a volume mount:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-consumer
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo 'Username:' $(cat /etc/db-creds/username) && echo 'Password:' $(cat /etc/db-creds/password) && sleep 3600"]
    volumeMounts:
    - name: db-creds
      mountPath: /etc/db-creds
      readOnly: true
  volumes:
  - name: db-creds
    secret:
      secretName: db-creds
EOF

kubectl wait --for=condition=Ready pod/secret-consumer --timeout=60s

# Each key is a file in the mount path
kubectl exec secret-consumer -- ls /etc/db-creds/
kubectl exec secret-consumer -- cat /etc/db-creds/password
```

**Step 4 — Consume the Secret as an environment variable:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-consumer
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo DB_PASSWORD=$DB_PASSWORD && sleep 3600"]
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: password
EOF

kubectl wait --for=condition=Ready pod/secret-env-consumer --timeout=60s
kubectl exec secret-env-consumer -- env | grep DB_PASSWORD
```

### Lab 3: Downward API

Deploy a pod that exposes its own metadata as environment variables:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: downward-demo
  labels:
    app: demo
    version: v2
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "env | grep -E 'POD_|NODE_' && sleep 3600"]
    resources:
      requests:
        memory: "32Mi"
        cpu: "100m"
      limits:
        memory: "64Mi"
        cpu: "200m"
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: POD_MEMORY_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: limits.memory
EOF

kubectl wait --for=condition=Ready pod/downward-demo --timeout=60s

# The pod knows about itself without calling the Kubernetes API
kubectl exec downward-demo -- env | grep -E 'POD_|NODE_'
```

You should see output like:

```
POD_NAME=downward-demo
POD_NAMESPACE=default
POD_IP=10.244.0.5
NODE_NAME=config-lab-control-plane
POD_MEMORY_LIMIT=67108864
```

The pod knows its own name, namespace, IP, the node it's running on, and its memory limit — all without any Kubernetes API calls.

### Lab 4: Update a ConfigMap and See Propagation

**Step 1 — Verify current state:**

```bash
# The volume-consumer pod from Lab 1 should still be running
kubectl exec volume-consumer -- cat /etc/nginx/conf.d/nginx.conf | head -3
```

**Step 2 — Update the ConfigMap:**

```bash
cat <<'EOF' > nginx-updated.conf
server {
    listen 80;
    server_name localhost;
    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
    location /health {
        return 200 'healthy-v2';
        add_header Content-Type text/plain;
    }
    location /ready {
        return 200 'ready';
        add_header Content-Type text/plain;
    }
}
EOF

kubectl create configmap nginx-config --from-file=nginx.conf=nginx-updated.conf \
  --dry-run=client -o yaml | kubectl apply -f -
```

**Step 3 — Wait for propagation and verify:**

```bash
# Volume mounts update within roughly 60-90 seconds
echo "Waiting for ConfigMap propagation (up to 90 seconds)..."
sleep 90

# Check if the file was updated (look for the /ready location)
kubectl exec volume-consumer -- cat /etc/nginx/conf.d/nginx.conf
```

You should see the updated config with the new `/ready` location block.

**Important:** The file on disk was updated, but nginx doesn't know about it yet. You'd need to `nginx -s reload` inside the container (or restart the pod) for nginx to pick up the change. Kubernetes updates the file; your application is responsible for noticing.

**Step 4 — Show that env vars do NOT update:**

```bash
# env-consumer still has the original values
kubectl exec env-consumer -- env | grep LOG_LEVEL

# Even if we update the ConfigMap...
kubectl patch configmap app-config -p '{"data":{"LOG_LEVEL":"debug"}}'

# ...the env var in the running pod is unchanged
sleep 5
kubectl exec env-consumer -- env | grep LOG_LEVEL
# Still shows "info", not "debug"

# Only a pod restart picks up the new env var value
kubectl delete pod env-consumer
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: env-consumer
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo LOG_LEVEL=$LOG_LEVEL && sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-config
EOF

kubectl wait --for=condition=Ready pod/env-consumer --timeout=60s
kubectl exec env-consumer -- env | grep LOG_LEVEL
# Now shows "debug"
```

### Clean Up

```bash
# Delete the Kind cluster
kind delete cluster --name config-lab

# Remove temp files
rm -f nginx.conf nginx-updated.conf
```

---

## Key Takeaways

1. **ConfigMaps externalize application configuration.** They hold non-sensitive key-value data or entire configuration files, injected into pods as environment variables or mounted volumes.

2. **Secrets are for sensitive data, but base64 is not encryption.** Anyone who can `kubectl get secret` can decode your passwords. Use RBAC, encryption at rest, and external secret managers for real security.

3. **Volume-mounted ConfigMaps auto-update; env vars never do.** If you need live configuration updates without pod restarts, use volume mounts and have your application watch for file changes.

4. **The Downward API lets pods know about themselves.** Pod name, namespace, IP, node, labels, and resource limits can all be injected as env vars or files — no Kubernetes API calls needed.

5. **`envFrom` is bulk injection; `env` is selective injection.** Use `envFrom` to inject all keys from a ConfigMap or Secret. Use `env` with `valueFrom` to pick specific keys or rename them.

6. **Immutable ConfigMaps and Secrets improve performance.** Mark them as `immutable: true` to stop the kubelet from watching for changes — essential in clusters with thousands of mounted ConfigMaps.

7. **Separate config from code, always.** The same container image should run identically in dev, staging, and production. Only the injected ConfigMaps, Secrets, and environment variables should differ.

---

## Further Reading

- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Downward API](https://kubernetes.io/docs/concepts/workloads/pods/downward-api/)
- [Immutable Secrets and ConfigMaps](https://kubernetes.io/docs/concepts/configuration/secret/#secret-immutable)
- [Managing Secrets Using kubectl](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/)
- [Configure a Pod to Use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
- [Expose Pod Information to Containers Through Environment Variables](https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/)

---

**Previous:** [Chapter 8 — Storage and Persistence](08-storage-and-persistence.md)
**Next:** [Chapter 10 — Packaging and Delivery](10-packaging-and-delivery.md)
