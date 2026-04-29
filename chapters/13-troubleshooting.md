# Chapter 13: Troubleshooting

*"The first rule of debugging: don't guess. Observe, hypothesize, test."*

---

## From `/var/log` to `kubectl describe`

On a Linux server, when something breaks, your muscle memory takes over. You check the process: `systemctl status nginx`. You read the logs: `journalctl -u nginx -f`. You check the network: `ss -tlnp`, `curl localhost:80`. You look at resources: `top`, `df -h`. You know where everything lives, because everything lives on the one machine in front of you.

In Kubernetes, the same instincts apply — but the execution is different. The process you want to inspect might be running on any of a dozen nodes. The logs might vanish the moment the pod restarts. The network isn't just local sockets — it's a cluster-wide overlay managed by CNI plugins and kube-proxy rules. And the "machine" you're debugging on is the API server, accessed remotely through `kubectl`.

Here's the good news: the mental model is the same. Something is broken. You need to find out what, where, and why. The difference is that your toolbox changes from `systemctl`, `journalctl`, and `ss` to `kubectl get`, `kubectl describe`, and `kubectl logs`. And the debugging hierarchy changes from "check the machine" to "check the pod, then the container, then the node, then the cluster."

This chapter teaches you a systematic approach to Kubernetes troubleshooting — the same way experienced Linux sysadmins debug servers, translated to the orchestrated world.

---

## The Debugging Hierarchy

When something goes wrong in Kubernetes, work your way outward:

```
Pod → Container → Node → Cluster
```

1. **Pod level:** Is the pod even running? What state is it in? What do the events say?
2. **Container level:** Did the container start? What do the logs say? Is it crashing?
3. **Node level:** Is the node healthy? Does it have enough resources? Is the kubelet running?
4. **Cluster level:** Is the control plane operational? Are there cluster-wide resource constraints?

Most issues (90%+) are caught at the Pod and Container levels. Start there.

---

## Essential Debugging Commands

### `kubectl get pods` — Status at a Glance

```bash
kubectl get pods -o wide
```

The STATUS column is your first indicator:

| Status | What It Means |
|--------|--------------|
| **Running** | Container(s) started and not (yet) terminated |
| **Pending** | Pod accepted but not scheduled or image not pulled |
| **CrashLoopBackOff** | Container crashes repeatedly; K8s backs off restarts |
| **ImagePullBackOff** | Can't pull the container image |
| **OOMKilled** | Container exceeded its memory limit |
| **CreateContainerConfigError** | Bad ConfigMap/Secret reference |
| **Evicted** | Node under resource pressure, pod was kicked |
| **Terminating** | Pod is shutting down (stuck here = finalizer issue) |
| **Init:Error** | Init container failed |
| **Completed** | Pod ran to completion (normal for Jobs) |

Pay attention to the **RESTARTS** column. A pod showing `Running` with 47 restarts is *not* healthy.

### `kubectl describe pod` — The Most Useful Command

```bash
kubectl describe pod <pod-name>
```

