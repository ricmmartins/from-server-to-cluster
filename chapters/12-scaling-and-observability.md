# Capítulo 12: Escalabilidade e Observabilidade

*"Você não pode melhorar o que não pode medir. E não pode escalar o que não entende."*

---

## De top a kubectl top: Monitoramento em Escala de Cluster

Em um servidor Linux, o gerenciamento de recursos é feito de forma direta. Você abre o `top` e vê cada processo, seu percentual de CPU, seu uso de memória. Você ajusta cgroups para limitar um container descontrolado. Configura o `sar` para rastrear tendências históricas. Quando o tráfego aumenta, você faz SSH, verifica a carga e talvez suba outra VM atrás de um load balancer. É manual, reativo e vinculado a máquinas que você pode nomear.

O Kubernetes pega essas mesmas preocupações — *quanto de CPU essa coisa está usando? Quanta memória ela tem? O que acontece quando a demanda excede a capacidade?* — e as transforma em sistemas declarativos e automatizados. Em vez de verificar o `top` e iniciar novos processos manualmente, você define **requests** e **limits** de recursos na spec do seu pod, e o Kubernetes cuida do scheduling, enforcement e até do escalonamento automático.

Os conceitos são os mesmos. CPU ainda é CPU. Memória ainda é memória. OOM kills ainda acontecem. Mas o mecanismo muda de "admin observa um dashboard e reage" para "o sistema observa métricas e age." Seu trabalho não é mais ficar de babá nos servidores — é definir as regras e deixar o Kubernetes aplicá-las.

Para um administrador Linux, este capítulo é onde seu conhecimento de `cgroups`, `top`, `sar` e `ulimit` se torna diretamente útil — porque o Kubernetes usa exatamente esses recursos do kernel por baixo dos panos. Ele apenas os envolve em YAML.

---

## Gerenciamento de Recursos: Requests e Limits

Todo container no Kubernetes pode declarar quanto de CPU e memória precisa (requests) e quanto pode usar (limits). É assim que o Kubernetes faz planejamento de capacidade e enforcement — e mapeia diretamente para cgroups do Linux.

### CPU: Medida em Millicores

O Kubernetes mede CPU em **millicores** (também chamados millicpu). 1000m = 1 core completo de CPU.

- `100m` = 10% de um core
- `500m` = metade de um core
- `2000m` ou `2` = dois cores completos

Por baixo dos panos, isso configura o valor `cpu.max` do cgroup do container. Um limit de `500m` significa que o container é limitado (throttled) se tentar usar mais da metade de um core.

### Memória: Medida em Bytes

Memória é especificada em bytes, tipicamente usando sufixos:

- `64Mi` = 64 mebibytes (67.108.864 bytes)
- `256Mi` = 256 mebibytes
- `1Gi` = 1 gibibyte

Por baixo dos panos, isso configura o `memory.max` do cgroup do container. Se um container exceder seu limite de memória, o OOM killer do Linux o encerra imediatamente — sem aviso, sem shutdown gracioso.

### Requests vs. Limits

| | Requests | Limits |
|---|---------|--------|
| **O que significa** | Mínimo garantido | Máximo permitido |
| **Quem usa** | Scheduler (para posicionamento de pods) | Kubelet/kernel (para enforcement) |
| **Comportamento de CPU** | O node deve ter essa quantidade de CPU disponível para agendar o pod | Container é limitado (throttled) além disso |
| **Comportamento de memória** | O node deve ter essa quantidade de memória disponível para agendar o pod | Container sofre OOMKill além disso |
| **Equivalente cgroup Linux** | `cpu.weight` / garantia de scheduling | `cpu.max` / `memory.max` |

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: app
    image: nginx:1.27
    resources:
      requests:
        cpu: "100m"        # Garantia: 10% de um core
        memory: "64Mi"     # Garantia: 64 MiB
      limits:
        cpu: "200m"        # Teto: 20% de um core
        memory: "128Mi"    # Teto: 128 MiB (OOMKill se exceder)
