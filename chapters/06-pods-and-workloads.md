# Capítulo 6: Pods e Workloads

*"Um único processo é frágil. Um sistema que gerencia processos — isso é infraestrutura."*

---

## De Processos a Pods

Em um servidor Linux, tudo se resume a processos. Você os inicia, monitora, reinicia quando travam e os agenda para rodar em horários específicos. Você tem `systemd` para serviços de longa duração, `cron` para tarefas agendadas e `at` para trabalhos únicos. Todo o sistema é uma coleção cuidadosamente orquestrada de processos.

O Kubernetes tem as mesmas necessidades — mas em uma escala muito maior. Em vez de gerenciar processos em uma máquina, você está gerenciando workloads através de um cluster. E assim como o Linux tem ferramentas diferentes para diferentes tipos de gerenciamento de processos (`systemd` para daemons, `cron` para agendamentos, `systemctl` para saúde de serviços), o Kubernetes tem diferentes tipos de recursos para diferentes tipos de workloads.

Este capítulo é a taxonomia desses recursos de workload. Pense nele como o capítulo "deep dive no systemd" — uma vez que você entenda como cada controller de workload funciona, saberá qual usar em cada situação.

---

## Pods: A Unidade Atômica

Um **Pod** é a menor unidade implantável no Kubernetes. Não é um container — é um invólucro em torno de um ou mais containers que compartilham rede e armazenamento.

Se processos Linux são átomos, um Pod é uma molécula: um ou mais processos fortemente acoplados (containers) que precisam viver e morrer juntos.

### Por Que Não Executar Apenas Containers?

Você pode se perguntar: se o Docker já executa containers, por que precisamos de Pods?

Porque alguns workloads precisam de múltiplos processos que compartilham recursos:

- **Namespace de rede compartilhado** — Todos os containers em um Pod compartilham o mesmo endereço IP e espaço de portas. Eles se comunicam via `localhost`, assim como processos na mesma máquina Linux.
- **Volumes de armazenamento compartilhados** — Containers em um Pod podem montar os mesmos volumes, permitindo compartilhar arquivos sem overhead de rede.
- **Co-scheduling** — Todos os containers em um Pod têm garantia de rodar no mesmo node. Sem cenários de split-brain.

### Pods de Container Único (O Caso Comum)

A maioria dos Pods roda um único container. Este é o padrão standard para servidores web, APIs, workers e a maioria dos microsserviços:

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

### Pods Multi-Container

Quando você precisa de processos fortemente acoplados, Pods multi-container fazem sentido. Os padrões mais comuns:

**Sidecar containers** — Um auxiliar que roda ao lado da aplicação principal. Exemplos: enviadores de log, proxies de service mesh, terminação TLS. Sidecar containers nativos (usando `initContainers` com `restartPolicy: Always`) foram introduzidos como alpha na v1.28, beta na v1.29 e graduaram para estável (GA) na v1.31. Eles garantem que o sidecar inicie antes e pare depois do container principal.

**Init containers** — Executam até a conclusão *antes* que os containers principais iniciem. Usados para tarefas de setup: esperar que um banco de dados esteja pronto, popular um volume compartilhado, executar migrações.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
  - name: init-db-check
    image: busybox:1.36
    command: ['sh', '-c', 'until nslookup db-service; do echo waiting for db; sleep 2; done']
  containers:
  - name: app
    image: nginx:1.27
    ports:
    - containerPort: 80
