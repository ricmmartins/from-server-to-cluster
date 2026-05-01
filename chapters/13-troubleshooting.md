# Capítulo 13: Solução de Problemas

*"A primeira regra do debugging: não adivinhe. Observe, formule hipóteses, teste."*

---

## De `/var/log` para `kubectl describe`

Em um servidor Linux, quando algo quebra, sua memória muscular assume o controle. Você verifica o processo: `systemctl status nginx`. Lê os logs: `journalctl -u nginx -f`. Verifica a rede: `ss -tlnp`, `curl localhost:80`. Olha os recursos: `top`, `df -h`. Você sabe onde tudo está, porque tudo está na única máquina à sua frente.

No Kubernetes, os mesmos instintos se aplicam — mas a execução é diferente. O processo que você quer inspecionar pode estar rodando em qualquer um de uma dúzia de nós. Os logs podem desaparecer no momento em que o pod reinicia. A rede não são apenas sockets locais — é um overlay de todo o cluster gerenciado por plugins CNI e regras do kube-proxy. E a "máquina" que você está debugando é o API server, acessado remotamente através do `kubectl`.

Aqui está a boa notícia: o modelo mental é o mesmo. Algo está quebrado. Você precisa descobrir o quê, onde e por quê. A diferença é que sua caixa de ferramentas muda de `systemctl`, `journalctl` e `ss` para `kubectl get`, `kubectl describe` e `kubectl logs`. E a hierarquia de debugging muda de "verificar a máquina" para "verificar o pod, depois o container, depois o nó, depois o cluster."

Este capítulo ensina uma abordagem sistemática para solução de problemas no Kubernetes — da mesma forma que sysadmins Linux experientes debugam servidores, traduzida para o mundo orquestrado.

---

## A Hierarquia de Debugging

Quando algo dá errado no Kubernetes, trabalhe de dentro para fora:

```
Pod → Container → Node → Cluster
```

1. **Nível do Pod:** O pod está rodando? Em que estado ele está? O que os eventos dizem?
2. **Nível do Container:** O container iniciou? O que os logs dizem? Está crashando?
3. **Nível do Node:** O nó está saudável? Tem recursos suficientes? O kubelet está rodando?
4. **Nível do Cluster:** O control plane está operacional? Existem restrições de recursos em todo o cluster?

A maioria dos problemas (90%+) são detectados nos níveis de Pod e Container. Comece por aí.

---

## Comandos Essenciais de Debugging

### `kubectl get pods` — Status Rápido

```bash
kubectl get pods -o wide
```

A coluna STATUS é seu primeiro indicador:

| Status | O Que Significa |
|--------|--------------|
| **Running** | Container(s) iniciou(aram) e não foi(ram) (ainda) terminado(s) |
| **Pending** | Pod aceito mas não agendado ou imagem não baixada |
| **CrashLoopBackOff** | Container crasha repetidamente; K8s recua nos restarts |
| **ImagePullBackOff** | Não consegue baixar a imagem do container |
| **OOMKilled** | Container excedeu seu limite de memória |
| **CreateContainerConfigError** | Referência inválida a ConfigMap/Secret |
| **Evicted** | Nó sob pressão de recursos, pod foi removido |
| **Terminating** | Pod está desligando (preso aqui = problema de finalizer) |
| **Init:Error** | Init container falhou |
| **Completed** | Pod executou até o final (normal para Jobs) |

Preste atenção na coluna **RESTARTS**. Um pod mostrando `Running` com 47 restarts *não* está saudável.

### `kubectl describe pod` — O Comando Mais Útil

```bash
kubectl describe pod <pod-name>
```

Este único comando mostra:
- **Status e Conditions** — por que o pod está no estado atual
- **Events** — a história cronológica do que aconteceu (agendamento, pull, criação, início, encerramento)
- **Estado do container** — estado atual, estado anterior, códigos de saída, contagem de restarts
- **Requests/limits de recursos** — o que o pod solicitou
- **Volumes e mounts** — o que está anexado e onde
- **Atribuição de nó** — onde ele foi alocado (ou por que não foi)