```

**Regra crítica:** Sempre defina requests. Sempre defina limits. Pods sem especificação de recursos são os primeiros a serem despejados sob pressão, e tornam o planejamento de capacidade impossível.

---

## Classes QoS: Prioridade de Despejo

Com base em como você define requests e limits, o Kubernetes atribui a cada pod uma **classe de Quality of Service (QoS)** que determina sua prioridade de despejo quando o node fica com poucos recursos. Este é o equivalente Kubernetes das prioridades do OOM killer do Linux (`oom_score_adj`).

| Classe QoS | Condição | Prioridade de Despejo | Analogia Linux |
|-----------|-----------|-------------------|---------------|
| **Guaranteed** | `requests == limits` para todos os containers | Último a ser despejado | `oom_score_adj = -997` |
| **Burstable** | `requests < limits` para pelo menos um container | Prioridade média | `oom_score_adj` varia |
| **BestEffort** | Sem requests ou limits definidos | **Primeiro a ser despejado** | `oom_score_adj = 1000` |

```yaml
# QoS Guaranteed — requests iguais a limits
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

```yaml
# QoS Burstable — requests menores que limits
resources:
  requests:
    cpu: "100m"
    memory: "64Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

```yaml
# QoS BestEffort — sem requests ou limits
# (Não faça isso em produção!)
resources: {}
```

Quando um node fica sem memória, o kubelet despeja pods nesta ordem: BestEffort primeiro, depois Burstable (começando pelos que usam mais memória em relação ao seu request), e então Guaranteed por último. É triagem — assim como o OOM killer do Linux, mas com consciência de prioridades de pods no nível do Kubernetes.

---

## LimitRange: Padrões no Nível do Namespace

O que acontece quando um desenvolvedor faz deploy de um pod sem especificar recursos? Sem proteções, ele se torna um pod BestEffort que pode consumir recursos ilimitados e é o primeiro a ser despejado. **LimitRange** resolve isso definindo valores padrão de recursos e restrições por namespace.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: default
spec:
  limits:
  - type: Container
    default:              # Aplicado se nenhum limit for especificado
      cpu: "200m"
      memory: "128Mi"
    defaultRequest:       # Aplicado se nenhum request for especificado
      cpu: "100m"
      memory: "64Mi"
    max:                  # Limits máximos que um container pode solicitar
      cpu: "1"
      memory: "512Mi"
    min:                  # Requests mínimos que um container deve especificar
      cpu: "50m"
      memory: "32Mi"
```

Pense no LimitRange como `ulimit` para um namespace Kubernetes — ele define restrições padrão e máximas de recursos para que nenhum pod individual seja um vizinho mal-educado.

---

## Horizontal Pod Autoscaler (HPA)

Em um servidor Linux, escalar é manual: você observa o `top`, decide que o servidor está sobrecarregado, sobe outra VM, configura o load balancer e faz deploy da aplicação novamente. Leva minutos a horas.

O Horizontal Pod Autoscaler (HPA) torna isso automático. Ele observa métricas (utilização de CPU, uso de memória ou métricas customizadas), compara com targets que você define, e ajusta o número de réplicas para cima ou para baixo.

### Como o HPA Funciona

O controller do HPA executa um loop de controle (a cada 15 segundos por padrão) que:

1. Busca métricas atuais do Metrics Server (ou de uma API de métricas customizadas)
2. Compara a utilização atual com o target
3. Calcula o número desejado de réplicas usando esta fórmula:

```
desiredReplicas = ceil(currentReplicas × (currentMetricValue / targetMetricValue))
```

Por exemplo: se você tem 2 réplicas com 80% de utilização de CPU e seu target é 50%, o HPA calcula:

```
desiredReplicas = ceil(2 × (80 / 50)) = ceil(3.2) = 4
```

Ele escala de 2 para 4 réplicas para trazer a utilização média de volta para próximo de 50%.

### Criando um HPA

```bash
# Imperativo (rápido)
kubectl autoscale deployment my-app --cpu-percent=50 --min=1 --max=10

# Ou declarativo (recomendado)
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### Comportamento de Escalonamento e Cooldowns

O HPA inclui políticas de escalonamento configuráveis para prevenir thrashing (ciclos rápidos de scale-up/scale-down):

- **Scale-up:** Por padrão, o HPA pode escalar rapidamente — responde em 15–30 segundos após detectar aumento de carga.
- **Scale-down:** Por padrão, o HPA espera 5 minutos (a janela de estabilização) antes de reduzir. Isso previne scale-down prematuro se um pico de carga for temporário.

Você pode customizar este comportamento:

```yaml
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60       # Esperar 60s antes de escalar mais
      policies:
      - type: Percent
        value: 100                          # Pode dobrar réplicas em um passo
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300      # Esperar 5 min antes de reduzir
      policies:
      - type: Percent
        value: 10                           # Remover no máximo 10% das réplicas por passo
        periodSeconds: 60