```

### Ciclo de Vida do Pod

Um Pod passa por várias fases:

| Fase | Descrição |
|-------|-------------|
| `Pending` | Aceito pelo cluster, mas containers ainda não rodando (baixando imagem, scheduling) |
| `Running` | Pelo menos um container está rodando |
| `Succeeded` | Todos os containers terminaram com sucesso (código de saída 0) |
| `Failed` | Pelo menos um container terminou com erro |
| `Unknown` | Estado do Pod não pode ser determinado (geralmente um problema de comunicação com o node) |

---

## Deployments e ReplicaSets

Você quase nunca cria Pods diretamente em produção. Em vez disso, usa **Deployments**, que gerenciam Pods através de **ReplicaSets**.

Pense nisso como uma hierarquia:

```
Deployment → gerencia → ReplicaSet → gerencia → Pods
```

### ReplicaSets: Garantindo N Réplicas

O trabalho de um ReplicaSet é simples: garantir que exatamente N cópias de um Pod estejam rodando o tempo todo. Se um Pod morre, o ReplicaSet cria um novo. Se há muitos, ele elimina os extras.

Este é o equivalente Kubernetes do `Restart=always` no systemd — mas em vez de reiniciar um processo, ele mantém uma frota.

Você raramente cria ReplicaSets diretamente. Deployments os criam e gerenciam para você.

### Deployments: Rolling Updates em Cima de ReplicaSets

Um **Deployment** adiciona gerenciamento de atualizações em cima de ReplicaSets. Quando você muda o template do Pod (nova imagem, nova configuração), o Deployment cria um *novo* ReplicaSet e gradualmente transfere o tráfego do antigo para o novo.

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
```

Campos-chave:
- **`replicas: 3`** — Manter 3 Pods o tempo todo
- **`selector.matchLabels`** — Como o Deployment encontra seus Pods (deve corresponder aos labels do template de Pod)
- **`template`** — A especificação de Pod usada para cada réplica

---

## DaemonSets: Um Pod Por Node

Um **DaemonSet** garante que todo node (ou um subconjunto de nodes) rode exatamente uma cópia de um Pod. Quando um novo node entra no cluster, o DaemonSet automaticamente agenda um Pod nele. Quando um node é removido, o Pod é coletado pelo garbage collector.

Este é o equivalente Kubernetes de um daemon de sistema — algo como `syslog`, `node_exporter`, ou `filebeat` que você instala em todo servidor.

Casos de uso comuns:
- **Coleta de logs** — Fluentd ou Filebeat em todo node
- **Monitoramento** — Prometheus node exporter em todo node
- **Armazenamento** — Plugins CSI de node
- **Rede** — Plugins CNI, o próprio kube-proxy roda como DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  labels:
    app: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      containers:
      - name: log-collector
        image: busybox:1.36
        command: ['sh', '-c', 'tail -f /var/log/syslog || tail -f /var/log/messages || sleep infinity']
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

---

## StatefulSets: Identidade Estável para Workloads Stateful

Alguns workloads não podem ser tratados como gado intercambiável. Bancos de dados, sistemas distribuídos (ZooKeeper, Kafka, etcd) e qualquer aplicação que precisa de **identidade estável** requer um **StatefulSet**.

StatefulSets fornecem garantias que Deployments não oferecem:

| Garantia | O Que Significa |
|-----------|---------------|
| **Hostname estável** | Pods recebem nomes previsíveis: `web-0`, `web-1`, `web-2` (não sufixos aleatórios) |
| **Armazenamento persistente estável** | Cada Pod recebe seu próprio PersistentVolumeClaim que persiste entre reschedulings |
| **Deploy ordenado** | Pods são criados em ordem (0, 1, 2) e terminados em ordem reversa (2, 1, 0) |
| **Rolling updates ordenados** | Atualizações prosseguem um Pod por vez, em ordem ordinal reversa |

Pense nisso como servidores de banco de dados nomeados no Linux: você não tem "algum servidor de banco de dados" — você tem `db-primary`, `db-replica-1`, `db-replica-2`, cada um com seu próprio diretório de dados em um disco dedicado.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db-headless
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: db
        image: postgres:16
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

O campo `volumeClaimTemplates` é exclusivo de StatefulSets — ele cria um PersistentVolumeClaim separado para cada Pod (`data-db-0`, `data-db-1`, `data-db-2`). Mesmo se um Pod for deletado e recriado, ele se reconecta ao mesmo volume.

---

## Jobs e CronJobs: Workloads de Execução até Conclusão

Nem todo workload roda eternamente. Às vezes você precisa executar uma tarefa uma vez, ou em um agendamento.

### Jobs: Execução Única

Um **Job** cria um ou mais Pods e garante que eles rodem até conclusão bem-sucedida. Se um Pod falha, o Job controller cria um novo (até um `backoffLimit` configurável).

