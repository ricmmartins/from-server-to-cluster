# Apêndice B: Referência Rápida do kubectl

*Comandos essenciais e padrões YAML para o trabalho diário com Kubernetes.*

---

A base didática local é **Kubernetes 1.35.0, kind 0.31.0 e kubectl 1.35.x**. Para clusters gerenciados, primeiro consulte as versões disponíveis e suportadas na região (Apêndice C); use um cliente kubectl com diferença de no máximo uma versão minor em relação ao API server, em vez de presumir que a base local serve para qualquer cluster cloud.

## Configuração Inicial

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
gcloud container clusters get-credentials my-cluster --region us-central1  # GKE

# Enable shell autocompletion
source <(kubectl completion bash)                      # Bash (current session)
echo 'source <(kubectl completion bash)' >> ~/.bashrc  # Bash (permanent)
source <(kubectl completion zsh)                       # Zsh
```

## Obtendo Informações

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

## Criando e Modificando Recursos

```bash
# Declarative management (recommended)
kubectl apply -f manifest.yaml                         # Create or update from file
kubectl apply -f ./manifests/                          # Apply all files in a directory
kubectl apply -f https://example.com/manifest.yaml     # Apply from URL
kubectl apply -k ./overlays/production/                # Apply Kustomize overlay

# Imperative creation (quick tasks, scripting)
kubectl create namespace staging                       # Create a namespace
kubectl create deployment nginx --image=nginx:1.27     # Create a deployment
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

## Depuração e Solução de Problemas

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

## Trabalhando com Pods

```bash
# Run Pods
kubectl run nginx --image=nginx:1.27                   # Run a single Pod
kubectl run nginx --image=nginx:1.27 --port=80         # With a declared port
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

## Deployments e Escalonamento

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

## Namespaces e Contexto

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

## Saída e Filtragem

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

## Aliases e Atalhos Úteis

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
kubectl create deployment nginx --image=nginx:1.27 --dry-run=client -o yaml > deploy.yaml
kubectl create service clusterip my-svc --tcp=80:8080 --dry-run=client -o yaml
kubectl create configmap my-config --from-literal=key=value --dry-run=client -o yaml
kubectl create secret generic my-secret --from-literal=pass=s3cr3t --dry-run=client -o yaml
kubectl create job my-job --image=busybox --dry-run=client -o yaml -- echo "hello"
kubectl create cronjob my-cron --image=busybox --schedule="*/5 * * * *" --dry-run=client -o yaml -- echo "tick"
kubectl run nginx --image=nginx:1.27 --dry-run=client -o yaml               # Pod YAML

# Quick resource generation
kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f -

# Useful one-liners
kubectl get pods --no-headers | wc -l                  # Count Pods
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.cpu}{"\n"}{end}'
```

## Padrões YAML Essenciais

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

Exige um CNI/plugin de rede com **aplicação de NetworkPolicy habilitada**. A aceitação pela API, por si só, não comprova nada; a rede padrão do kind não aplica essas regras sem uma implementação de rede adequada.

Aplique esta política no namespace do backend (`kubectl apply -n <namespace> -f policy.yaml`). Ela isola o tráfego de entrada dos Pods `app=backend` e permite TCP 8080 a partir de Pods `app=frontend` **nesse mesmo namespace**. Não isola o tráfego de saída do backend. Origens em outros namespaces precisam de um `namespaceSelector`; combiná-lo com um `podSelector` na mesma origem representa AND, enquanto origens separadas representam OR.

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

As políticas são **aditivas**: outra política que selecione os mesmos Pods pode permitir mais tráfego. Uma conexão precisa atender, de forma independente, às políticas de entrada do destino e de saída da origem. Teste um backend real escutando na porta 8080 a partir de um frontend permitido, de um cliente sem label/bloqueado e de um cliente em outro namespace; na ausência de outras regras de permissão, apenas o cliente permitido deve conseguir se conectar. Use conexões novas após a propagação da política e inspecione todas as políticas se os resultados forem diferentes:

```bash
kubectl get networkpolicy -n <namespace>
kubectl describe networkpolicy allow-frontend -n <namespace>
# Substitute existing test Pod names and the backend Pod IP; avoids DNS ambiguity.
kubectl exec -n <namespace> <frontend-pod> -- curl --connect-timeout 3 http://<backend-pod-ip>:8080
kubectl exec -n <namespace> <denied-client-pod> -- curl --connect-timeout 3 http://<backend-pod-ip>:8080
kubectl exec -n <other-namespace> <client-pod> -- curl --connect-timeout 3 http://<backend-pod-ip>:8080
```

Os clientes precisam de curl. Não se alega aqui a realização de testes de conectividade em runtime. Consulte a documentação oficial sobre [semântica e pré-requisitos de NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) (consultada em **2026-10-15**).

### Ingress

Este exemplo exige um controller **Traefik** mantido, com seu provider Kubernetes Ingress habilitado e uma `IngressClass` chamada `traefik` (`spec.controller: traefik.io/ingress-controller`), em conformidade com a versão `traefik:v3.7.15` fixada no Capítulo 7. Primeiro verifique `kubectl get ingressclass traefik` e se o controller está pronto. O caminho da requisição é preservado; não há annotation de reescrita específica do controller. O backend `nginx-svc` e o Secret TLS `app-tls` devem existir no namespace do Ingress, com certificado válido para o host e DNS apontando para o controller. Para esta variante com TLS, também configure e exponha um entrypoint do Traefik com TLS habilitado; o mapeamento de portas apenas HTTP do Capítulo 7 não basta para HTTPS.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: traefik
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

Referências consultadas em **2026-10-15**: [Traefik Kubernetes Ingress](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/ingress/) e [descontinuação do ingress-nginx da comunidade](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/) (a manutenção terminou em março de 2026). A imagem da aplicação nginx usada em outros exemplos desta referência rápida não é o controller ingress-nginx descontinuado.

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

*Navegação: [← Apêndice A: Glossário](appendix-a-glossary.md) | [Apêndice C: Comparação de Plataformas Cloud →](appendix-c-cloud-platforms.md)*