```

---

## Vertical Pod Autoscaler (VPA)

Enquanto o HPA escala *horizontalmente* (mais réplicas), o Vertical Pod Autoscaler escala *verticalmente* (mais recursos por pod). O VPA observa o uso real de recursos ao longo do tempo e recomenda — ou aplica automaticamente — ajustes de requests e limits de CPU e memória.

### HPA vs. VPA

| | HPA | VPA |
|---|-----|-----|
| **Escala** | Número de réplicas (horizontal) | Requests/limits de recursos (vertical) |
| **Quando usar** | Workloads stateless que podem rodar múltiplas instâncias | Workloads onde o tamanho certo de recurso é incerto |
| **Requer** | Metrics Server | Controller VPA (instalação separada) |
| **Prontidão para produção** | GA, amplamente usado | Maduro, mas modo auto-update reinicia pods |

> **Nota:** VPA é um projeto separado, não integrado ao core do Kubernetes. Requer a instalação dos componentes do controller VPA. Para os labs deste livro, focamos no HPA — mas VPA é um conceito importante para entender o dimensionamento correto em produção.

---

## Metrics Server: A Base

Tanto o HPA quanto o `kubectl top` dependem do **Metrics Server** — um agregador leve, em memória, que coleta métricas de CPU e memória dos kubelets em cada node.

### O Que o Metrics Server Fornece

- `kubectl top nodes` — Uso de CPU e memória por node
- `kubectl top pods` — Uso de CPU e memória por pod
- API de Métricas (`metrics.k8s.io`) consumida pelo HPA

### O Que o Metrics Server Não Fornece

- Dados históricos (ele só mostra valores atuais)
- Métricas customizadas (métricas no nível da aplicação)
- Métricas de disco, rede ou outros recursos

Pense no Metrics Server como o `top` para seu cluster: tempo real, leve e limitado a CPU e memória. Para qualquer coisa além disso, você precisa de uma stack de monitoramento completa.

---

## Prometheus + Grafana: Monitoramento de Produção

Se o Metrics Server é o `top`, então o **Prometheus** é o `sar` + `collectd` + `nagios` combinados — um sistema completo de coleta de métricas, armazenamento e alertas. Combinado com o **Grafana** para visualização, é a stack de monitoramento de fato para Kubernetes.

### Como o Prometheus Funciona

O Prometheus usa um **modelo pull**: ele coleta (scrape) endpoints de métricas em intervalos regulares (tipicamente a cada 15–30 segundos). Pods Kubernetes expõem métricas em endpoints `/metrics` em um formato padrão, e o Prometheus descobre targets via service discovery do Kubernetes.

Conceitos-chave:

| Conceito | Descrição |
|---------|------------|
| **Scraping** | Prometheus periodicamente coleta métricas dos targets |
| **PromQL** | Linguagem de consulta para fatiar e agregar dados de métricas |
| **Alertmanager** | Roteia alertas para Slack, PagerDuty, email, etc. |
| **Time series** | Toda métrica é armazenada com timestamps para análise histórica |

### Métricas Importantes para Monitorar

| Métrica | O Que Ela Diz |
|--------|-------------------|
| `container_cpu_usage_seconds_total` | Consumo de CPU por container |
| `container_memory_working_set_bytes` | Memória real em uso (o que o OOM killer observa) |
| `kube_pod_status_phase` | Quantos pods estão em cada fase (Running, Pending, Failed) |
| `kube_deployment_status_replicas_available` | Se seus deployments têm a contagem esperada de réplicas |
| `node_cpu_seconds_total` | Utilização de CPU no nível do node |
| `node_memory_MemAvailable_bytes` | Memória disponível por node |

### O kube-prometheus-stack

A forma mais fácil de fazer deploy do Prometheus e Grafana no Kubernetes é o chart Helm **kube-prometheus-stack**. Ele instala Prometheus, Grafana, Alertmanager, node-exporter, kube-state-metrics e um conjunto de dashboards e regras de alerta pré-configurados — tudo em um único Helm install.

---

## Comparação Linux ↔ Kubernetes

| Conceito Linux | Equivalente Kubernetes | Notas |
|---------------|----------------------|-------|
| cgroups `cpu.max` | `resources.limits.cpu` | Teto rígido de CPU |
| cgroups `memory.max` | `resources.limits.memory` | OOMKill se exceder |
| Prioridade `nice`/`renice` | Classes QoS (Guaranteed, Burstable, BestEffort) | Prioridade de despejo sob pressão |
| OOM killer (`oom_score_adj`) | Despejo de pods | Menor QoS é despejado primeiro |
| `top`, `htop` | `kubectl top`, Metrics Server | Uso de recursos em tempo real |
| `sar`, `collectd` | Prometheus | Coleta de métricas históricas e alertas |
| Grafana/Cacti | Grafana | Visualização e dashboards |
| Escalonamento manual (adicionar VMs) | HPA | Escalonamento automático de réplicas baseado em métricas |
| `ulimit` | LimitRange | Restrições padrão/máximas de recursos por namespace |

---

> ### 📊 Onde a Analogia com Linux Quebra
>
> - **Escalonamento Linux é vertical ou manual. Escalonamento Kubernetes é automático e horizontal.** No Linux, quando um servidor está sobrecarregado, você ou adiciona mais RAM/CPU (escalonamento vertical) ou provisiona manualmente novos servidores e configura load balancers (escalonamento horizontal). Ambos requerem intervenção humana e levam minutos a horas. O HPA reage em segundos e cuida de tudo — criação de réplicas, scheduling, distribuição de carga — sem nenhum passo manual. A infraestrutura cuida do posicionamento automaticamente.
>
> - **`kubectl top` mostra agora, não antes.** Não há equivalente integrado ao `sar` ou `vmstat` que rastreie uso de recursos ao longo do tempo. O Metrics Server é puramente tempo real — um snapshot deste momento. Se você precisa de "o que aconteceu às 3 da manhã quando os alertas dispararam?" você precisa do Prometheus ou de um banco de dados de séries temporais similar. O Kubernetes não vem com monitoramento histórico.
>
> - **Limites de recursos no Kubernetes são rígidos — não há rede de segurança.** No Linux, se um processo excede a memória física disponível, o sistema pode usar swap em disco, dando tempo para reagir. No Kubernetes, não há swap por padrão. Exceder um limite de memória dispara um OOMKill imediato — o container é terminado instantaneamente sem shutdown gracioso para a violação de limite. Defina seus limits muito baixos, e seus pods morrem. Defina muito altos, e você desperdiça capacidade do cluster. Acertar os limites de recursos é um dos desafios operacionais mais difíceis no Kubernetes.

---

## Laboratório Diagnóstico: Escalabilidade e Observabilidade na Prática

### Pré-requisitos

Crie um cluster Kind:

```bash
kind create cluster --name scaling-lab
```

### Lab 1: Resource Requests, Limits e Classes QoS

**Passo 1 — Fazer deploy de pods com diferentes configurações de recursos:**

```bash
# QoS Guaranteed (requests == limits)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
  - name: app
    image: nginx:1.27
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "100m"
        memory: "64Mi"