Este é o equivalente Kubernetes do comando `at` — executar algo uma vez e reportar sucesso ou falha.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  template:
    spec:
      containers:
      - name: hello
        image: busybox:1.36
        command: ["echo", "Hello from Job"]
      restartPolicy: Never
  backoffLimit: 4
```

Pontos-chave:
- **`restartPolicy: Never`** ou **`OnFailure`** — Jobs não podem usar `Always` (para isso servem Deployments)
- **`backoffLimit`** — Máximo de tentativas antes de marcar o Job como falho
- **Paralelismo** — Jobs podem rodar múltiplos Pods em paralelo com `spec.parallelism` e `spec.completions`

### CronJobs: Execução Agendada

Um **CronJob** cria Jobs em um agendamento recorrente, usando a mesma sintaxe cron que você já conhece do Linux:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.36
            command:
            - /bin/sh
            - -c
            - date; echo "Hello from CronJob"
          restartPolicy: OnFailure
```

O campo `schedule` usa o formato cron padrão: `minuto hora dia-do-mês mês dia-da-semana`. O exemplo acima roda a cada minuto.

---

## Probes: Health Checks para Containers

No Linux, o systemd pode verificar se um serviço está saudável usando `ExecStartPre`, temporizadores watchdog ou scripts de health check personalizados. O Kubernetes tem um sistema mais granular com três tipos de probes:

| Probe | Propósito | O Que Acontece em Caso de Falha |
|-------|---------|------------------------|
| **Liveness** | O container está vivo? | Container é **reiniciado** |
| **Readiness** | O container está pronto para servir tráfego? | Pod é **removido dos endpoints do Service** (sem tráfego) |
| **Startup** | O container terminou de iniciar? | Probes de liveness/readiness são **pausados** até o startup ter sucesso |

### Mecanismos de Probe

Cada probe pode usar um de quatro mecanismos:

- **`httpGet`** — Faz uma requisição HTTP GET. Sucesso = código de resposta 200-399
- **`tcpSocket`** — Abre uma conexão TCP. Sucesso = porta está aberta
- **`exec`** — Executa um comando dentro do container. Sucesso = código de saída 0
- **`grpc`** — Executa um health check gRPC (Kubernetes v1.27+, estável)

### Exemplo: HTTP Probes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-probed
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 2
      periodSeconds: 5
    startupProbe:
      httpGet:
        path: /
        port: 80
      failureThreshold: 30
      periodSeconds: 2
```

Parâmetros-chave de temporização:
- **`initialDelaySeconds`** — Esperar antes do primeiro probe
- **`periodSeconds`** — Com que frequência executar o probe
- **`failureThreshold`** — Quantas falhas consecutivas antes de tomar ação
- **`successThreshold`** — Quantos sucessos consecutivos para ser considerado saudável (padrão 1; para probes de readiness, pode ser definido mais alto)

---

## Rolling Updates e Rollbacks

Deployments suportam **atualizações sem downtime** através de rolling updates. Quando você muda o template do Pod (nova imagem, nova variável de ambiente, nova configuração), o Deployment cria um novo ReplicaSet e gradualmente transfere os Pods:

```
Old ReplicaSet (3 pods) ──────→ New ReplicaSet (3 pods)
   ↓ scale down                    ↑ scale up
   3 → 2 → 1 → 0                  0 → 1 → 2 → 3
```

### Controlando o Rollout

Dois parâmetros-chave na seção `strategy` do Deployment:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

- **`maxSurge`** — Quantos Pods extras podem existir acima da contagem desejada durante uma atualização (número absoluto ou porcentagem). `maxSurge: 1` significa que você pode temporariamente ter 4 Pods quando `replicas: 3`.
- **`maxUnavailable`** — Quantos Pods podem estar indisponíveis durante a atualização. `maxUnavailable: 0` significa que todos os 3 Pods devem estar sempre rodando — a opção mais segura, mas o rollout mais lento.

### Rollback

Toda vez que você atualiza um Deployment, o Kubernetes mantém o ReplicaSet antigo (escalado para 0). Este é o seu histórico de rollback. Você pode desfazer uma atualização instantaneamente:

```bash
# Desfazer a última atualização
kubectl rollout undo deployment/nginx

