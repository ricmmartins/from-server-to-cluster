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

Por baixo dos panos, um limite de memória configura o `memory.max` do cgroup do container no cgroup v2. O kernel tenta recuperar memória quando uma alocação encontra o limite; se não conseguir atendê-la dentro desse limite, um OOM do cgroup pode encerrar um processo sem shutdown gracioso. Ultrapassar um valor de uso amostrado não é um gatilho instantâneo e determinístico de encerramento.

### Requests vs. Limits

| | Requests | Limits |
|---|---------|--------|
| **O que significa** | Reserva para scheduling e parâmetro de compartilhamento de recursos | Limite de aplicação da restrição |
| **Quem usa** | Scheduler (para posicionamento de pods) | Kubelet/kernel (para enforcement) |
| **Comportamento de CPU** | O node deve ter essa quantidade de CPU disponível para agendar o pod | Container é limitado (throttled) além disso |
| **Comportamento de memória** | O scheduler contabiliza a reserva em relação à memória alocável | Recuperação de memória, seguida de possível OOM do cgroup se as alocações não puderem ser atendidas |
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
        cpu: "100m"        # Guarantee: 10% of a core
        memory: "64Mi"     # Guarantee: 64 MiB
      limits:
        cpu: "200m"        # Ceiling: 20% of a core
        memory: "128Mi"    # Cgroup limit: reclaim, then possible OOM
```

**Regra crítica:** Defina requests realistas para planejar a capacidade e limites de memória baseados em necessidades medidas. Escolha limites de CPU conforme os requisitos de throttling e isolamento do workload. A ausência de requests pode levar a sobrealocação; por si só, ela não estabelece uma ordem absoluta de despejo.

---

## Classes QoS, Despejo e OOM

Com base nos requests e limits de CPU e memória, o Kubernetes atribui a cada pod uma **classe de Quality of Service (QoS)**. A QoS influencia a pontuação de OOM do Linux (`oom_score_adj`), mas **o despejo por pressão de recursos do node feito pelo kubelet é um mecanismo separado**, não uma ordenação absoluta por classe QoS. Para as configurações por container usadas aqui:

| Classe QoS | Condição | Tendência sob Pressão de Memória (Não é Garantia) | Ajuste de OOM do Linux |
|-----------|-----------|-------------------|---------------|
| **Guaranteed** | Requests e limits não nulos de CPU **e** memória, iguais em todos os containers | Geralmente protegido enquanto o uso fica dentro dos requests | `oom_score_adj = -997` |
| **Burstable** | Pelo menos um request ou limit de CPU/memória, mas sem atender a Guaranteed | Depende do uso em relação aos requests e da prioridade do Pod | `oom_score_adj` varia |
| **BestEffort** | Nenhum request ou limit de CPU ou memória em qualquer container | Qualquer uso de memória excede seu request de zero | `oom_score_adj = 1000` |

```yaml
# Guaranteed QoS — requests equal limits
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