EOF

# QoS Burstable (requests < limits)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
spec:
  containers:
  - name: app
    image: nginx:1.27
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "200m"
        memory: "128Mi"
EOF

# QoS BestEffort (sem requests ou limits)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
spec:
  containers:
  - name: app
    image: nginx:1.27
EOF
```

**Passo 2 — Verificar a classe QoS atribuída a cada pod:**

```bash
kubectl get pod guaranteed-pod -o jsonpath='{.status.qosClass}'
echo ""
kubectl get pod burstable-pod -o jsonpath='{.status.qosClass}'
echo ""
kubectl get pod besteffort-pod -o jsonpath='{.status.qosClass}'
echo ""
```

Saída esperada: `Guaranteed`, `Burstable`, `BestEffort`.

**Passo 3 — Inspecionar as alocações de recursos:**

```bash
kubectl describe pod guaranteed-pod | grep -A6 "Limits:"
kubectl describe pod burstable-pod | grep -A6 "Limits:"
kubectl describe pod besteffort-pod | grep -A6 "Limits:"
```

**Passo 4 — Limpar os pods de demo QoS:**

```bash
kubectl delete pod guaranteed-pod burstable-pod besteffort-pod
```

### Lab 2: Instalar Metrics Server no Kind

O Kind não vem com Metrics Server. Vamos instalá-lo.

**Passo 1 — Aplicar o manifesto do Metrics Server:**

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**Passo 2 — Fazer patch no deployment para adicionar `--kubelet-insecure-tls`:**

O Kind usa certificados auto-assinados, então o Metrics Server precisa pular a verificação TLS:

```bash
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
```

**Passo 3 — Aguardar o Metrics Server ficar pronto:**

```bash
kubectl wait --for=condition=Available deployment/metrics-server -n kube-system --timeout=120s
```

**Passo 4 — Verificar que está coletando métricas (pode levar 30–60 segundos):**

```bash
# Métricas no nível do node
kubectl top nodes