# Desfazer para uma revisão específica
kubectl rollout undo deployment/nginx --to-revision=2

# Verificar histórico de rollout
kubectl rollout history deployment/nginx
```

---

## Pod Disruption Budgets (PDB)

Rolling updates são disrupções voluntárias — você escolheu fazê-las. Mas disrupções também acontecem durante drains de node (para manutenção ou upgrades), autoscaling de cluster e outras operações administrativas.

Um **PodDisruptionBudget** diz ao Kubernetes: "Durante disrupções voluntárias, nunca deixe menos que N Pods disponíveis."

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: nginx
```

Com este PDB e 3 réplicas, o Kubernetes só fará drain de um Pod por vez do Deployment nginx. Se drenar um node violaria o budget, a operação de drain bloqueia até ser seguro.

Você pode especificar:
- **`minAvailable`** — Mínimo de Pods que devem continuar rodando (número absoluto ou porcentagem)
- **`maxUnavailable`** — Máximo de Pods que podem estar fora de uma vez (número absoluto ou porcentagem)

---

## Comparação Linux ↔ Kubernetes

| Conceito Linux | Workload K8s | Caso de Uso |
|---|---|---|
| Processo / grupo de processos | Pod | Unidade de execução atômica — um ou mais containers compartilhando rede e armazenamento |
| Serviço systemd (`Restart=always`) | Deployment | Aplicações de longa duração, escaláveis com rolling updates |
| Daemon de sistema (`/usr/lib/systemd/system/`) | DaemonSet | Agentes por node (monitoramento, logging, rede) |
| Serviços nomeados (`db-primary`, `db-replica`) | StatefulSet | Bancos de dados, sistemas distribuídos que precisam de identidade estável |
| cron job (`crontab -e`) | CronJob | Tarefas batch agendadas |
| Comando `at` | Job | Execução batch única |
| Health checks do systemd (`ExecStartPre`, watchdog) | Probes | Detecção de liveness, readiness e startup |

---

> ### ⚠️ Onde a Analogia com Linux Quebra
>
> **Pods são efêmeros — verdadeiramente efêmeros.** No Linux, um processo mantém seu PID e memória até ser encerrado ou a máquina reiniciar. Um Pod Kubernetes pode ser destruído e recriado a qualquer momento — em um node diferente, com um IP diferente, com um sistema de arquivos limpo. Pods são gado, não animais de estimação. Nunca armazene estado dentro de um Pod e espere que ele sobreviva.
>
> **Um rollback de Deployment não "faz downgrade" de nada.** No Linux, `yum downgrade nginx` substitui o binário in-place. No Kubernetes, `kubectl rollout undo` cria Pods inteiramente novos rodando a imagem antiga. Os Pods antigos se foram para sempre — não existe downgrade in-place. É mais como "fazer deploy da versão anterior" do que "desfazer a atualização."
>
> **StatefulSets dão hostnames estáveis, NÃO IPs estáveis.** Um Pod chamado `db-0` sempre será acessível em `db-0.db-headless.default.svc.cluster.local` — mas seu endereço IP muda toda vez que é reagendado. A identidade estável é DNS, não endereço de rede. Isso é deliberado: descoberta baseada em DNS é mais resiliente do que IPs hardcoded.

---

## Laboratório Diagnóstico: Trabalhando com Workloads

### Pré-requisitos

Certifique-se de que seu cluster Kind do Capítulo 5 está rodando:

```bash
kind get clusters
```

Se não, crie um:

```bash
kind create cluster --name lab-cluster --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF
```

---

### Lab 1: Crie um Pod Manualmente

**Passo 1 — Crie um arquivo YAML de Pod:**

```bash
cat <<'EOF' > pod.yaml
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
EOF
```

**Passo 2 — Aplique e inspecione:**

```bash
kubectl apply -f pod.yaml
```

```bash
kubectl get pods
```

Saída esperada:

```
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          10s
```

**Passo 3 — Descreva o Pod para ver eventos e detalhes:**

```bash
kubectl describe pod nginx
```