> **Dica profissional:** Sempre leia a seção de **Events** no final. Ela conta a história.

### `kubectl logs` — Saída do Container

```bash
# Logs do container atual
kubectl logs <pod-name>

# Container específico em pod multi-container
kubectl logs <pod-name> -c <container-name>

# Container anterior que crashou (o que existia ANTES do restart atual)
kubectl logs <pod-name> -p

# Acompanhar logs em tempo real
kubectl logs <pod-name> -f

# Últimas 50 linhas
kubectl logs <pod-name> --tail=50

# Logs desde um tempo específico
kubectl logs <pod-name> --since=5m
```

A flag `-p` (previous) é crítica. Quando um pod está em CrashLoopBackOff, o container atual pode ter zero logs (acabou de iniciar). O container *anterior* tem a saída do crash.

### `kubectl get events` — Linha do Tempo do Cluster

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

Events são o log de atividades do cluster. Eles mostram decisões de agendamento, pulls de imagem, inícios de containers, falhas de health check, ações de scaling e erros — tudo com timestamp e ordenável.

```bash
# Events em um namespace específico
kubectl get events -n my-namespace --sort-by=.metadata.creationTimestamp

# Acompanhar events em tempo real
kubectl get events --watch
```

### `kubectl debug` — Containers Efêmeros de Debug

```bash
# Anexar um container de debug a um pod em execução
kubectl debug <pod-name> -it --image=busybox:1.36 --target=<container-name>

# Criar uma cópia de um pod com um container de debug
kubectl debug <pod-name> -it --image=busybox:1.36 --copy-to=debug-pod

# Debugar um nó diretamente
kubectl debug node/<node-name> -it --image=busybox:1.36
```

Containers efêmeros são injetados em um pod existente sem reiniciá-lo. Eles compartilham o namespace de rede do pod, então você pode inspecionar conectividade de rede, verificar montagens de filesystem e executar ferramentas de diagnóstico.

### `kubectl exec` — Shell Interativo

```bash
# Executar um shell em um container em execução
kubectl exec -it <pod-name> -- /bin/sh

# Container específico
kubectl exec -it <pod-name> -c <container-name> -- /bin/bash

# Executar um único comando
kubectl exec <pod-name> -- cat /etc/resolv.conf
```

### `kubectl port-forward` — Testar Conectividade

```bash
# Encaminhar porta local para o pod
kubectl port-forward pod/<pod-name> 8080:80

# Encaminhar para um service
kubectl port-forward svc/<service-name> 8080:80
```

Isso bypassa Services e Ingress inteiramente — útil para confirmar que a aplicação em si funciona antes de investigar camadas de rede.

---

## Estados Comuns de Falha de Pod — Análise Detalhada

### Pending — Não Pode Ser Agendado

O scheduler não consegue encontrar um nó adequado. Causas comuns:

- **Recursos insuficientes:** Nenhum nó tem CPU/memória suficiente para satisfazer os requests
- **NodeSelector/affinity incompatível:** Pod requer um label que nenhum nó possui
- **Taints sem tolerations:** Nós estão com taint, pod não tolera
- **PVC não vinculado:** Pod solicita um volume que não existe ou não pode ser provisionado
- **Restrições de topologia de pod:** Não consegue satisfazer requisitos de distribuição

**Diagnóstico:**
```bash
kubectl describe pod <pending-pod> | grep -A 20 "Events"
# Procure por: "FailedScheduling" com o motivo
```

### ImagePullBackOff — Não Consegue Obter a Imagem

O container runtime não consegue baixar a imagem especificada. Causas comuns:

- **Erro de digitação no nome da imagem:** `ngnix` em vez de `nginx`
- **Tag não existe:** `nginx:1.99` (versão inexistente)
- **Registry privado:** `imagePullSecrets` ausente no pod
- **Rate limiting:** Limites de taxa do Docker Hub excedidos
- **Registry indisponível:** O servidor do registry está inacessível

