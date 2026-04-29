# Appendix B: kubectl Cheatsheet

*Essential commands and YAML patterns for daily Kubernetes work.*

---

## Setup and Configuration

```bash
# View and manage kubeconfig
kubectl config view                                    # Show merged kubeconfig
kubectl config get-contexts                            # List all contexts
kubectl config current-context                         # Show active context
kubectl config use-context <context-name>              # Switch to a different context
kubectl config set-context --current --namespace=dev   # Set default namespace for current context

# Get cluster credentials (managed Kubernetes)
az aks get-credentials --resource-group myRG --name myCluster       # AKS
aws eks update-kubeconfig --name myCluster --region us-west-2       # EKS
gcloud container clusters get-credentials myCluster --region us-c1  # GKE

# Enable shell autocompletion
source <(kubectl completion bash)                      # Bash (current session)
echo 'source <(kubectl completion bash)' >> ~/.bashrc  # Bash (permanent)
source <(kubectl completion zsh)                       # Zsh
```

## Getting Information

```bash
# Cluster overview
kubectl cluster-info                                   # Control plane and CoreDNS endpoints
kubectl get nodes -o wide                              # List nodes with IPs and OS info
kubectl api-resources                                  # List all available resource types
kubectl api-versions                                   # List supported API versions
kubectl version --output=yaml                          # Client and server versions

# List resources
kubectl get pods                                       # Pods in current namespace
kubectl get pods -A                                    # Pods in all namespaces
kubectl get pods -o wide                               # Show node placement and IPs
kubectl get all                                        # Common resources in current namespace
kubectl get deploy,svc,ingress                         # Multiple resource types at once

# Detailed information
kubectl describe pod <pod-name>                        # Detailed state, events, conditions
kubectl describe node <node-name>                      # Node capacity, allocatable, conditions

# Discover resource fields
kubectl explain pod.spec.containers                    # Schema docs for any field
kubectl explain pod.spec.containers.resources          # Drill deeper into nested fields
kubectl explain pod --recursive | head -100            # Full field tree (truncated)
```

## Creating and Modifying Resources

```bash
# Declarative management (recommended)
kubectl apply -f manifest.yaml                         # Create or update from file
kubectl apply -f ./manifests/                          # Apply all files in a directory
kubectl apply -f https://example.com/manifest.yaml     # Apply from URL
kubectl apply -k ./overlays/production/                # Apply Kustomize overlay

# Imperative creation (quick tasks, scripting)
kubectl create namespace staging                       # Create a namespace
kubectl create deployment nginx --image=nginx          # Create a deployment
kubectl create service clusterip my-svc --tcp=80:8080  # Create a service

# Editing live resources
kubectl edit deployment/nginx                          # Opens in $EDITOR
kubectl patch deployment nginx -p '{"spec":{"replicas":5}}'  # JSON patch
kubectl replace -f updated-manifest.yaml               # Replace entire resource

# Deleting resources
kubectl delete -f manifest.yaml                        # Delete from file
kubectl delete pod my-pod                              # Delete by name
kubectl delete pods -l app=old-version                 # Delete by label selector
kubectl delete namespace staging                       # Delete namespace and all contents
```

## Debugging and Troubleshooting

```bash
# Logs
kubectl logs <pod-name>                                # Stdout/stderr from container
kubectl logs <pod-name> -c <container>                 # Specific container in multi-container Pod
kubectl logs <pod-name> -f                             # Follow (stream) logs
kubectl logs <pod-name> --previous                     # Logs from previous crashed container
kubectl logs -l app=nginx --all-containers             # Logs from all Pods with label
kubectl logs deploy/nginx                              # Logs from a Deployment's Pods

# Interactive debugging
kubectl exec -it <pod-name> -- /bin/sh                 # Shell into a running container
kubectl exec <pod-name> -- cat /etc/resolv.conf        # Run a single command
kubectl debug -it <pod-name> --image=busybox --target=<container>  # Ephemeral debug container
kubectl debug node/<node-name> -it --image=ubuntu      # Debug at node level

# Port forwarding
kubectl port-forward pod/<pod-name> 8080:80            # Forward local port to Pod port
kubectl port-forward svc/<svc-name> 8080:80            # Forward via Service

# Resource utilization (requires Metrics Server)
kubectl top nodes                                      # CPU/memory per node
kubectl top pods                                       # CPU/memory per Pod
kubectl top pods --sort-by=memory                      # Sort by memory usage

# Events
kubectl get events --sort-by='.lastTimestamp'          # Recent cluster events
kubectl get events --field-selector reason=FailedScheduling  # Filter by reason
kubectl get events -w                                  # Watch events in real time

# Connectivity testing
kubectl run tmp-shell --rm -it --image=busybox -- /bin/sh              # Temporary debug Pod
kubectl run tmp-curl --rm -it --image=curlimages/curl -- curl http://my-svc  # Test service
```

## Working with Pods