Procure a seção `Events` no final — você verá o scheduler atribuindo o Pod a um node, o kubelet baixando a imagem e o container iniciando.

**Passo 4 — Exec no Pod:**

```bash
kubectl exec -it nginx -- /bin/sh
```

Dentro do container, tente:

```bash
hostname
cat /etc/os-release
curl localhost:80
exit
```

**Passo 5 — Limpeza:**

```bash
kubectl delete pod nginx
```

---

### Lab 2: Deployments com Rolling Updates

**Passo 1 — Crie um Deployment com nginx:1.26:**

```bash
cat <<'EOF' > deployment.yaml
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
        image: nginx:1.26
        ports:
        - containerPort: 80
EOF
```

```bash
kubectl apply -f deployment.yaml
```

```bash
kubectl get pods -l app=nginx
```

Você deve ver 3 Pods rodando.

**Passo 2 — Atualize a imagem para acionar um rolling update:**

```bash
kubectl set image deployment/nginx nginx=nginx:1.27
```

**Passo 3 — Observe o rollout:**

```bash
kubectl rollout status deployment/nginx
```

Saída esperada:

```
Waiting for deployment "nginx" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
deployment "nginx" successfully rolled out
```

**Passo 4 — Verifique o histórico de rollout:**

```bash
kubectl rollout history deployment/nginx
```

**Passo 5 — Faça rollback para a versão anterior:**

```bash
kubectl rollout undo deployment/nginx
```

Verifique o rollback:

```bash
kubectl describe deployment nginx | grep Image
```

Você deve ver `nginx:1.26` novamente.

**Passo 6 — Limpeza:**

```bash
kubectl delete deployment nginx
```

---

### Lab 3: DaemonSet — Um Pod Por Node

**Passo 1 — Crie um DaemonSet:**

```bash
cat <<'EOF' > daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: collector
        image: busybox:1.36
        command: ['sh', '-c', 'while true; do echo "$(date) - collecting logs from $(hostname)"; sleep 60; done']
EOF
```

```bash
kubectl apply -f daemonset.yaml
```

**Passo 2 — Verifique um Pod por node:**

```bash
kubectl get pods -l app=log-collector -o wide
```

Você deve ver um Pod em cada node (incluindo o control plane, já que adicionamos uma toleration para ele):

```
NAME                  READY   STATUS    RESTARTS   AGE   IP           NODE
log-collector-abc12   1/1     Running   0          10s   10.244.0.5   lab-cluster-control-plane
log-collector-def34   1/1     Running   0          10s   10.244.1.3   lab-cluster-worker
log-collector-ghi56   1/1     Running   0          10s   10.244.2.4   lab-cluster-worker2
```

**Passo 3 — Verifique logs de um dos Pods:**

```bash
kubectl logs -l app=log-collector --tail=5
```

**Passo 4 — Limpeza:**

```bash
kubectl delete daemonset log-collector
```

---

### Lab 4: Job e CronJob

**Passo 1 — Crie um Job:**

```bash
cat <<'EOF' > job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  template:
    spec:
      containers:
      - name: hello
        image: busybox:1.36
        command: ["echo", "Hello from Job"]
      restartPolicy: Never
  backoffLimit: 4
EOF
```

```bash
kubectl apply -f job.yaml
```

**Passo 2 — Observe o Job completar:**

```bash
kubectl get jobs
```

Saída esperada:

```
NAME        COMPLETIONS   DURATION   AGE
hello-job   1/1           3s         10s
```

Verifique a saída:

```bash
kubectl logs job/hello-job
```

```
Hello from Job
```

**Passo 3 — Crie um CronJob:**

```bash
cat <<'EOF' > cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.36
            command:
            - /bin/sh
            - -c
            - date; echo "Hello from CronJob"
          restartPolicy: OnFailure
EOF
```

```bash
kubectl apply -f cronjob.yaml
```

**Passo 4 — Espere cerca de 60-90 segundos, depois verifique:**

```bash
kubectl get cronjobs
kubectl get jobs --watch
```

Você deve ver um novo Job criado a cada minuto. Verifique a saída do Job mais recente:

```bash
kubectl logs job/$(kubectl get jobs --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
```