```yaml
# Burstable QoS — requests less than limits
resources:
  requests:
    cpu: "100m"
    memory: "64Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

```yaml
# BestEffort QoS — no requests or limits
# (Don't do this in production!)
resources: {}
```

No despejo por pressão de recursos do node, o kubelet primeiro tenta recuperar recursos no próprio node (por exemplo, excluindo imagens não utilizadas sob pressão de disco). Se o despejo ainda for necessário, ele ordena os pods por **se o uso do recurso escasso excede os requests**, depois pela **prioridade do Pod** e, por fim, pelo **uso relativo aos requests**. Pods BestEffort e Burstable que excedem requests de memória podem ser considerados juntos; Guaranteed não significa imunidade. A ordenação sob pressão de disco depende do sistema de arquivos e do uso de armazenamento efêmero, não da QoS de CPU/memória. Sob pressão de PIDs ou inodes, a prioridade importa porque esses recursos não têm requests.

Se a memória se esgotar antes de o kubelet conseguir agir, o OOM killer do kernel seleciona uma vítima com base no uso de memória e no `oom_score_adj`. Um container também pode sofrer **OOM local ao cgroup no seu limite de memória enquanto o node ainda tem memória livre**. Inspecione separadamente motivos de término, eventos de cgroup e pressão no node; consulte os [critérios oficiais de despejo por pressão de recursos](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/#pod-selection-for-kubelet-eviction).

---

## LimitRange: Padrões no Nível do Namespace

O que acontece quando um desenvolvedor faz deploy de um pod sem especificar recursos? Sem valores padrão injetados, ele se torna BestEffort, sem limite próprio de CPU/memória (ainda sujeito às restrições do node e dos cgroups ancestrais) e bastante exposto sob pressão de memória. **LimitRange** define valores padrão de recursos e restrições por namespace.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: default
spec:
  limits:
  - type: Container
    default:              # Applied if no limits are specified
      cpu: "200m"
      memory: "128Mi"
    defaultRequest:       # Applied if no requests are specified
      cpu: "100m"
      memory: "64Mi"
    max:                  # Maximum limits a container can request
      cpu: "1"
      memory: "512Mi"
    min:                  # Minimum requests a container must specify
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
# Imperative (quick)
kubectl autoscale deployment my-app --cpu-percent=50 --min=1 --max=10

# Or declarative (recommended)
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
      stabilizationWindowSeconds: 60       # Wait 60s before scaling up further
      policies:
      - type: Percent
        value: 100                          # Can double replicas in one step
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300      # Wait 5 min before scaling down
      policies:
      - type: Percent
        value: 10                           # Remove at most 10% of replicas per step
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
| `container_memory_working_set_bytes` | Working set estimado: uso menos cache de arquivos inativo; não é um contador de OOM nem o parâmetro exato da decisão de OOM do cgroup |
| `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}` | Último motivo de término observado pelo kube-state-metrics; correlacione com reinícios, eventos e evidências do cgroup/kernel |
| `kube_pod_status_phase` | Quantos pods estão em cada fase (Running, Pending, Failed) |
| `kube_deployment_status_replicas_available` | Se seus deployments têm a contagem esperada de réplicas |
| `node_cpu_seconds_total` | Utilização de CPU no nível do node |
| `node_memory_MemAvailable_bytes` | Memória disponível por node |

### O kube-prometheus-stack

O chart Helm **kube-prometheus-stack** instala Prometheus, Grafana, Alertmanager, node-exporter, kube-state-metrics e dashboards e regras de alerta pré-configurados. Instalar apenas o chart **não** torna o monitoramento pronto para produção; o lab opcional abaixo é deliberadamente efêmero.

---

## Comparação Linux ↔ Kubernetes

| Conceito Linux | Equivalente Kubernetes | Notas |
|---------------|----------------------|-------|
| cgroups `cpu.max` | `resources.limits.cpu` | Teto rígido de CPU |
| cgroups `memory.max` | `resources.limits.memory` | Recuperação de memória e possível OOM do cgroup |
| Prioridade `nice`/`renice` | Prioridade do Pod e requests de recursos (mecanismos diferentes) | Parâmetros da ordenação de despejo sob pressão, não o nice de CPU do Linux |
| OOM killer (`oom_score_adj`) | Seleção para OOM de containers influenciada pela QoS | Diferente do despejo de pods pelo kubelet |
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
> - **Limites de memória são impostos pelo kernel Linux, não por um dashboard de métricas.** O kernel tenta recuperar memória antes de um OOM do cgroup; o encerramento por OOM em si não oferece shutdown gracioso. Não presuma que há swap disponível (o lab não o configura), nem que uma amostra de working set prevê o ponto exato do encerramento. Limites baixos demais podem causar OOM; requests inflados reduzem a capacidade disponível para scheduling. Meça tanto a pressão do workload quanto a do node.

---

## Laboratório Diagnóstico: Escalabilidade e Observabilidade na Prática

### Pré-requisitos

Use **Kind v0.31.0**, **kubectl v1.35.0** e Kubernetes **v1.35.0** com o digest da imagem publicado na [versão do Kind](https://github.com/kubernetes-sigs/kind/releases/tag/v0.31.0). A versão `.0` fixada é uma base de ensino; em produção, use o patch mais recente com suporte da versão minor escolhida. Os comandos exigem Bash (WSL2 serve no Windows), um daemon Docker em execução com containers Linux/cgroup v2, internet e CPU/RAM/disco suficientes no host. O Metrics Server e, principalmente, a stack opcional de monitoramento exigem capacidade adicional além dos pods de aplicação. Confirme um contexto de lab isolado; as saídas esperadas abaixo não são uma alegação de validação em runtime.

Crie um cluster Kind:

```bash
kind create cluster --name scaling-lab \
  --image kindest/node:v1.35.0@sha256:452d707d4862f52530247495d180205e029056831160e22870e37e3f6c1ac31f
