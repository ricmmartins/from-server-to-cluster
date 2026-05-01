# Capítulo 4: Como o Kubernetes Pensa

*"Você não gerencia mais servidores. Você descreve intenções, e controllers as tornam realidade."*

---

## A Maior Mudança Mental Que Você Vai Fazer

Se existe um capítulo neste livro que você deve ler duas vezes, é este.

Em um servidor Linux, você está acostumado a estar no controle. Você digita um comando, algo acontece, e você vê o resultado. Instalar o nginx? `apt install nginx`. Iniciar? `systemctl start nginx`. Ele travou? Coloque `Restart=always` na unit do systemd. Você é o operador, e a máquina faz o que você diz, quando você diz.

O Kubernetes inverte isso completamente. Você não *faz* coisas — você *declara* coisas. Você escreve um documento (um manifesto YAML) que diz "eu quero três réplicas do nginx rodando, cada uma com 256MB de memória, exposta na porta 80." Então você entrega esse documento ao API server e vai embora. O Kubernetes cuida do resto.

Isso não é apenas uma diferença sintática. É uma relação fundamentalmente diferente entre você e o sistema. No Linux, você é o mecânico com as mãos na massa, debaixo do capô. No Kubernetes, você é o arquiteto que desenha a planta e confia na equipe de construção (controllers) para construí-la — e para *mantê-la* construída, para sempre.

Vamos detalhar exatamente como o Kubernetes pensa.

---

## Desired State vs. Actual State

Todo objeto no Kubernetes tem dois estados:

- **Desired state** (estado desejado) — o que você pediu (definido no seu manifesto YAML, armazenado no etcd)
- **Actual state** (estado atual) — o que realmente está acontecendo agora no cluster

O propósito inteiro do Kubernetes é continuamente fazer o estado atual corresponder ao estado desejado. Este é o conceito mais importante de todo o sistema.

### Uma Comparação com Linux

Pense em uma unit do systemd com `Restart=always`:

```ini
[Service]
ExecStart=/usr/bin/nginx
Restart=always
RestartSec=5
```

Se o nginx travar, o systemd percebe e o reinicia. O "desired state" é "nginx deveria estar rodando." O "actual state" pode temporariamente ser "nginx está morto." O systemd reconcilia a diferença.

O Kubernetes faz isso para *tudo*, não apenas para reiniciar processos:

- Quer 3 réplicas? Um controller garante que sempre haja exatamente 3.
- Quer um Service roteando para pods saudáveis? Um controller atualiza os endpoints quando pods vão e vêm.
- Quer um volume montado? Um controller o anexa ao node correto.
- Quer um node drenado? Um controller remove pods e respeita disruption budgets.

### Declarando o Desired State

Na prática, o desired state se parece com isso:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-frontend
  template:
    metadata:
      labels:
        app: web-frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
