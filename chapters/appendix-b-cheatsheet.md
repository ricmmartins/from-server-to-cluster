# Apêndice B: Referência Rápida do kubectl

*Comandos essenciais e padrões YAML para o trabalho diário com Kubernetes.*

---

## Configuração Inicial

```bash
# Visualizar e gerenciar kubeconfig
kubectl config view                                    # Mostrar kubeconfig mesclado
kubectl config get-contexts                            # Listar todos os contextos
kubectl config current-context                         # Mostrar contexto ativo
kubectl config use-context <context-name>              # Alternar para um contexto diferente
kubectl config set-context --current --namespace=dev   # Definir namespace padrão para o contexto atual

# Obter credenciais do cluster (Kubernetes gerenciado)
az aks get-credentials --resource-group myRG --name myCluster       # AKS
aws eks update-kubeconfig --name myCluster --region us-west-2       # EKS
gcloud container clusters get-credentials myCluster --region us-central1  # GKE

# Habilitar autocompletar no shell
source <(kubectl completion bash)                      # Bash (sessão atual)
echo 'source <(kubectl completion bash)' >> ~/.bashrc  # Bash (permanente)
source <(kubectl completion zsh)                       # Zsh
```

## Obtendo Informações

```bash
# Visão geral do cluster
kubectl cluster-info                                   # Endpoints do control plane e CoreDNS
kubectl get nodes -o wide                              # Listar nós com IPs e info do SO
kubectl api-resources                                  # Listar todos os tipos de recursos disponíveis
kubectl api-versions                                   # Listar versões de API suportadas
kubectl version --output=yaml                          # Versões do cliente e servidor

# Listar recursos
kubectl get pods                                       # Pods no namespace atual
kubectl get pods -A                                    # Pods em todos os namespaces
kubectl get pods -o wide                               # Mostrar alocação em nós e IPs
kubectl get all                                        # Recursos comuns no namespace atual
kubectl get deploy,svc,ingress                         # Múltiplos tipos de recursos de uma vez

# Informações detalhadas
kubectl describe pod <pod-name>                        # Estado detalhado, eventos, condições
kubectl describe node <node-name>                      # Capacidade do nó, alocável, condições

# Descobrir campos de recursos
kubectl explain pod.spec.containers                    # Documentação do schema para qualquer campo
kubectl explain pod.spec.containers.resources          # Aprofundar em campos aninhados
kubectl explain pod --recursive | head -100            # Árvore completa de campos (truncada)
```

## Criando e Modificando Recursos

```bash
# Gerenciamento declarativo (recomendado)
kubectl apply -f manifest.yaml                         # Criar ou atualizar a partir de arquivo
kubectl apply -f ./manifests/                          # Aplicar todos os arquivos em um diretório
kubectl apply -f https://example.com/manifest.yaml     # Aplicar a partir de URL
kubectl apply -k ./overlays/production/                # Aplicar overlay Kustomize

# Criação imperativa (tarefas rápidas, scripts)
kubectl create namespace staging                       # Criar um namespace
kubectl create deployment nginx --image=nginx          # Criar um deployment
kubectl create service clusterip my-svc --tcp=80:8080  # Criar um service

# Editando recursos ativos
kubectl edit deployment/nginx                          # Abre no $EDITOR
kubectl patch deployment nginx -p '{"spec":{"replicas":5}}'  # Patch JSON
kubectl replace -f updated-manifest.yaml               # Substituir recurso inteiro

# Excluindo recursos
kubectl delete -f manifest.yaml                        # Excluir a partir de arquivo
kubectl delete pod my-pod                              # Excluir por nome
kubectl delete pods -l app=old-version                 # Excluir por seletor de label
kubectl delete namespace staging                       # Excluir namespace e todo seu conteúdo
```

## Depuração e Solução de Problemas

```bash
# Logs
kubectl logs <pod-name>                                # Stdout/stderr do container
kubectl logs <pod-name> -c <container>                 # Container específico em Pod multi-container
kubectl logs <pod-name> -f                             # Seguir (stream) logs
kubectl logs <pod-name> --previous                     # Logs do container anterior que falhou
kubectl logs -l app=nginx --all-containers             # Logs de todos os Pods com label
kubectl logs deploy/nginx                              # Logs dos Pods de um Deployment

# Depuração interativa
kubectl exec -it <pod-name> -- /bin/sh                 # Shell em um container em execução
kubectl exec <pod-name> -- cat /etc/resolv.conf        # Executar um único comando
kubectl debug -it <pod-name> --image=busybox --target=<container>  # Container efêmero de debug
kubectl debug node/<node-name> -it --image=ubuntu      # Debug no nível do nó

# Port forwarding
kubectl port-forward pod/<pod-name> 8080:80            # Encaminhar porta local para porta do Pod
kubectl port-forward svc/<svc-name> 8080:80            # Encaminhar via Service

# Utilização de recursos (requer Metrics Server)
kubectl top nodes                                      # CPU/memória por nó
kubectl top pods                                       # CPU/memória por Pod
kubectl top pods --sort-by=memory                      # Ordenar por uso de memória

# Eventos
kubectl get events --sort-by='.lastTimestamp'          # Eventos recentes do cluster
kubectl get events --field-selector reason=FailedScheduling  # Filtrar por motivo
kubectl get events -w                                  # Observar eventos em tempo real

# Teste de conectividade
kubectl run tmp-shell --rm -it --image=busybox -- /bin/sh              # Pod temporário de debug
kubectl run tmp-curl --rm -it --image=curlimages/curl -- curl http://my-svc  # Testar service
```