**Passo 5 — Limpeza:**

```bash
kubectl delete cronjob hello-cron
kubectl delete job hello-job
```

---

### Lab 5: Probes em Ação

**Passo 1 — Crie um Pod com um liveness probe que vai falhar:**

Este Pod cria um arquivo `/tmp/healthy` ao iniciar, depois o remove após 30 segundos — simulando uma aplicação que se torna não saudável:

```bash
cat <<'EOF' > probe-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-test
  labels:
    test: liveness
spec:
  containers:
  - name: liveness
    image: busybox:1.36
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
EOF
```

```bash
kubectl apply -f probe-test.yaml
```

**Passo 2 — Observe o Pod:**

```bash
kubectl get pod liveness-test --watch
```

Nos primeiros ~30 segundos, o Pod está saudável. Depois que o arquivo é removido, o liveness probe falha. Após 3 falhas consecutivas (15 segundos), o kubelet reinicia o container.

Você verá `RESTARTS` incrementar:

```
NAME            READY   STATUS    RESTARTS   AGE
liveness-test   1/1     Running   0          30s
liveness-test   1/1     Running   1 (2s ago) 52s
```

Pressione `Ctrl+C` para parar de observar.

**Passo 3 — Inspecione os eventos:**

```bash
kubectl describe pod liveness-test
```

Na seção Events, você verá:

```
Warning  Unhealthy  ...  Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
Normal   Killing    ...  Container liveness failed liveness probe, will be restarted
```

**Passo 4 — Crie um Deployment com readiness probes:**

```bash
cat <<'EOF' > readiness-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-probed
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-probed
  template:
    metadata:
      labels:
        app: nginx-probed
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 2
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
EOF
```

```bash
kubectl apply -f readiness-deploy.yaml
```

```bash
kubectl get pods -l app=nginx-probed
```

Ambos os Pods devem mostrar `1/1 READY` — significando que o readiness probe está passando e eles estão recebendo tráfego.

**Passo 5 — Limpeza:**

```bash
kubectl delete pod liveness-test
kubectl delete deployment nginx-probed
```

Limpe os arquivos YAML do lab:

```bash
rm -f pod.yaml deployment.yaml daemonset.yaml job.yaml cronjob.yaml probe-test.yaml readiness-deploy.yaml
```

---

## Pontos-Chave

1. **Um Pod é um ou mais containers compartilhando rede e armazenamento — é a unidade atômica do Kubernetes.** A maioria dos Pods roda um único container, mas padrões multi-container (sidecar, init) existem para processos fortemente acoplados.

2. **Nunca crie bare Pods em produção.** Sempre use um controller (Deployment, DaemonSet, StatefulSet, Job) que gerencia o ciclo de vida do Pod — bare Pods não serão reagendados se um node falhar.

3. **Deployments são sua escolha padrão para workloads stateless.** Eles lidam com replicação, rolling updates e rollbacks. Por baixo dos panos, eles gerenciam ReplicaSets.

4. **DaemonSets rodam um Pod por node — perfeito para agentes de infraestrutura.** Coletores de log, agentes de monitoramento e plugins de nível de node pertencem aqui.

5. **StatefulSets são para workloads que precisam de identidade estável.** Hostnames previsíveis (`pod-0`, `pod-1`), armazenamento persistente dedicado e startup/shutdown ordenados. Use-os para bancos de dados e sistemas distribuídos.

6. **Probes são seu sistema de health check em três camadas.** Liveness reinicia containers quebrados. Readiness controla fluxo de tráfego. Startup dá tempo para aplicações lentas inicializarem. Configure-os para todo workload em produção.

7. **Pod Disruption Budgets protegem suas aplicações durante manutenção planejada.** Sem um PDB, um `kubectl drain` pode derrubar todas as suas réplicas simultaneamente.

---

## Leitura Adicional

- [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [DaemonSets](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
- [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Disruptions (Pod Disruption Budgets)](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)

---

**Anterior:** [Capítulo 5 — Seu Primeiro Cluster](05-your-first-cluster.md)
**Próximo:** [Capítulo 7 — Networking](07-networking.md)