```

Você aplica isso com `kubectl apply -f deployment.yaml`, e pronto. Você não diz ao Kubernetes *em qual* node executar, *como* baixar a imagem, ou *o que* fazer se um pod travar. Você apenas diz o que quer. O sistema cuida do resto.

---

## O Reconciliation Loop

Todo controller no Kubernetes executa o mesmo algoritmo básico, eternamente:

```
1. WATCH — Observe o estado atual dos objetos pelos quais você é responsável
2. DIFF  — Compare o estado atual com o estado desejado
3. ACT   — Tome ação para fechar a lacuna
4. REPEAT
```

Isso é chamado de **reconciliation loop** (ou control loop), e é o batimento cardíaco do sistema.

### A Analogia do Termostato

A própria documentação do Kubernetes usa esta analogia: pense em um termostato. Você o configura para 22°C (desired state). O termostato continuamente lê a temperatura do ambiente (actual state). Se estiver muito frio, ele liga o aquecedor. Se estiver muito quente, liga o ar-condicionado. Ele nunca para de verificar.

Todo controller do Kubernetes é um termostato para o seu tipo de recurso.

### O Que Acontece Quando Você Cria um Deployment

Vamos rastrear o que acontece quando você executa `kubectl apply` no deployment acima:

1. `kubectl` envia o manifesto para o **API server**
2. O API server valida e armazena no **etcd**
3. O **Deployment controller** percebe um novo objeto Deployment
4. Ele cria um **ReplicaSet** para gerenciar as 3 réplicas desejadas
5. O **ReplicaSet controller** percebe o novo ReplicaSet
6. Ele cria 3 objetos **Pod** (desired state: esses pods devem existir)
7. O **Scheduler** percebe 3 Pods não agendados
8. Ele atribui cada Pod a um node com base nos recursos disponíveis
9. O **kubelet** em cada node selecionado percebe novos Pods atribuídos a ele
10. Cada kubelet baixa a imagem do container e inicia o container

São pelo menos cinco controllers diferentes, cada um fazendo um pequeno trabalho, encadeados pelo padrão de reconciliação. Nenhum deles conhece os outros. Eles apenas observam mudanças nos objetos com os quais se preocupam e reagem.

### O Que Acontece Quando um Pod Morre

Agora suponha que um node falhe e leve um dos três pods consigo:

1. O **Node controller** detecta que o node não está respondendo (via heartbeats perdidos)
2. Ele marca o node como `NotReady`
3. O **ReplicaSet controller** percebe que agora há apenas 2 pods rodando, mas a contagem desejada é 3
4. Ele cria um novo objeto Pod
5. O **Scheduler** o coloca em um node saudável
6. O **kubelet** o inicia

Você não fez nada. Você não foi acionado. O sistema se auto-curou porque todo controller continuou verificando: "A realidade corresponde à spec?"

---

## Controllers e o Padrão Controller

Um **controller** é um processo que observa o estado do seu cluster através do API server, e então faz mudanças tentando mover o estado atual em direção ao estado desejado.

### Controllers Embutidos

O Kubernetes vem com dezenas de controllers, todos rodando dentro do processo `kube-controller-manager`. Alguns importantes:

| Controller | Observa | Garante |
|------------|---------|---------|
| Deployment controller | Deployments | ReplicaSets corretos existem para cada Deployment |
| ReplicaSet controller | ReplicaSets | O número certo de Pods está rodando |
| Node controller | Nodes | Nodes não saudáveis são detectados e tratados |
| Job controller | Jobs | Pods rodam até a conclusão para workloads batch |
| EndpointSlice controller | Services, Pods | Endpoints de Service apontam para IPs de Pods saudáveis |
| ServiceAccount controller | Namespaces | ServiceAccount padrão existe em cada namespace |

### O Princípio da Responsabilidade Única

Note como cada controller faz *uma coisa*. O Deployment controller não cria Pods diretamente — ele cria ReplicaSets, e então o ReplicaSet controller cria Pods. Esta cadeia de responsabilidade é deliberada. Isso significa:

- Cada controller é simples e testável
- Você pode trocar ou estender peças individuais
- O sistema é resiliente — se um controller está lento, outros continuam funcionando

Este é o mesmo princípio de design que você vê na filosofia Unix: faça uma coisa bem. `grep` encontra texto, `sort` ordena, `uniq` deduplica. Cada ferramenta é simples; o poder vem da composição.

---

## Labels, Selectors e Annotations

Se controllers são os músculos do Kubernetes, labels e selectors são o sistema nervoso. É assim que objetos encontram uns aos outros.

### Labels

Labels são pares chave-valor anexados a qualquer objeto Kubernetes. É como você organiza, categoriza e seleciona recursos.

```yaml
metadata:
  labels:
    app: web-frontend
    env: production
    team: platform
    version: v2.1.0
```

Labels são *livres* — você define quaisquer chaves e valores que façam sentido para sua organização. O Kubernetes não impõe nenhum esquema além de regras básicas de sintaxe (máximo de 63 caracteres para o segmento do nome, caracteres alfanuméricos, hífens, underscores e pontos).

### Selectors

Selectors são consultas contra labels. É como controllers, Services e você (via kubectl) encontram os objetos com os quais se preocupam.

**Selectors baseados em igualdade:**

```bash
# Encontrar todos os pods em produção
kubectl get pods -l env=production