**Diagnóstico:**
```bash
kubectl describe pod <pod> | grep -A 5 "Events"
# Procure por: "Failed to pull image" com o erro específico
```

### CrashLoopBackOff — Crasha Repetidamente

O container inicia, crasha, e o K8s reinicia com backoff exponencial (10s, 20s, 40s, ... até 5 minutos).

- **Bug na aplicação:** Código sai com código diferente de zero imediatamente
- **Dependência ausente:** Variável de ambiente, arquivo de configuração ou conexão de serviço necessária
- **Entrypoint incorreto:** Comando errado no Dockerfile ou spec do pod
- **Liveness probe falhando:** Probe mata o container antes de estar pronto

**Diagnóstico:**
```bash
# Verificar os logs do container ANTERIOR que crashou
kubectl logs <pod> -p

# Verificar o código de saída
kubectl describe pod <pod> | grep -A 5 "Last State"
```

Códigos de saída comuns:
- **Exit 1** — erro geral da aplicação
- **Exit 137** — morto por SIGKILL (OOMKilled ou liveness probe)
- **Exit 139** — segfault (SIGSEGV)
- **Exit 143** — SIGTERM (desligamento gracioso)

### OOMKilled — Sem Memória

O container excedeu seu limite de memória e foi morto pelo OOM killer do Linux (via cgroups).

**Diagnóstico:**
```bash
kubectl describe pod <pod> | grep -i "OOMKilled"
kubectl describe pod <pod> | grep -A 3 "Last State"
```

**Correção:** Aumente o limite de memória, ou corrija o vazamento de memória na aplicação.

### CreateContainerConfigError — Referência de Config Inválida

O spec do pod referencia um ConfigMap ou Secret que não existe.

**Diagnóstico:**
```bash
kubectl describe pod <pod> | grep -A 5 "Events"
# "Error: configmap \"my-config\" not found"
```

### Evicted — Pressão no Nó

O kubelet remove pods quando o nó fica com pouco disco, memória ou PIDs.

**Diagnóstico:**
```bash
kubectl get pods | grep Evicted
kubectl describe pod <evicted-pod> | grep -i "evict"
kubectl describe node <node> | grep -A 10 "Conditions"
```

---

## Comparação de Debugging Linux ↔ Kubernetes

| Debugging Linux | Equivalente K8s | O Que Mostra |
|----------------|---------------|---------------|
| `systemctl status svc` | `kubectl get pods` | Estado do processo/pod |
| `journalctl -u svc` | `kubectl logs pod` | Saída da aplicação |
| `journalctl -xe` | `kubectl get events` | Linha do tempo de eventos do sistema |
| `ps aux`, `top` | `kubectl top pods` | Uso de recursos |
| `strace -p PID` | `kubectl debug` (efêmero) | Inspeção profunda do container |
| `ss -tlnp` | `kubectl get svc,endpoints` | Listeners de rede e backends |
| `ping`, `curl`, `traceroute` | `kubectl exec -- wget/curl` | Teste de conectividade de rede |
| `dmesg`, `/var/log/syslog` | `kubectl describe node` | Problemas no nível do nó (OOM, disco) |
| `df -h` | `kubectl describe node` Conditions | Detecção de pressão de disco |

---

> ### ⚠️ Onde a Analogia com Linux Quebra
>
> **Debugging remoto vs. acesso local:** No Linux, você debuga na própria máquina — faz SSH, executa comandos, lê arquivos. No Kubernetes, a "máquina" é abstraída. Você debuga através do API server usando `kubectl`. Se o API server estiver fora, você está completamente bloqueado (diferente do Linux onde ainda pode usar o console local ou IPMI). Sua cadeia de ferramentas de debugging depende do cluster estar pelo menos parcialmente funcional.
>
> **Restarts automáticos mascaram falhas:** Processos Linux crasham e ficam mortos até você reiniciá-los. Pods Kubernetes crasham e são automaticamente reiniciados — isso é CrashLoopBackOff. Isso pode mascarar problemas: o status do pod mostra "Running" mas a coluna RESTARTS mostra 47. Sempre verifique RESTARTS. Um pod que está rodando há 2 minutos com 50 restarts não está "funcionando."
>
> **Logs efêmeros:** Logs no Linux persistem em `/var/log` — você sempre pode voltar e lê-los. Logs de pods desaparecem quando o pod é deletado ou substituído. Se você precisar investigar um crash de ontem, esses logs se foram a menos que você os tenha enviado externamente (Fluent Bit, Loki, etc.) ANTES do pod morrer. Em produção, logging centralizado não é opcional — é um pré-requisito para debugging.