# Métricas no nível do pod (execute um pod primeiro se necessário)
kubectl run metrics-test --image=nginx:1.27 --restart=Never
sleep 30
kubectl top pods
```

Se `kubectl top` retornar "Metrics not available yet," aguarde mais 30 segundos — o Metrics Server precisa de alguns ciclos de coleta.

**Passo 5 — Limpar o pod de teste:**

```bash
kubectl delete pod metrics-test
```

### Lab 3: HPA em Ação

Este lab segue o walkthrough oficial do HPA do Kubernetes usando o exemplo `php-apache`.

**Passo 1 — Fazer deploy da aplicação php-apache:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      run: php-apache
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "200m"
          limits:
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache
EOF
```

Aguarde o deployment ficar pronto:

```bash
kubectl wait --for=condition=Available deployment/php-apache --timeout=120s
```

**Passo 2 — Criar um HPA com target de 50% de utilização de CPU:**

```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

**Passo 3 — Verificar que o HPA está ativo:**

```bash
kubectl get hpa
```

Você deve ver o HPA com `TARGETS` mostrando o uso atual de CPU vs. o target de 50%. Se mostrar `<unknown>/50%`, aguarde o Metrics Server coletar dados.

**Passo 4 — Gerar carga:**

Abra um terminal separado (ou execute em background) para gerar requisições HTTP contínuas:

```bash
kubectl run load-generator --image=busybox:1.37 --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
```

**Passo 5 — Observar o HPA escalando:**

```bash
kubectl get hpa php-apache --watch
```

Dentro de 1–2 minutos, você deve ver:
- Utilização de CPU subindo acima de 50%
- `REPLICAS` aumentando conforme o HPA adiciona mais pods
- Eventualmente, a utilização estabilizando em torno do target de 50%

Pressione `Ctrl+C` para parar de observar, depois verifique os pods:

```bash
kubectl get pods -l run=php-apache
```

Você deve ver múltiplas réplicas rodando.

**Passo 6 — Parar a carga e observar o scale-down:**

```bash
kubectl delete pod load-generator
```

Agora observe o HPA novamente:

```bash
kubectl get hpa php-apache --watch
```

Após aproximadamente 5 minutos (a janela de estabilização padrão), as réplicas começarão a reduzir de volta para 1. Este delay previne scale-down prematuro em caso de carga intermitente.

Pressione `Ctrl+C` para parar de observar.

**Passo 7 — Inspecionar eventos do HPA:**

```bash
kubectl describe hpa php-apache
```

A seção de eventos mostra cada decisão de escalonamento — quando escalou, por quê, e quando reduziu.

### Lab 4: Setup Rápido de Prometheus e Grafana (Opcional)

Este lab requer Helm. Se você completou o Capítulo 10, já o tem instalado.

**Passo 1 — Adicionar o repositório Helm da comunidade Prometheus:**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

**Passo 2 — Instalar o kube-prometheus-stack:**

```bash
helm install kube-prom prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.resources.requests.memory=256Mi \
  --set prometheus.prometheusSpec.resources.requests.cpu=100m