This single command shows you:
- **Status and Conditions** — why the pod is in its current state
- **Events** — the chronological story of what happened (scheduling, pulling, creating, starting, killing)
- **Container state** — current state, last state, exit codes, restart count
- **Resource requests/limits** — what the pod asked for
- **Volumes and mounts** — what's attached and where
- **Node assignment** — where it landed (or why it didn't)

> **Pro tip:** Always read the **Events** section at the bottom. It tells the story.

### `kubectl logs` — Container Output

```bash
# Current container logs
kubectl logs <pod-name>

# Specific container in multi-container pod
kubectl logs <pod-name> -c <container-name>

# Previous crashed container (the one BEFORE the current restart)
kubectl logs <pod-name> -p

# Follow logs in real-time
kubectl logs <pod-name> -f

# Last 50 lines
kubectl logs <pod-name> --tail=50

# Logs since a specific time
kubectl logs <pod-name> --since=5m
```

The `-p` (previous) flag is critical. When a pod is in CrashLoopBackOff, the current container might have zero logs (it just started). The *previous* container has the crash output.

### `kubectl get events` — Cluster Timeline

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

Events are the cluster's activity log. They show scheduling decisions, image pulls, container starts, health check failures, scaling actions, and errors — all timestamped and sortable.

```bash
# Events in a specific namespace
kubectl get events -n my-namespace --sort-by=.metadata.creationTimestamp

# Watch events in real-time
kubectl get events --watch
```

### `kubectl debug` — Ephemeral Debug Containers

```bash
# Attach a debug container to a running pod
kubectl debug <pod-name> -it --image=busybox:1.36 --target=<container-name>

# Create a copy of a pod with a debug container
kubectl debug <pod-name> -it --image=busybox:1.36 --copy-to=debug-pod

# Debug a node directly
kubectl debug node/<node-name> -it --image=busybox:1.36
```

Ephemeral containers are injected into an existing pod without restarting it. They share the pod's network namespace, so you can inspect network connectivity, check filesystem mounts, and run diagnostic tools.

### `kubectl exec` — Interactive Shell

```bash
# Run a shell in a running container
kubectl exec -it <pod-name> -- /bin/sh

# Specific container
kubectl exec -it <pod-name> -c <container-name> -- /bin/bash

# Run a single command
kubectl exec <pod-name> -- cat /etc/resolv.conf
```

### `kubectl port-forward` — Test Connectivity

```bash
# Forward local port to pod
kubectl port-forward pod/<pod-name> 8080:80

# Forward to a service
kubectl port-forward svc/<service-name> 8080:80
```

This bypasses Services and Ingress entirely — useful for confirming the application itself works before investigating network layers.

---

## Common Pod Failure States — Deep Dive

### Pending — Can't Be Scheduled

The scheduler can't find a suitable node. Common causes:

- **Insufficient resources:** No node has enough CPU/memory to satisfy requests
- **NodeSelector/affinity mismatch:** Pod requires a label no node has
- **Taints without tolerations:** Nodes are tainted, pod doesn't tolerate
- **PVC not bound:** Pod requests a volume that doesn't exist or can't be provisioned
- **Pod topology constraints:** Can't satisfy spread requirements

**Diagnosis:**
```bash
kubectl describe pod <pending-pod> | grep -A 20 "Events"
# Look for: "FailedScheduling" with the reason
```

### ImagePullBackOff — Can't Get the Image

The container runtime can't pull the specified image. Common causes:

- **Typo in image name:** `ngnix` instead of `nginx`
- **Tag doesn't exist:** `nginx:1.99` (no such version)
- **Private registry:** Missing `imagePullSecrets` on the pod
- **Rate limiting:** Docker Hub rate limits exceeded
- **Registry down:** The registry server is unreachable

**Diagnosis:**
```bash
kubectl describe pod <pod> | grep -A 5 "Events"
# Look for: "Failed to pull image" with the specific error
```

### CrashLoopBackOff — Crashes Repeatedly

The container starts, crashes, and K8s restarts it with exponential backoff (10s, 20s, 40s, ... up to 5 minutes).

- **Application bug:** Code exits with non-zero immediately
- **Missing dependency:** Required env var, config file, or service connection
- **Bad entrypoint:** Wrong command in Dockerfile or pod spec
- **Failing liveness probe:** Probe kills the container before it's ready

**Diagnosis:**
```bash
# Check previous container's logs
kubectl logs <pod> -p

# Check the exit code
kubectl describe pod <pod> | grep -A 5 "Last State"
```

Common exit codes:
- **Exit 1** — general application error
- **Exit 137** — killed by SIGKILL (OOMKilled or liveness probe)
- **Exit 139** — segfault (SIGSEGV)
- **Exit 143** — SIGTERM (graceful shutdown)

### OOMKilled — Out of Memory

The container exceeded its memory limit and was killed by the Linux OOM killer (via cgroups).

**Diagnosis:**
```bash
kubectl describe pod <pod> | grep -i "OOMKilled"
kubectl describe pod <pod> | grep -A 3 "Last State"
```

**Fix:** Increase the memory limit, or fix the memory leak in the application.

### CreateContainerConfigError — Bad Config Reference

The pod spec references a ConfigMap or Secret that doesn't exist.

**Diagnosis:**
```bash
kubectl describe pod <pod> | grep -A 5 "Events"
# "Error: configmap \"my-config\" not found"
```

### Evicted — Node Pressure

The kubelet evicts pods when the node runs low on disk, memory, or PIDs.

**Diagnosis:**
```bash
kubectl get pods | grep Evicted
kubectl describe pod <evicted-pod> | grep -i "evict"
kubectl describe node <node> | grep -A 10 "Conditions"
```

---

## Linux ↔ Kubernetes Debugging Comparison

| Linux Debugging | K8s Equivalent | What It Shows |
|----------------|---------------|---------------|
| `systemctl status svc` | `kubectl get pods` | Process/pod state |
| `journalctl -u svc` | `kubectl logs pod` | Application output |
| `journalctl -xe` | `kubectl get events` | System-level events timeline |
| `ps aux`, `top` | `kubectl top pods` | Resource usage |
| `strace -p PID` | `kubectl debug` (ephemeral) | Deep container inspection |
| `ss -tlnp` | `kubectl get svc,endpoints` | Network listeners and backends |
| `ping`, `curl`, `traceroute` | `kubectl exec -- wget/curl` | Network connectivity testing |
| `dmesg`, `/var/log/syslog` | `kubectl describe node` | Node-level issues (OOM, disk) |
| `df -h` | `kubectl describe node` Conditions | Disk pressure detection |

---

> ### ⚠️ Where the Linux Analogy Breaks
>
> **Remote debugging vs. local access:** On Linux, you debug on the machine itself — SSH in, run commands, read files. In Kubernetes, the "machine" is abstracted away. You debug through the API server using `kubectl`. If the API server is down, you're locked out entirely (unlike Linux where you can still use the local console or IPMI). Your debugging toolchain depends on the cluster being at least partially functional.
>
> **Automatic restarts mask failures:** Linux processes crash and stay dead until you restart them. Kubernetes pods crash and get automatically restarted — that's CrashLoopBackOff. This can mask issues: the pod status shows "Running" but the RESTARTS column shows 47. Always check RESTARTS. A pod that's been running for 2 minutes with 50 restarts is not "working."
>
> **Ephemeral logs:** Logs in Linux persist in `/var/log` — you can always go back and read them. Pod logs vanish when the pod is deleted or replaced. If you need to investigate a crash from yesterday, those logs are gone unless you shipped them externally (Fluent Bit, Loki, etc.) BEFORE the pod died. In production, centralized logging isn't optional — it's a prerequisite for debugging.

---

## Diagnostic Lab: 5 Real-World Troubleshooting Scenarios

### Prerequisites

```bash
# Create a Kind cluster for troubleshooting practice
kind create cluster --name troubleshooting-lab --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

# Verify cluster is ready
kubectl get nodes
kubectl cluster-info
```

---

### Scenario 1: Pod Stuck in Pending

**Deploy the broken state:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: greedy-app
  labels:
    app: greedy-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: greedy-app
  template:
    metadata:
      labels:
        app: greedy-app
    spec:
      containers:
      - name: greedy
        image: nginx:1.27
        resources:
          requests:
            cpu: "100"
            memory: "256Gi"
EOF
```

This pod requests 100 CPUs and 256 GiB of memory — far more than any node in a Kind cluster can provide.

**Diagnose step by step:**

```bash
# Step 1: Check pod status
kubectl get pods
# STATUS: Pending

# Step 2: Describe the pod — read the Events section
kubectl describe pod -l app=greedy-app
# Events will show:
#   Warning  FailedScheduling  ... 0/3 nodes are available:
#   3 Insufficient cpu, 3 Insufficient memory.

# Step 3: Check available node resources
kubectl describe nodes | grep -A 5 "Allocatable"
```

**Apply the fix:**

```bash
# Fix: set reasonable resource requests
kubectl patch deployment greedy-app --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/resources/requests/cpu", "value": "100m"},
  {"op": "replace", "path": "/spec/template/spec/containers/0/resources/requests/memory", "value": "128Mi"}
]'
```

**Verify resolution:**

```bash
kubectl get pods -l app=greedy-app -w
# Wait until STATUS shows Running
```

**Clean up:**

```bash
kubectl delete deployment greedy-app
```

---

### Scenario 2: CrashLoopBackOff

**Deploy the broken state:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crash-app
  labels:
    app: crash-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crash-app
  template:
    metadata:
      labels:
        app: crash-app
    spec:
      containers:
      - name: crash
        image: busybox:1.36
        command: ["/bin/sh", "-c", "echo 'Starting app...' && sleep 2 && echo 'FATAL: Missing DATABASE_URL' && exit 1"]
        env:
        - name: APP_NAME
          value: "my-service"
EOF
```