---

## Laboratório Diagnóstico: 5 Cenários Reais de Solução de Problemas

### Pré-requisitos

```bash
# Criar um cluster Kind para prática de troubleshooting
kind create cluster --name troubleshooting-lab --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

# Verificar que o cluster está pronto
kubectl get nodes
kubectl cluster-info
```

---

### Cenário 1: Pod Preso em Pending

**Implantar o estado quebrado:**

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

Este pod solicita 100 CPUs e 256 GiB de memória — muito mais do que qualquer nó em um cluster Kind pode fornecer.

**Diagnosticar passo a passo:**

```bash
# Passo 1: Verificar status do pod
kubectl get pods
# STATUS: Pending

# Passo 2: Descrever o pod — leia a seção Events
kubectl describe pod -l app=greedy-app
# Events mostrarão:
#   Warning  FailedScheduling  ... 0/3 nodes are available:
#   3 Insufficient cpu, 3 Insufficient memory.

# Passo 3: Verificar recursos disponíveis nos nós
kubectl describe nodes | grep -A 5 "Allocatable"
```

**Aplicar a correção:**

```bash
# Correção: definir requests de recursos razoáveis
kubectl patch deployment greedy-app --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/resources/requests/cpu", "value": "100m"},
  {"op": "replace", "path": "/spec/template/spec/containers/0/resources/requests/memory", "value": "128Mi"}
]'
```

**Verificar resolução:**

```bash
kubectl get pods -l app=greedy-app -w
# Aguarde até STATUS mostrar Running
```

**Limpeza:**

```bash
kubectl delete deployment greedy-app
```

---

### Cenário 2: CrashLoopBackOff

**Implantar o estado quebrado:**

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

Este container inicia, imprime um erro sobre uma URL de banco de dados ausente e sai com código 1.

**Diagnosticar passo a passo:**

```bash
# Passo 1: Verificar status do pod (aguarde ~30 segundos para o backoff aparecer)
kubectl get pods -l app=crash-app
# STATUS: CrashLoopBackOff, RESTARTS: 2+

# Passo 2: Verificar os logs do container ANTERIOR que crashou
kubectl logs -l app=crash-app -p
# Saída: "Starting app..."
#         "FATAL: Missing DATABASE_URL"

# Passo 3: Verificar logs atuais (pode estar vazio se o container acabou de reiniciar)
kubectl logs -l app=crash-app

# Passo 4: Descrever para ver código de saída e estado
kubectl describe pod -l app=crash-app | grep -A 10 "Last State"
# Last State: Terminated
#   Reason: Error
#   Exit Code: 1
```

**Aplicar a correção:**

```bash
# Correção: fornecer a variável de ambiente ausente
kubectl set env deployment/crash-app DATABASE_URL="postgres://db:5432/myapp"
```

Espere — este pod ainda vai crashar porque o comando do busybox está hardcoded. Vamos corrigir o comando também:

```bash
kubectl patch deployment crash-app --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/command", "value": ["/bin/sh", "-c", "echo Starting app... && echo Connected to $DATABASE_URL && sleep 3600"]}
]'
```

**Verificar resolução:**

```bash
kubectl get pods -l app=crash-app -w
# Aguarde STATUS: Running, RESTARTS: 0

kubectl logs -l app=crash-app
# "Starting app..."
# "Connected to postgres://db:5432/myapp"
```