## Trabalhando com Pods

```bash
# Executar Pods
kubectl run nginx --image=nginx                        # Executar um único Pod
kubectl run nginx --image=nginx --port=80              # Com uma porta declarada
kubectl run busybox --rm -it --image=busybox -- sh     # Pod interativo descartável

# Copiar arquivos de/para Pods
kubectl cp <pod-name>:/var/log/app.log ./app.log       # Copiar do Pod para local
kubectl cp ./config.yaml <pod-name>:/app/config.yaml   # Copiar do local para Pod

# Anexar ao processo em execução
kubectl attach <pod-name> -it                          # Anexar stdin/stdout ao processo principal

# Verificar detalhes do status do Pod
kubectl get pod <pod-name> -o yaml | grep -A5 status   # Campos brutos de status
kubectl get pod <pod-name> -o jsonpath='{.status.phase}'  # Apenas a fase
```

## Deployments e Escalonamento

```bash
# Rollouts
kubectl rollout status deploy/nginx                    # Acompanhar progresso do rollout
kubectl rollout history deploy/nginx                   # Ver histórico de revisões
kubectl rollout undo deploy/nginx                      # Reverter para revisão anterior
kubectl rollout undo deploy/nginx --to-revision=2      # Reverter para revisão específica
kubectl rollout restart deploy/nginx                   # Disparar reinício progressivo
kubectl rollout pause deploy/nginx                     # Pausar rollout (para mudanças em lote)
kubectl rollout resume deploy/nginx                    # Retomar rollout pausado

# Escalonamento
kubectl scale deploy/nginx --replicas=5                # Escalar para quantidade exata
kubectl autoscale deploy/nginx --min=2 --max=10 --cpu-percent=70  # Criar HPA
kubectl get hpa                                        # Listar Horizontal Pod Autoscalers

# Atualizar imagem (imperativo — prefira `kubectl apply` para produção)
kubectl set image deploy/nginx nginx=nginx:1.27        # Atualizar imagem do container
```

## Namespaces e Contexto

```bash
# Gerenciamento de namespaces
kubectl get namespaces                                 # Listar todos os namespaces
kubectl create namespace production                    # Criar namespace
kubectl config set-context --current --namespace=production  # Alterar namespace padrão

# Trabalhando entre namespaces
kubectl get pods -n kube-system                        # Listar Pods em namespace específico
kubectl get pods -A                                    # Todos os namespaces
kubectl get all -n monitoring                          # Todos os recursos em um namespace

# Gerenciamento de contexto para trabalho multi-cluster
kubectl config get-contexts                            # Mostrar todos os contextos
kubectl config rename-context old-name new-name        # Renomear um contexto
kubectl config delete-context old-context              # Remover um contexto
```

## Saída e Filtragem

```bash
# Formatos de saída
kubectl get pods -o yaml                               # Saída YAML completa
kubectl get pods -o json                               # Saída JSON completa
kubectl get pods -o wide                               # Colunas extras (IP, nó)
kubectl get pods -o name                               # Apenas nomes dos recursos

# Consultas JSONPath
kubectl get pods -o jsonpath='{.items[*].metadata.name}'           # Todos os nomes de Pods
kubectl get pods -o jsonpath='{.items[*].status.podIP}'            # Todos os IPs de Pods
kubectl get nodes -o jsonpath='{.items[*].status.addresses[0].address}'  # IPs dos nós
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].image}'    # Imagens dos containers

# Colunas customizadas
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP

# Ordenação
kubectl get pods --sort-by='.metadata.creationTimestamp'  # Ordenar por data de criação
kubectl get pods --sort-by='.status.startTime'            # Ordenar por hora de início
kubectl get pv --sort-by='.spec.capacity.storage'         # Ordenar PVs por tamanho

# Seletores de label
kubectl get pods -l app=nginx                          # Correspondência exata de label
kubectl get pods -l 'app in (nginx,apache)'            # Seletor baseado em conjunto
kubectl get pods -l app=nginx,env=prod                 # Múltiplas labels (AND)
kubectl get pods -l '!canary'                          # Label não deve existir

# Seletores de campo
kubectl get pods --field-selector=status.phase=Running                 # Apenas Pods em execução
kubectl get pods --field-selector=spec.nodeName=worker-1               # Pods em nó específico
kubectl get events --field-selector=involvedObject.name=my-pod         # Eventos de um Pod
```

## Aliases e Atalhos Úteis

```bash
# Aliases comuns
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias ka='kubectl apply -f'
alias kdel='kubectl delete'
alias kl='kubectl logs'
alias ke='kubectl exec -it'

# Padrões dry-run para gerar YAML (nunca memorize YAML do zero)
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl create service clusterip my-svc --tcp=80:8080 --dry-run=client -o yaml
kubectl create configmap my-config --from-literal=key=value --dry-run=client -o yaml
kubectl create secret generic my-secret --from-literal=pass=s3cr3t --dry-run=client -o yaml
kubectl create job my-job --image=busybox --dry-run=client -o yaml -- echo "hello"
kubectl create cronjob my-cron --image=busybox --schedule="*/5 * * * *" --dry-run=client -o yaml -- echo "tick"
kubectl run nginx --image=nginx --dry-run=client -o yaml                    # YAML de Pod

# Geração rápida de recursos
kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f -

# One-liners úteis
kubectl get pods --no-headers | wc -l                  # Contar Pods
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

*Navegação: [← Apêndice A: Glossário](appendix-a-glossary.md) | [Apêndice C: Comparação de Plataformas Cloud →](appendix-c-cloud-platforms.md)*