This container starts, prints an error about a missing database URL, and exits with code 1.

**Diagnose step by step:**

```bash
# Step 1: Check pod status (wait ~30 seconds for backoff to show)
kubectl get pods -l app=crash-app
# STATUS: CrashLoopBackOff, RESTARTS: 2+

# Step 2: Check the logs of the PREVIOUS crashed container
kubectl logs -l app=crash-app -p
# Output: "Starting app..."
#         "FATAL: Missing DATABASE_URL"

# Step 3: Check current logs (might be empty if container just restarted)
kubectl logs -l app=crash-app

# Step 4: Describe to see exit code and state
kubectl describe pod -l app=crash-app | grep -A 10 "Last State"
# Last State: Terminated
#   Reason: Error
#   Exit Code: 1
```

**Apply the fix:**

```bash
# Fix: provide the missing environment variable
kubectl set env deployment/crash-app DATABASE_URL="postgres://db:5432/myapp"
```

Wait — this pod will still crash because the busybox command is hardcoded. Let's fix the command too:

```bash
kubectl patch deployment crash-app --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/command", "value": ["/bin/sh", "-c", "echo Starting app... && echo Connected to $DATABASE_URL && sleep 3600"]}
]'
```

**Verify resolution:**

```bash
kubectl get pods -l app=crash-app -w
# Wait for STATUS: Running, RESTARTS: 0

kubectl logs -l app=crash-app
# "Starting app..."
# "Connected to postgres://db:5432/myapp"
```