# Encontrar todos os pods que NÃO estão em produção
kubectl get pods -l env!=production
```

**Selectors baseados em conjunto:**

```bash
# Encontrar pods em staging ou produção
kubectl get pods -l 'env in (staging, production)'

# Encontrar pods com um label "team" (qualquer valor)
kubectl get pods -l team

# Encontrar pods sem um label "deprecated"
kubectl get pods -l '!deprecated'
```

### Como Services Encontram Pods

É aqui que labels se tornam essenciais. Um Service não conhece Pods específicos — ele usa um selector para encontrá-los dinamicamente:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-frontend
spec:
  selector:
    app: web-frontend    # "encontre todos os Pods com este label"
  ports:
  - port: 80
    targetPort: 80
```

Quando um novo Pod com `app: web-frontend` aparece, o Service automaticamente o inclui. Quando um Pod com esse label é deletado, é automaticamente removido. Sem registro manual. Sem recarregamento de configuração.

Pense assim: no Linux, você pode configurar um load balancer com uma lista estática de IPs de backend. No Kubernetes, o load balancer (Service) diz "envie tráfego para qualquer coisa com label `app: web-frontend`," e o sistema mantém a lista atualizada em tempo real.

### Annotations

Annotations também são metadados chave-valor, mas com um propósito diferente. Enquanto labels são para *identificação e seleção*, annotations são para *anexar metadados arbitrários não identificadores*.

```yaml
metadata:
  annotations:
    description: "Frontend principal voltado ao cliente"
    git-commit: "a1b2c3d4"
    config.kubernetes.io/managed-by: "kustomize"
```

Diferenças-chave em relação a labels:

- Annotations **não podem** ser usadas em selectors
- Valores de annotations podem ser muito maiores (até 256KB total por objeto)
- São tipicamente usadas por ferramentas, controllers e humanos para fins informativos
- Usos comuns: informações de build, hashes de commits git, configuração de ferramentas, razões de mudanças

Pense em labels como o sistema de arquivamento que você usa para encontrar documentos, e annotations como post-its nos documentos com contexto extra.

---

## Namespaces

Namespaces do Kubernetes fornecem uma maneira de dividir recursos do cluster em grupos logicamente isolados. São uma das primeiras coisas que confundem profissionais de Linux, porque o nome colide com um conceito Linux muito diferente.

### Não São Linux Kernel Namespaces

Vamos ser cristalinos: **Kubernetes namespaces NÃO são a mesma coisa que Linux kernel namespaces.**

| Aspecto | Linux Kernel Namespaces | Kubernetes Namespaces |
|---------|------------------------|-----------------------|
| Propósito | Isolamento em nível de processo (PID, rede, mount, etc.) | Agrupamento lógico e controle de acesso |
| Escopo | Máquina única | Cluster inteiro |
| Isolamento | Isolamento rígido (processos não conseguem ver uns aos outros) | Isolamento suave (RBAC, resource quotas, network policies) |
| Mecanismo | Nível de kernel | Nível de API server |

Linux kernel namespaces tornam containers possíveis — eles isolam árvores de processos, pilhas de rede e sistemas de arquivos no nível do kernel. Kubernetes namespaces são um mecanismo de *particionamento lógico* — mais como diretórios em um sistema de arquivos do que fronteiras de isolamento do kernel.

### O Que Namespaces Realmente Fazem

Namespaces fornecem:

1. **Escopo de nomes** — Você pode ter um Pod chamado `nginx` no namespace `staging` e outro Pod chamado `nginx` no namespace `production`. Sem conflito.
2. **Fronteiras de controle de acesso** — Políticas RBAC podem conceder permissões por namespace. "Alice pode fazer deploy em staging mas não em production."
3. **Resource quotas** — Você pode limitar quanto de CPU, memória e quantos objetos um namespace pode consumir.
4. **Configurações padrão** — LimitRanges podem definir requests/limits padrão de recursos para todos os Pods em um namespace.