```

Isso instala Prometheus, Grafana, Alertmanager, node-exporter e kube-state-metrics. Leva alguns minutos.

**Passo 3 — Aguardar todos os pods ficarem prontos:**

```bash
kubectl wait --for=condition=Ready pods --all -n monitoring --timeout=300s
kubectl get pods -n monitoring
```

**Passo 4 — Obter a senha de admin do Grafana:**

```bash
kubectl get secret kube-prom-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

O usuário padrão é `admin`.

**Passo 5 — Port-forward para acessar o Grafana:**

```bash
kubectl port-forward -n monitoring svc/kube-prom-grafana 3000:80
```

Abra seu navegador em [http://localhost:3000](http://localhost:3000) e faça login com `admin` e a senha do Passo 4.

**Passo 6 — Explorar os dashboards pré-configurados:**

Navegue até **Dashboards** na barra lateral esquerda. O kube-prometheus-stack vem com dezenas de dashboards pré-configurados:

- **Kubernetes / Compute Resources / Cluster** — Uso de CPU e memória em todo o cluster
- **Kubernetes / Compute Resources / Namespace (Pods)** — Uso de recursos por pod
- **Kubernetes / Compute Resources / Node (Pods)** — Detalhamento no nível do node
- **Node Exporter / Nodes** — Métricas de nível de SO dos nodes (disco, rede, CPU)

Estes dashboards dão a visibilidade histórica que o `kubectl top` não pode — você pode ver tendências, picos e anomalias ao longo do tempo.

Pressione `Ctrl+C` para parar o port-forward quando terminar.

### Limpeza

```bash
# Deletar recursos do HPA
kubectl delete hpa php-apache --ignore-not-found
kubectl delete deployment php-apache --ignore-not-found
kubectl delete svc php-apache --ignore-not-found
kubectl delete pod load-generator --ignore-not-found

# Deletar Prometheus/Grafana (se instalado)
helm uninstall kube-prom -n monitoring 2>/dev/null
kubectl delete namespace monitoring --ignore-not-found

# Deletar o cluster Kind
kind delete cluster --name scaling-lab
```

---

## Principais Conclusões

1. **Sempre defina resource requests e limits.** Pods sem especificação de recursos se tornam BestEffort — primeiros a serem despejados sob pressão e invisíveis para o scheduler no planejamento de capacidade. Requests garantem recursos mínimos; limits impõem uso máximo.

2. **Classes QoS determinam a ordem de despejo.** Pods Guaranteed (requests == limits) são despejados por último. Pods BestEffort (sem recursos especificados) são despejados primeiro. Para workloads críticos, use QoS Guaranteed.

3. **HPA automatiza o escalonamento horizontal.** Ele observa métricas, calcula as réplicas desejadas usando uma fórmula de proporção simples, e ajusta a contagem de réplicas automaticamente. Sem intervenção manual necessária — apenas defina a utilização alvo e os limites.

4. **HPA requer Metrics Server.** Sem o Metrics Server, o HPA não tem dados para trabalhar. No Kind, você precisa instalá-lo separadamente e adicionar a flag `--kubelet-insecure-tls`.

5. **`kubectl top` é apenas tempo real.** Para métricas históricas, tendências e alertas, você precisa do Prometheus (ou de um sistema de monitoramento similar). O chart Helm kube-prometheus-stack é o caminho mais rápido para monitoramento de nível de produção.

6. **Limites de memória são rígidos — OOMKill é imediato.** Não há rede de segurança de swap no Kubernetes. Se um container exceder seu limite de memória, ele é terminado instantaneamente. Defina limites de memória baseados em padrões de uso real, não em suposições.

7. **LimitRange protege namespaces da anarquia de recursos.** Defina requests, limits e máximos padrão por namespace para que mesmo pods implantados sem especificação de recursos recebam padrões sensatos. É o `ulimit` do Kubernetes.

---

## Leitura Complementar

- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Configure Quality of Service for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
- [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [HorizontalPodAutoscaler Walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
- [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)
- [kube-prometheus-stack Helm Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/)

---

**Anterior:** [Capítulo 11 — Segurança](11-security.md)
**Próximo:** [Capítulo 13 — Troubleshooting](13-troubleshooting.md)