**Clean up:**

```bash
kubectl delete deployment crash-app
```

---

### Scenario 3: Service Not Reachable

**Deploy the broken state:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
        version: v2
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web-app-typo
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF
```

The Service selector says `app: web-app-typo` but the pods have `app: web-app`.

**Diagnose step by step:**

```bash
# Step 1: Check pods are running
kubectl get pods -l app=web-app
# STATUS: Running (pods are fine!)

# Step 2: Check the service
kubectl get svc web-service
# ClusterIP assigned — looks normal

# Step 3: Check endpoints (THE KEY STEP)
kubectl get endpoints web-service
# ENDPOINTS: <none>  ← This is the problem!

# Step 4: Compare labels
kubectl get pods --show-labels | grep web-app
# Labels: app=web-app,version=v2

kubectl describe svc web-service | grep Selector
# Selector: app=web-app-typo  ← Doesn't match!

# Step 5: Try to reach the service (will fail)
kubectl run debug-curl --image=busybox:1.36 --rm -it --restart=Never -- wget -qO- --timeout=3 http://web-service 2>&1 || true
# wget: download timed out
```

**Apply the fix:**

```bash
# Fix: correct the selector
kubectl patch svc web-service --type='json' -p='[
  {"op": "replace", "path": "/spec/selector", "value": {"app": "web-app"}}
]'
```

**Verify resolution:**

```bash
# Endpoints should now show pod IPs
kubectl get endpoints web-service
# ENDPOINTS: 10.244.x.x:80,10.244.y.y:80

# Test connectivity
kubectl run debug-curl --image=busybox:1.36 --rm -it --restart=Never -- wget -qO- --timeout=3 http://web-service
# Returns nginx welcome page HTML
```

**Clean up:**

```bash
kubectl delete deployment web-app
kubectl delete svc web-service
```

---

### Scenario 4: ImagePullBackOff

**Deploy the broken state:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bad-image-app
  labels:
    app: bad-image-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bad-image-app
  template:
    metadata:
      labels:
        app: bad-image-app
    spec:
      containers:
      - name: app
        image: nginx:9.99.99-nonexistent
        ports:
        - containerPort: 80
EOF
```

The image tag `9.99.99-nonexistent` doesn't exist on Docker Hub.

**Diagnose step by step:**

```bash
# Step 1: Check pod status
kubectl get pods -l app=bad-image-app
# STATUS: ErrImagePull or ImagePullBackOff

# Step 2: Describe the pod
kubectl describe pod -l app=bad-image-app | grep -A 10 "Events"
# Events:
#   Warning  Failed   ... Failed to pull image "nginx:9.99.99-nonexistent":
#     rpc error: ... manifest unknown

# Step 3: Verify the image exists (outside the cluster)
# You would check Docker Hub or your registry to confirm valid tags
```

**Apply the fix:**

```bash
# Fix: use a valid image tag
kubectl set image deployment/bad-image-app app=nginx:1.27
```

**Verify resolution:**

```bash
kubectl get pods -l app=bad-image-app -w
# Wait for STATUS: Running

kubectl describe pod -l app=bad-image-app | grep "Image:"
# Image: nginx:1.27
```

**Clean up:**