### Namespaces Padrão

Todo cluster começa com quatro namespaces:

| Namespace | Propósito |
|-----------|---------|
| `default` | Onde seus objetos vão se você não especificar um namespace |
| `kube-system` | Componentes do sistema Kubernetes (API server, scheduler, CoreDNS, etc.) |
| `kube-public` | Recursos publicamente legíveis (raramente usado diretamente) |
| `kube-node-lease` | Leases de heartbeat de nodes para detecção de falhas |

### Trabalhando com Namespaces

```bash
# Listar todos os namespaces
kubectl get namespaces

# Criar um namespace
kubectl create namespace staging

# Deploy em um namespace específico
kubectl apply -f deployment.yaml -n staging

# Listar pods em um namespace específico
kubectl get pods -n staging

# Listar pods em TODOS os namespaces
kubectl get pods -A

# Definir seu namespace padrão (para não precisar digitar -n toda vez)
kubectl config set-context --current --namespace=staging
```

### Quando Usar Namespaces

Bons casos de uso:

- **Por ambiente**: `staging`, `production`, `dev`
- **Por equipe**: `team-platform`, `team-payments`
- **Por aplicação**: `app-frontend`, `app-backend` (em organizações maiores)

Você *não* precisa de namespaces para cada pequena diferença. Diferentes versões da mesma aplicação? Use labels, não namespaces. A documentação oficial é explícita: "Para clusters com poucos a dezenas de usuários, você não deveria precisar criar ou pensar sobre namespaces."

---

## Fundamentos de Scheduling

O **kube-scheduler** é o componente que decide em qual node um Pod roda. Toda vez que um novo Pod é criado sem atribuição de node, o scheduler escolhe um.

No Linux, você pode usar `taskset` para fixar um processo em CPUs específicas, ou `nice` para ajustar a prioridade. O scheduling do Kubernetes é o equivalente em nível de cluster — mas em vez de CPUs em uma máquina, você está escolhendo entre nodes em um cluster.

### Como o Scheduler Decide

O scheduler trabalha em duas fases:

1. **Filtragem** — Eliminar nodes que não podem rodar o Pod (recursos insuficientes, labels errados, taints não tolerados)
2. **Pontuação** — Classificar os nodes restantes e escolher o melhor (mais recursos disponíveis, melhor distribuição, etc.)

### Resource Requests e Limits

Quando você define um Pod, pode especificar quanto de CPU e memória ele precisa:

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "250m"      # 250 millicores = 0.25 CPU
  limits:
    memory: "256Mi"
    cpu: "500m"
```

- **Requests** são o que o scheduler usa para posicionamento. "Este Pod precisa de pelo menos 128Mi de memória e 250m de CPU." O scheduler só coloca o Pod em um node que tenha recursos *não requisitados* suficientes.
- **Limits** são o máximo que o Pod pode consumir. Se um container exceder seu limit de memória, ele sofre OOM-kill. Se exceder seu limit de CPU, ele é limitado (throttled).

Isso mapeia diretamente para cgroups do Linux. Por baixo dos panos, o kubelet traduz requests e limits em configurações de cgroup no node. Requests se tornam reservas de cgroup; limits se tornam limites rígidos de cgroup.

### Node Affinity

Node affinity permite restringir em quais nodes um Pod pode ser agendado, com base em labels do node.

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
```

Isso diz: "Só agende este Pod em nodes com label `disktype=ssd`." É o equivalente Kubernetes do `taskset` do Linux, mas em vez de fixar em CPUs, você está fixando em nodes com características específicas.

Existe também `preferredDuringSchedulingIgnoredDuringExecution`, que é uma preferência suave em vez de um requisito rígido — o scheduler *tentará* honrá-la mas não falhará se não conseguir.

### Taints e Tolerations

Taints são o *oposto* da affinity — elas permitem que um node *repila* Pods.

```bash
# Aplicar taint em um node: "este node é apenas para workloads GPU"
kubectl taint nodes gpu-node-1 workload=gpu:NoSchedule
```