```

### Lab 1: Resource Requests, Limits e Classes QoS

**Passo 1 — Fazer deploy de pods com diferentes configurações de recursos:**

```bash
# Guaranteed QoS (requests == limits)
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

# Burstable QoS (requests < limits)
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

# BestEffort QoS (no requests or limits)
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

O Kind não vem com Metrics Server. Fixe a versão **v0.8.1**: a [matriz oficial de compatibilidade da v0.8.1](https://github.com/kubernetes-sigs/metrics-server/blob/v0.8.1/README.md#compatibility-matrix) lista Kubernetes **1.31+**, o que inclui este lab em 1.35. São pré-requisitos o suporte à camada de agregação, autenticação/autorização por webhook do kubelet e conectividade do API server ao Metrics Server e deste aos kubelets.

**Passo 1 — Aplicar o manifesto do Metrics Server:**

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.8.1/components.yaml
```

**Passo 2 — Fazer patch no deployment para adicionar `--kubelet-insecure-tls`:**

Neste lab Kind descartável, ignore a verificação dos certificados de serviço dos kubelets que o Metrics Server não consegue validar. **Somente para testes:** em produção, forneça certificados de serviço confiáveis aos kubelets e mantenha a verificação TLS; não copie esta solução de contorno para uma instalação de produção.

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
# Node-level metrics
kubectl top nodes

# Pod-level metrics (run a pod first if needed)
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

Este lab requer Helm e capacidade livre suficiente no host para toda a stack. É uma **demonstração efêmera, não uma implantação de monitoramento pronta para produção**: nenhum PVC durável do Prometheus está configurado, e excluir/recriar seu pod ou cluster pode perder o histórico. Os pequenos requests abaixo são valores de demonstração, não um dimensionamento de capacidade.

Fixe o chart **92.9.0**. Seu [Chart.yaml oficial versionado](https://github.com/prometheus-community/helm-charts/blob/kube-prometheus-stack-92.9.0/charts/kube-prometheus-stack/Chart.yaml) declara `kubeVersion: ">=1.25.0-0"` (incluindo 1.35) e Prometheus Operator `v0.94.1`. Essa restrição declarada não comprova que este lab foi testado em runtime.

**Passo 1 — Adicionar o repositório Helm da comunidade Prometheus:**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

**Passo 2 — Instalar o kube-prometheus-stack:**

```bash
helm install kube-prom prometheus-community/kube-prometheus-stack \
  --version 92.9.0 \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.resources.requests.memory=256Mi \
  --set prometheus.prometheusSpec.resources.requests.cpu=100m
```

Isso instala Prometheus, Grafana, Alertmanager, node-exporter e kube-state-metrics. O tempo de inicialização depende dos downloads e dos recursos disponíveis. Alguns targets de coleta do control plane do Kind podem ficar indisponíveis sem configurações adicionais de endpoints/TLS; inspecione os targets em vez de considerar todos os dashboards completos.

Antes de adaptar isso para produção:

- Configure `prometheus.prometheusSpec.storageSpec.volumeClaimTemplate` com StorageClass, modo de acesso e tamanho de PVC adequados; planeje persistência para Alertmanager e Grafana quando necessário. Um PVC não é backup nem alta disponibilidade.
- Defina `retention` e `retentionSize` explicitamente em `prometheus.prometheusSpec`. Dimensione o armazenamento considerando taxa de ingestão, cardinalidade, retenção, dados de WAL/head e folga para compactação; monitore o espaço livre. Consulte o [planejamento de armazenamento do Prometheus](https://prometheus.io/docs/prometheus/latest/storage/).
- Teste CPU e memória de coleta/consulta sob carga, defina requests/limits medidos e planeje distribuição entre domínios de falha, replicação ou armazenamento remoto e recuperação.
- Configure e teste os destinatários, roteamento, credenciais e entrega ao plantão do Alertmanager; verifique alertas de lacunas de coleta, pressão de recursos e falhas da stack de monitoramento. Regras padrão sem notificações funcionais não são um serviço de alertas operacional.
- Proteja o acesso às interfaces e APIs (autenticação, TLS, restrições de rede) e teste upgrades, backups e restaurações.

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
# Delete HPA resources
kubectl delete hpa php-apache --ignore-not-found
kubectl delete deployment php-apache --ignore-not-found
kubectl delete svc php-apache --ignore-not-found
kubectl delete pod load-generator --ignore-not-found

# Delete Prometheus/Grafana (if installed)
helm uninstall kube-prom -n monitoring 2>/dev/null
kubectl delete namespace monitoring --ignore-not-found

# Delete the Kind cluster
kind delete cluster --name scaling-lab
```

---

## Principais Conclusões

1. **Defina requests realistas e limites adequados.** Requests orientam o scheduling e o compartilhamento; sua ausência traz risco de sobrealocação. Escolha limites de memória medidos e políticas deliberadas de limites de CPU, em vez de presumir que todos os workloads precisam das mesmas configurações.

2. **QoS não estabelece uma ordem absoluta de despejo.** O kubelet considera uso versus requests, prioridade do Pod e uso relativo para o recurso sob pressão. A QoS influencia a pontuação de OOM do kernel; nem QoS Guaranteed nem um PDB impedem todas as falhas por pressão no node.

3. **HPA automatiza o escalonamento horizontal.** Ele observa métricas, calcula as réplicas desejadas usando uma fórmula de proporção simples, e ajusta a contagem de réplicas automaticamente. Sem intervenção manual necessária — apenas defina a utilização alvo e os limites.

4. **HPA requer Metrics Server.** Sem o Metrics Server, o HPA não tem dados para trabalhar. No Kind, você precisa instalá-lo separadamente e adicionar a flag `--kubelet-insecure-tls`.

5. **`kubectl top` é apenas tempo real.** Para métricas históricas, tendências e alertas, use Prometheus ou um sistema similar. Este lab do chart é efêmero; produção exige armazenamento durável, planejamento de retenção/capacidade e entrega de alertas testada.

6. **A recuperação de memória precede um possível OOM do cgroup.** Working set é uma estimativa, não um contador de OOM nem um limiar exato de encerramento. Correlacione limites, motivos de término, reinícios, eventos de cgroup/kernel e pressão no node ao diagnosticar falhas de memória.

7. **LimitRange protege namespaces da anarquia de recursos.** Defina requests, limits e máximos padrão por namespace para que mesmo pods implantados sem especificação de recursos recebam padrões sensatos. É o `ulimit` do Kubernetes.

---

## Leitura Complementar

- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Configure Quality of Service for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
- [Despejo por Pressão de Recursos do Node](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
- [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [HorizontalPodAutoscaler Walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
- [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)
- [kube-prometheus-stack Helm Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/)

---

**Anterior:** [Capítulo 11 — Segurança](11-security.md)
**Próximo:** [Capítulo 13 — Troubleshooting](13-troubleshooting.md)