**Limpeza:**

```bash
kubectl delete deployment crash-app
```

---

### Cenário 3: Service Inacessível

**Implantar o estado quebrado:**

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

O selector do Service diz `app: web-app-typo` mas os pods têm `app: web-app`.

**Diagnosticar passo a passo:**

```bash
# Passo 1: Verificar se os pods estão rodando
kubectl get pods -l app=web-app
# STATUS: Running (pods estão bem!)

# Passo 2: Verificar o service
kubectl get svc web-service
# ClusterIP atribuído — parece normal

# Passo 3: Verificar endpoints (O PASSO CHAVE)
kubectl get endpoints web-service
# ENDPOINTS: <none>  ← Este é o problema!

# Passo 4: Comparar labels
kubectl get pods --show-labels | grep web-app
# Labels: app=web-app,version=v2

kubectl describe svc web-service | grep Selector
# Selector: app=web-app-typo  ← Não corresponde!

# Passo 5: Tentar alcançar o service (vai falhar)
kubectl run debug-curl --image=busybox:1.36 --rm -it --restart=Never -- wget -qO- --timeout=3 http://web-service 2>&1 || true
# wget: download timed out
```

**Aplicar a correção:**

```bash
# Correção: corrigir o selector
kubectl patch svc web-service --type='json' -p='[
  {"op": "replace", "path": "/spec/selector", "value": {"app": "web-app"}}
]'
```

**Verificar resolução:**

```bash
# Endpoints agora devem mostrar IPs dos pods
kubectl get endpoints web-service
# ENDPOINTS: 10.244.x.x:80,10.244.y.y:80

# Testar conectividade
kubectl run debug-curl --image=busybox:1.36 --rm -it --restart=Never -- wget -qO- --timeout=3 http://web-service
# Retorna HTML da página de boas-vindas do nginx
```

**Limpeza:**

```bash
kubectl delete deployment web-app
kubectl delete svc web-service
```

---

### Cenário 4: ImagePullBackOff

**Implantar o estado quebrado:**

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

A tag de imagem `9.99.99-nonexistent` não existe no Docker Hub.

**Diagnosticar passo a passo:**

```bash
# Passo 1: Verificar status do pod
kubectl get pods -l app=bad-image-app
# STATUS: ErrImagePull ou ImagePullBackOff

# Passo 2: Descrever o pod
kubectl describe pod -l app=bad-image-app | grep -A 10 "Events"
# Events:
#   Warning  Failed   ... Failed to pull image "nginx:9.99.99-nonexistent":
#     rpc error: ... manifest unknown

# Passo 3: Verificar se a imagem existe (fora do cluster)
# Você verificaria o Docker Hub ou seu registry para confirmar tags válidas
```

**Aplicar a correção:**

```bash
# Correção: usar uma tag de imagem válida
kubectl set image deployment/bad-image-app app=nginx:1.27
```

**Verificar resolução:**

```bash
kubectl get pods -l app=bad-image-app -w
# Aguarde STATUS: Running

kubectl describe pod -l app=bad-image-app | grep "Image:"
# Image: nginx:1.27
```

**Limpeza:**

```bash
kubectl delete deployment bad-image-app
```

---

### Cenário 5: Node NotReady

Em um cluster Kind, podemos simular um problema de nó parando o container que roda o worker node.

**Implantar workloads primeiro:**

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

# Aguardar todos os pods estarem rodando
kubectl wait --for=condition=Ready pod -l app=resilient-app --timeout=60s
kubectl get pods -l app=resilient-app -o wide
```

**Simular a falha:**

```bash
# Obter os nomes dos worker nodes
kubectl get nodes

# Pausar um worker node (nós Kind são containers Docker)
docker pause troubleshooting-lab-worker

# Aguardar 40-60 segundos para o nó ser marcado como NotReady
sleep 60
```

**Diagnosticar passo a passo:**

```bash
# Passo 1: Verificar status do nó
kubectl get nodes
# Um nó mostra NotReady