Agora, apenas Pods com uma toleration correspondente podem ser agendados ali:

```yaml
spec:
  tolerations:
  - key: "workload"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
```

Pense nisso como uma área VIP em um evento. A taint é a corda de veludo (mantém todos fora por padrão), e a toleration é o passe VIP (permite que Pods específicos entrem).

Os efeitos de taint incluem:

| Efeito | Comportamento |
|--------|----------|
| `NoSchedule` | Novos Pods sem uma toleration correspondente não serão agendados neste node |
| `PreferNoSchedule` | O scheduler *tentará* evitar este node, mas não é garantido |
| `NoExecute` | Pods existentes sem uma toleration correspondente são removidos; novos não são agendados |

### Topology Spread Constraints

Topology spread constraints garantem que seus Pods sejam distribuídos uniformemente entre domínios de falha (nodes, zonas, regiões):

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web-frontend
```

Isso diz: "A diferença no número de Pods `web-frontend` entre quaisquer duas zonas deve ser no máximo 1." Se a zona A tem 3 Pods e a zona B tem 2, o próximo Pod vai para a zona B.

Não existe um equivalente direto no Linux para isso — é uma preocupação em nível de cluster que não existe em uma única máquina.

---

## Tabela Comparativa Linux ↔ K8s

| Conceito Linux | Equivalente K8s | Diferença-Chave |
|---------------|----------------|----------------|
| `systemd Restart=always` | Reconciliation loop do controller | Controllers do K8s gerenciam entre nodes, não apenas localmente em uma máquina |
| `/etc/systemd/system/*.service` | Manifestos YAML | Desired state declarativo armazenado no etcd, versionado e auditável |
| File labels (contextos SELinux) | Labels + Selectors | Labels são pares chave-valor livres; selectors os consultam dinamicamente |
| Linux kernel namespaces | Namespaces do K8s | Namespaces do K8s são fronteiras lógicas/RBAC, não isolamento de processos em nível de kernel |
| `nice`/`renice`, cgroups | Resource requests/limits | O scheduler usa requests para decisões de posicionamento em nodes; o kubelet aplica limits via cgroups |
| `/etc/hosts`, comentários em arquivos de config | Annotations | Metadados anexados a objetos; chave-valor estruturado mas não consultável para seleção |
| CPU affinity (`taskset`) | Node affinity / nodeSelector | Restringe onde workloads rodam com base em labels de nodes, não IDs de CPU |

---

> **Onde a Analogia com Linux Quebra**
>
> **Consistência eventual, não execução imediata.** Linux é imediato: você executa um comando, ele acontece agora. Kubernetes é eventualmente consistente — você submete um desired state, e controllers trabalham em direção a ele assincronamente. Você pode esperar segundos ou minutos para o sistema convergir. Isso não é um bug; é o design. Sistemas distribuídos *devem* ser eventualmente consistentes.
>
> **O manifesto é a fonte da verdade, não o estado em execução.** Não existe um "SSH e conserte" como escape para operações normais. Se você modificar manualmente um Pod em execução (digamos, instalando um pacote via `kubectl exec`), o controller sobrescreverá suas mudanças na próxima vez que recriar o Pod. O manifesto YAML no controle de versão é a única fonte da verdade. Se não está no manifesto, não existe.
>
> **Labels são planas, não hierárquicas.** Labels não são organizadas como uma hierarquia de sistema de arquivos — são pares chave-valor planos. O poder vem de consultas flexíveis com selectors (baseadas em igualdade, baseadas em conjunto), não de estruturas de diretório aninhadas. Você não pode fazer `labels/env/staging` — você faz `env=staging` e consulta com `-l env=staging`.

---

## Laboratório Diagnóstico: Reconciliação, Labels, Namespaces e Scheduling

Este laboratório demonstra os conceitos centrais deste capítulo usando um cluster Kind.

### Pré-requisitos

Certifique-se de ter um cluster Kind rodando:

```bash
kind create cluster --name chapter04
```

Verifique se está pronto:

```bash
kubectl cluster-info --context kind-chapter04
```

---

### Parte 1: Observe o Reconciliation Loop em Ação

**Passo 1 — Crie um Deployment com 3 réplicas:**

```bash
kubectl create deployment nginx --image=nginx:1.27 --replicas=3
```

**Passo 2 — Verifique que os Pods estão rodando:**

```bash
kubectl get pods -l app=nginx
```

Você deve ver 3 Pods no estado `Running`.

**Passo 3 — Delete um Pod e observe-o voltar:**

```bash
# Escolha um dos nomes de Pod da saída acima
kubectl delete pod $(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
```

Verifique imediatamente:

```bash
kubectl get pods -l app=nginx
```

Você verá um novo Pod com um nome diferente sendo criado (ou já rodando). O ReplicaSet controller percebeu que a contagem caiu para 2 e criou um substituto. Este é o reconciliation loop em ação.

**Passo 4 — Escale para 5 réplicas:**

```bash
kubectl scale deployment nginx --replicas=5
```

Observe os novos Pods aparecerem:

```bash
kubectl get pods -l app=nginx -w
```

Pressione `Ctrl+C` quando todos os 5 estiverem Running.

**Passo 5 — Veja os eventos que contam a história da reconciliação:**

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

Você verá eventos do Deployment controller (escalando), do ReplicaSet controller (criando Pods), do Scheduler (atribuindo Pods a nodes) e do kubelet (baixando imagens, iniciando containers).

---

### Parte 2: Labels e Selectors

**Passo 1 — Inspecione labels existentes:**

```bash
kubectl get pods --show-labels
```

Todo Pod criado pelo Deployment já tem `app=nginx` — é assim que o ReplicaSet encontra seus Pods.

**Passo 2 — Adicione labels personalizados:**

```bash
# Rotule dois pods como staging
POD1=$(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
POD2=$(kubectl get pods -l app=nginx -o jsonpath='{.items[1].metadata.name}')
kubectl label pod $POD1 env=staging
kubectl label pod $POD2 env=staging

# Rotule o restante como production
kubectl label pod $(kubectl get pods -l app=nginx -o jsonpath='{.items[2].metadata.name}') env=production
kubectl label pod $(kubectl get pods -l app=nginx -o jsonpath='{.items[3].metadata.name}') env=production
kubectl label pod $(kubectl get pods -l app=nginx -o jsonpath='{.items[4].metadata.name}') env=production
```

**Passo 3 — Consulte por label:**

```bash
# Todos os pods de staging
kubectl get pods -l env=staging

# Todos os pods de production
kubectl get pods -l env=production

# Todos os pods com um label "env" (qualquer valor)
kubectl get pods -l env

# Pods que são TANTO nginx QUANTO production
kubectl get pods -l app=nginx,env=production
```

**Passo 4 — Veja como um Service usa selectors:**

Crie um Service que seleciona apenas Pods de produção:

```bash
kubectl expose deployment nginx --port=80 --name=nginx-prod --type=ClusterIP --overrides='{"spec":{"selector":{"app":"nginx","env":"production"}}}'
```

Verifique quais Pods o Service selecionou:

```bash
kubectl describe svc nginx-prod | grep Endpoints
```

Você deve ver apenas os IPs dos 3 Pods com label de produção.

Limpe o Service:

```bash
kubectl delete svc nginx-prod
```

---

### Parte 3: Isolamento por Namespace

**Passo 1 — Crie dois namespaces:**

```bash
kubectl create namespace team-alpha
kubectl create namespace team-beta
```

**Passo 2 — Faça deploy da mesma aplicação em ambos:**

```bash
kubectl create deployment web --image=nginx:1.27 -n team-alpha
kubectl create deployment web --image=nginx:1.27 -n team-beta
```

**Passo 3 — Mostre que são independentes:**

```bash
# Cada namespace tem seu próprio deployment "web"
kubectl get deployments -n team-alpha
kubectl get deployments -n team-beta

# Pods têm nomes e IPs diferentes
kubectl get pods -o wide -n team-alpha
kubectl get pods -o wide -n team-beta
```

**Passo 4 — Demonstre o escopo de nomes:**

```bash
# Isso funciona — "web" existe em ambos os namespaces independentemente
kubectl get deployment web -n team-alpha
kubectl get deployment web -n team-beta

# Mas no namespace default, não existe "web"
kubectl get deployment web 2>&1 || true
```

**Passo 5 — Visualize todos os namespaces de uma vez:**

```bash
kubectl get pods -A | grep web
```

Limpe:

```bash
kubectl delete namespace team-alpha
kubectl delete namespace team-beta
```

---

### Parte 4: Decisões de Scheduling

**Passo 1 — Descreva um pod para ver eventos do scheduler:**

```bash
kubectl describe pod $(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
```

Olhe a seção `Events` no final. Você verá entradas como:

```
Successfully assigned default/nginx-xxx to chapter04-control-plane
```

Isso indica que o Scheduler tomou uma decisão de posicionamento e atribuiu o Pod a um node.

**Passo 2 — Crie um Pod com um nodeSelector insatisfazível:**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: picky-pod
spec:
  nodeSelector:
    disktype: nvme
  containers:
  - name: nginx
    image: nginx:1.27
EOF
```

**Passo 3 — Observe o Pod ficar Pending:**

```bash
kubectl get pod picky-pod
```

Você verá `STATUS: Pending`. O scheduler não consegue encontrar um node com `disktype=nvme`.

Verifique os eventos:

```bash
kubectl describe pod picky-pod | tail -5
```

Você verá um aviso como:

```
0/1 nodes are available: 1 node(s) didn't match Pod's node affinity/selector.
```

**Passo 4 — Corrija rotulando o node:**

```bash
# Rotule o node
kubectl label node chapter04-control-plane disktype=nvme

# Observe o Pod ser agendado
kubectl get pod picky-pod -w
```

O Pod deve transicionar de `Pending` para `ContainerCreating` para `Running`.

Pressione `Ctrl+C` quando estiver Running.

---

### Limpeza

```bash
kubectl delete pod picky-pod
kubectl delete deployment nginx
kind delete cluster --name chapter04
```

---

## Pontos-Chave

1. **Kubernetes é um sistema de desired state.** Você declara o que quer em um manifesto, e controllers continuamente reconciliam a realidade para corresponder. Você não diz *como* — você diz *o quê*.

2. **O reconciliation loop (Watch → Diff → Act) é o algoritmo central.** Todo controller executa esse loop eternamente. Quando você internaliza esse padrão, todo o Kubernetes faz sentido.

3. **Controllers seguem o princípio da responsabilidade única.** Cada controller gerencia um tipo de recurso. Comportamentos complexos emergem de cadeias de controllers simples reagindo às mudanças uns dos outros.

4. **Labels e selectors são a cola que conecta objetos.** Services encontram Pods via selectors. ReplicaSets rastreiam seus Pods via selectors. Suas consultas usam selectors. Domine labels, e você domina a organização do Kubernetes.

5. **Kubernetes namespaces NÃO são Linux kernel namespaces.** São partições lógicas para organização, controle de acesso e resource quotas — não isolamento de processos em nível de kernel.

6. **O scheduler usa resource requests para decisões de posicionamento.** Sempre defina resource requests em seus Pods. Sem eles, o scheduler está voando às cegas e você terá problemas de contenção de recursos.

7. **Taints repelem Pods; tolerations são o passe de exceção.** Use taints e tolerations para dedicar nodes a workloads específicos (nodes GPU, nodes de alta memória) enquanto mantém todo o resto longe.

---

## Leitura Adicional

- [Kubernetes Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)
- [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)
- [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Scheduling, Preemption and Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/)
- [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)

---

**Anterior:** [Capítulo 3 — Arquitetura do Kubernetes](03-kubernetes-architecture.md)
**Próximo:** [Capítulo 5 — Seu Primeiro Cluster](05-your-first-cluster.md)