```bash
# Run Pods
kubectl run nginx --image=nginx                        # Run a single Pod
kubectl run nginx --image=nginx --port=80              # With a declared port
kubectl run busybox --rm -it --image=busybox -- sh     # Interactive one-shot Pod

# Copy files to/from Pods
kubectl cp <pod-name>:/var/log/app.log ./app.log       # Copy from Pod to local
kubectl cp ./config.yaml <pod-name>:/app/config.yaml   # Copy from local to Pod

# Attach to running process
kubectl attach <pod-name> -it                          # Attach stdin/stdout to main process

# Check Pod status details
kubectl get pod <pod-name> -o yaml | grep -A5 status   # Raw status fields
kubectl get pod <pod-name> -o jsonpath='{.status.phase}'  # Just the phase
```

## Deployments and Scaling

```bash
# Rollouts
kubectl rollout status deploy/nginx                    # Watch rollout progress
kubectl rollout history deploy/nginx                   # View revision history
kubectl rollout undo deploy/nginx                      # Roll back to previous revision
kubectl rollout undo deploy/nginx --to-revision=2      # Roll back to specific revision
kubectl rollout restart deploy/nginx                   # Trigger a rolling restart
kubectl rollout pause deploy/nginx                     # Pause rollout (for batched changes)
kubectl rollout resume deploy/nginx                    # Resume paused rollout

# Scaling
kubectl scale deploy/nginx --replicas=5                # Scale to exact count
kubectl autoscale deploy/nginx --min=2 --max=10 --cpu-percent=70  # Create HPA
kubectl get hpa                                        # List Horizontal Pod Autoscalers

# Update image (imperative — prefer `kubectl apply` for production)
kubectl set image deploy/nginx nginx=nginx:1.27        # Update container image
```

## Namespaces and Context

```bash
# Namespace management
kubectl get namespaces                                 # List all namespaces
kubectl create namespace production                    # Create namespace
kubectl config set-context --current --namespace=production  # Switch default namespace

# Working across namespaces
kubectl get pods -n kube-system                        # List Pods in specific namespace
kubectl get pods -A                                    # All namespaces
kubectl get all -n monitoring                          # All resources in a namespace

# Context management for multi-cluster work
kubectl config get-contexts                            # Show all contexts
kubectl config rename-context old-name new-name        # Rename a context
kubectl config delete-context old-context              # Remove a context
```

## Output and Filtering

```bash
# Output formats
kubectl get pods -o yaml                               # Full YAML output
kubectl get pods -o json                               # Full JSON output
kubectl get pods -o wide                               # Extra columns (IP, node)
kubectl get pods -o name                               # Just resource names

# JSONPath queries
kubectl get pods -o jsonpath='{.items[*].metadata.name}'           # All Pod names
kubectl get pods -o jsonpath='{.items[*].status.podIP}'            # All Pod IPs
kubectl get nodes -o jsonpath='{.items[*].status.addresses[0].address}'  # Node IPs
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].image}'    # Container images

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP

# Sorting
kubectl get pods --sort-by='.metadata.creationTimestamp'  # Sort by creation time
kubectl get pods --sort-by='.status.startTime'            # Sort by start time
kubectl get pv --sort-by='.spec.capacity.storage'         # Sort PVs by size

# Label selectors
kubectl get pods -l app=nginx                          # Exact label match
kubectl get pods -l 'app in (nginx,apache)'            # Set-based selector
kubectl get pods -l app=nginx,env=prod                 # Multiple labels (AND)
kubectl get pods -l '!canary'                          # Label must not exist

# Field selectors
kubectl get pods --field-selector=status.phase=Running                 # Running Pods only
kubectl get pods --field-selector=spec.nodeName=worker-1               # Pods on specific node
kubectl get events --field-selector=involvedObject.name=my-pod         # Events for a Pod
```

## Useful Aliases and Shortcuts

```bash
# Common aliases
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias ka='kubectl apply -f'
alias kdel='kubectl delete'
alias kl='kubectl logs'
alias ke='kubectl exec -it'

# Dry-run patterns for generating YAML (never memorize YAML from scratch)
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl create service clusterip my-svc --tcp=80:8080 --dry-run=client -o yaml
kubectl create configmap my-config --from-literal=key=value --dry-run=client -o yaml
kubectl create secret generic my-secret --from-literal=pass=s3cr3t --dry-run=client -o yaml
kubectl create job my-job --image=busybox --dry-run=client -o yaml -- echo "hello"
kubectl create cronjob my-cron --image=busybox --schedule="*/5 * * * *" --dry-run=client -o yaml -- echo "tick"
kubectl run nginx --image=nginx --dry-run=client -o yaml                    # Pod YAML

# Quick resource generation
kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f -

# Useful one-liners
kubectl get pods --no-headers | wc -l                  # Count Pods
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.cpu}{"\n"}{end}'
```

## Essential YAML Patterns

### Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    ports:
    - containerPort: 80
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            memory: 256Mi
```

### Service (ClusterIP)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  APP_LOG_LEVEL: info
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
```

### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  db-password: my-secret-password
  api-key: abc123
```

### PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

### NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
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
      port: 8080
```

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
```

### CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: busybox:1.36
            command: ["/bin/sh", "-c", "echo Running cleanup at $(date)"]
          restartPolicy: OnFailure
```

### HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

*Navigation: [← Appendix A: Glossary](appendix-a-glossary.md) | [Appendix C: Cloud Platform Comparison →](appendix-c-cloud-platforms.md)*