# Passo 2: Descrever o nó NotReady
kubectl describe node troubleshooting-lab-worker | grep -A 10 "Conditions"
# Ready: False — Kubelet parou de reportar status

# Passo 3: Verificar distribuição dos pods
kubectl get pods -l app=resilient-app -o wide
# Pods no nó NotReady eventualmente serão reagendados (após o
# pod-eviction-timeout, tipicamente 5 minutos)
```

**Aplicar a correção:**

```bash
# Correção: "reparar" o nó despausando-o
docker unpause troubleshooting-lab-worker

# Aguardar o nó ficar Ready
kubectl wait --for=condition=Ready node/troubleshooting-lab-worker --timeout=120s
kubectl get nodes
```

**Verificar resolução:**

```bash
kubectl get nodes
# Todos os nós mostram Ready

kubectl get pods -l app=resilient-app -o wide
# Todos os pods rodando e distribuídos entre os nós
```

**Limpeza:**

```bash
kubectl delete deployment resilient-app
```

---

### O Kit de Ferramentas de Debugging — Resumo

Quando você estiver travado, estas são suas ferramentas essenciais:

```bash
# Executar um pod temporário de debug com ferramentas comuns de rede
kubectl run debug --image=busybox:1.36 -it --rm --restart=Never -- /bin/sh

# Anexar um container efêmero a um pod em execução
kubectl debug <pod-name> -it --image=busybox:1.36 --target=<container-name>

# Debugar diretamente em um nó
kubectl debug node/<node-name> -it --image=busybox:1.36

# Obter o YAML completo de um pod (ver campos computados, valores padrão)
kubectl get pod <pod-name> -o yaml

# Verificar uso de recursos (requer metrics-server)
kubectl top pods
kubectl top nodes
```

### Limpeza do Laboratório

```bash
# Deletar o cluster Kind
kind delete cluster --name troubleshooting-lab
```

---

## Principais Conclusões

1. **Sempre comece com `kubectl describe`.** A seção Events no final conta a história cronológica do que aconteceu com o pod. É o comando de debugging mais informativo.

2. **Use `kubectl logs -p` para containers que crasharam.** O container atual pode não ter logs ainda (acabou de reiniciar). O container anterior tem a saída do crash que você precisa.

3. **Endpoints vazios = incompatibilidade de selector.** Quando um Service não consegue alcançar pods, verifique `kubectl get endpoints <svc>`. Se estiver vazio, o selector do Service não corresponde a nenhum label de pod.

4. **O status do pod indica onde investigar.** Pending = problema de agendamento. ImagePullBackOff = problema de imagem/registry. CrashLoopBackOff = problema da aplicação. OOMKilled = problema de memória. Cada status aponta para um caminho de investigação diferente.

5. **Verifique RESTARTS, não apenas STATUS.** Um pod mostrando "Running" com 100+ restarts não está saudável — é um CrashLoopBackOff que por acaso está na fase "running" do seu ciclo de crash quando você olhou.

6. **Events têm timestamp e são ordenáveis.** Use `kubectl get events --sort-by=.metadata.creationTimestamp` para uma visão de linha do tempo. Isso revela falhas em cascata — um evento desencadeando outro.

7. **Na dúvida, execute um pod de debug.** `kubectl run debug --image=busybox:1.36 -it --rm` dá a você um shell dentro da rede do cluster. De lá você pode usar `wget`, `nslookup`, e testar conectividade que é invisível de fora.

---

## Leitura Adicional

- [Troubleshooting Applications](https://kubernetes.io/docs/tasks/debug/debug-application/)
- [Debug Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
- [Debug Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- [kubectl debug](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_debug/)
- [Pod Lifecycle — Pod Phase](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase)
- [Resource Management for Pods](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

---

**Anterior:** [Capítulo 12 — Scaling e Observabilidade](12-scaling-and-observability.md)
**Próximo:** [Capítulo 14 — Operações em Produção](14-production-operations.md)