```bash
kubectl delete deployment bad-image-app
```

---

### Scenario 5: Node NotReady

In a Kind cluster, we can simulate a node issue by stopping the container running the worker node.

**Deploy workloads first:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resilient-app
  labels:
    app: resilient-app
spec:
  replicas: 4
  selector:
    matchLabels:
      app: resilient-app
  template:
    metadata:
      labels:
        app: resilient-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
EOF

# Wait for all pods to be running
kubectl wait --for=condition=Ready pod -l app=resilient-app --timeout=60s
kubectl get pods -l app=resilient-app -o wide
```

**Simulate the failure:**

```bash
# Get the worker node names
kubectl get nodes

# Stop one worker node (Kind nodes are Docker containers)
docker pause troubleshooting-lab-worker

# Wait 40-60 seconds for the node to be marked NotReady
sleep 60
```

**Diagnose step by step:**

```bash
# Step 1: Check node status
kubectl get nodes
# One node shows NotReady

# Step 2: Describe the NotReady node
kubectl describe node troubleshooting-lab-worker | grep -A 10 "Conditions"
# Ready: False — Kubelet stopped posting status

# Step 3: Check pod distribution
kubectl get pods -l app=resilient-app -o wide
# Pods on the NotReady node will eventually be rescheduled (after the
# pod-eviction-timeout, typically 5 minutes)
```

**Apply the fix:**

```bash
# Fix: "repair" the node by unpausing it
docker unpause troubleshooting-lab-worker

# Wait for node to become Ready
kubectl wait --for=condition=Ready node/troubleshooting-lab-worker --timeout=120s
kubectl get nodes
```

**Verify resolution:**

```bash
kubectl get nodes
# All nodes show Ready

kubectl get pods -l app=resilient-app -o wide
# All pods running and distributed across nodes
```

**Clean up:**

```bash
kubectl delete deployment resilient-app
```

---

### The Debugging Toolkit — Summary

When you're stuck, these are your go-to tools:

```bash
# Run a temporary debug pod with common networking tools
kubectl run debug --image=busybox:1.36 -it --rm --restart=Never -- /bin/sh

# Attach an ephemeral container to a running pod
kubectl debug <pod-name> -it --image=busybox:1.36 --target=<container-name>

# Debug directly on a node
kubectl debug node/<node-name> -it --image=busybox:1.36

# Get the full YAML of a pod (see computed fields, defaulted values)
kubectl get pod <pod-name> -o yaml

# Check resource usage (requires metrics-server)
kubectl top pods
kubectl top nodes
```

### Lab Cleanup

```bash
# Delete the Kind cluster
kind delete cluster --name troubleshooting-lab
```

---

## Key Takeaways

1. **Always start with `kubectl describe`.** The Events section at the bottom tells the chronological story of what happened to the pod. It's the single most informative debugging command.

2. **Use `kubectl logs -p` for crashed containers.** The current container might have no logs yet (it just restarted). The previous container has the crash output you need.

3. **Empty endpoints = selector mismatch.** When a Service can't reach pods, check `kubectl get endpoints <svc>`. If it's empty, the Service selector doesn't match any pod labels.

4. **Pod status tells you where to look.** Pending = scheduling problem. ImagePullBackOff = image/registry problem. CrashLoopBackOff = application problem. OOMKilled = memory problem. Each status points to a different investigation path.

5. **Check RESTARTS, not just STATUS.** A pod showing "Running" with 100+ restarts is not healthy — it's CrashLoopBackOff that happens to be in the "running" phase of its crash loop when you looked.

6. **Events are timestamped and sorted.** Use `kubectl get events --sort-by=.metadata.creationTimestamp` for a timeline view. This reveals cascading failures — one event triggering another.

7. **When in doubt, run a debug pod.** `kubectl run debug --image=busybox:1.36 -it --rm` gives you a shell inside the cluster network. From there you can `wget`, `nslookup`, and test connectivity that's invisible from outside.

---

## Further Reading

- [Troubleshooting Applications](https://kubernetes.io/docs/tasks/debug/debug-application/)
- [Debug Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
- [Debug Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- [kubectl debug](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_debug/)
- [Pod Lifecycle — Pod Phase](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase)
- [Resource Management for Pods](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

---

**Previous:** [Chapter 12 — Scaling and Observability](12-scaling-and-observability.md)
**Next:** [Chapter 14 — Production Operations](14-production-operations.md)
