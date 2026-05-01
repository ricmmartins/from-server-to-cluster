# Capítulo 3: Arquitetura do Kubernetes

*"No Linux, você gerencia um servidor. No Kubernetes, você gerencia a intenção — e controllers gerenciam os servidores para você."*

---

## O Cluster É um Sistema Operacional Distribuído

No Capítulo 2, você aprendeu que containers são apenas processos Linux com namespaces e cgroups. Agora imagine que você precisa executar centenas desses containers em dezenas de máquinas, mantê-los saudáveis, rotear tráfego entre eles e lidar com falhas automaticamente.

Esse é o problema que o Kubernetes resolve — e ele resolve tomando emprestado padrões arquiteturais diretamente do Linux.

Pense em como um único servidor Linux funciona:

- **systemd** (PID 1) garante que seus serviços estejam rodando e os reinicia se caírem
- **/etc/** armazena arquivos de configuração que definem como os serviços se comportam
- **iptables/nftables** gerencia regras de roteamento de rede
- O **escalonador de CPU** decide qual processo recebe tempo de CPU e em qual core
- **D-Bus / systemctl** é a interface que você usa para se comunicar com o sistema

O Kubernetes tem um equivalente para cada um desses — mas em vez de gerenciar processos em uma máquina, ele gerencia containers em uma frota de máquinas. A arquitetura se divide em duas partes: o **control plane** (o cérebro) e **worker nodes** (o músculo).

## Componentes do Control Plane

O control plane executa os componentes de tomada de decisão e gerenciamento de estado. Em um cluster de produção, ele tipicamente roda em máquinas dedicadas (ou múltiplas máquinas para alta disponibilidade). No Kind, tudo roda em um único container Docker.

### kube-apiserver — A Interface Central de Comando

**Analogia Linux: systemctl / D-Bus**

Toda interação com um cluster Kubernetes passa pelo API server. Quando você executa `kubectl get pods`, está fazendo uma chamada HTTPS REST ao API server. Quando o scheduler precisa atribuir um Pod a um node, ele fala com o API server. Quando o kubelet reporta o status de um node, ele fala com o API server.

O API server é:
- **O único ponto de entrada** — nenhum componente fala diretamente com outro; tudo passa pela API
- **Um gateway de autenticação e autorização** — cada requisição é autenticada (certificados, tokens) e autorizada (RBAC)
- **Um admission controller** — requisições podem ser validadas, mutadas ou rejeitadas por admission webhooks antes de serem persistidas
- **Uma REST API** — cada recurso Kubernetes (Pods, Services, Deployments) é um endpoint REST que você pode consultar com `curl`

Em um servidor Linux, `systemctl` é sua interface para gerenciar serviços, e o D-Bus é o backbone de comunicação entre componentes do sistema. O API server serve ambos os papéis para o cluster inteiro.

### etcd — O Armazenamento de Configuração Distribuído

**Analogia Linux: diretório /etc/ (mas distribuído e versionado)**

etcd é um armazenamento distribuído de chave-valor que mantém todo o estado do cluster: cada Pod, cada Service, cada ConfigMap, cada Secret. Quando o API server recebe uma requisição para criar um Pod, a definição do Pod é persistida no etcd. Quando um controller precisa saber quais Deployments existem, ele lê do etcd (via API server — nada fala com o etcd diretamente exceto o API server).

Características principais:
- **Consenso distribuído** usando o protocolo Raft — dados são replicados entre múltiplos nodes etcd (em produção, tipicamente 3 ou 5)
- **Fortemente consistente** — leituras sempre retornam o último valor commitado
- **Versionado** — cada chave tem um número de revisão; o Kubernetes usa isso para controle de concorrência otimista (o campo `resourceVersion` em cada objeto)
- **Suporte a watch** — componentes podem se inscrever para mudanças em chaves específicas, habilitando o padrão de controller reativo que faz o Kubernetes funcionar

Se `/etc/` em um servidor Linux é a fonte da verdade para a configuração daquela máquina, etcd é a fonte da verdade para todo o estado do cluster.

### kube-scheduler — O Posicionador de Workloads

**Analogia Linux: escalonador de CPU do Linux**

Quando você cria um Pod, ele inicialmente não tem node atribuído. O scheduler observa Pods não agendados, avalia quais nodes podem executá-los (filtragem), pontua os candidatos restantes (ranking) e atribui o Pod ao melhor node.

O scheduler considera:
- **Disponibilidade de recursos** — o node tem CPU e memória suficientes?
- **Regras de afinidade e anti-afinidade** — o Pod precisa estar perto (ou longe) de outros Pods?
- **Taints e tolerations** — o node está marcado como inadequado para workloads gerais?
- **Restrições de topologia** — os Pods devem ser distribuídos entre zonas de disponibilidade?

Em um servidor Linux, o escalonador de CPU decide qual processo roda em qual core baseado em prioridade e compartilhamento justo. O scheduler do Kubernetes faz a mesma coisa, mas no nível de máquina — decidindo qual node roda qual Pod.

### kube-controller-manager — O Motor de Reconciliação

**Analogia Linux: gerenciadores de serviço do systemd (combinados)**

O controller manager executa uma coleção de **controllers** — cada um é um loop de controle que observa o estado do cluster (via API server) e trabalha para fazer a realidade corresponder ao estado desejado.

Controllers principais incluem:
- **Node controller** — monitora a saúde dos nodes, marca nodes como NotReady se param de reportar
- **Replication controller** — garante que o número correto de réplicas de Pod esteja rodando
- **Job controller** — gerencia workloads em batch (executar até completar)
- **ServiceAccount controller** — cria ServiceAccounts padrão para novos namespaces
- **Endpoint controller** — popula o objeto Endpoints (quais Pods respondem por um Service)

Pense no systemd no Linux: ele lê unit files (estado desejado) e garante que os serviços correspondentes estejam rodando (estado real). Se um serviço cai, o systemd o reinicia. Controllers do Kubernetes fazem o mesmo, mas em escala — se um Pod morre, o ReplicaSet controller cria um substituto. Se um node cai, o node controller o marca como não saudável e controllers reagendam os Pods afetados em outro lugar.

### cloud-controller-manager — A Camada de Integração com a Nuvem

**Analogia Linux: drivers de kernel específicos de hardware**

Este componente opcional integra o Kubernetes com APIs de provedores de nuvem. Ele cuida de:
- **Gerenciamento de nodes** — detectando quando uma VM de nuvem é deletada e removendo o objeto Node correspondente
- **Load balancers** — provisionando load balancers de nuvem quando você cria um Service do tipo `LoadBalancer`
- **Armazenamento** — interagindo com APIs de armazenamento em nuvem para provisionamento de volumes

Assim como o Linux tem drivers de kernel que abstraem diferenças de hardware (você usa o mesmo comando `mount` independente do tipo de disco), o cloud controller manager abstrai diferenças de nuvem para que o Kubernetes funcione da mesma forma em todo lugar.

No Kind (e em clusters bare-metal), este componente não existe — não há nuvem para integrar.

## Componentes do Worker Node

Worker nodes são onde suas workloads reais rodam. Cada node executa um pequeno conjunto de componentes que recebem instruções do control plane e garantem que containers estejam rodando corretamente.

### kubelet — O Agente do Node

**Analogia Linux: systemd (PID 1) em cada máquina**

O kubelet é um agente rodando em cada node. Seu trabalho é simples e crítico: **garantir que os containers descritos em PodSpecs estejam rodando e saudáveis.**

O kubelet:
- **Recebe atribuições de Pod** do API server (via watch)
- **Diz ao container runtime** (containerd) para puxar imagens e iniciar containers
- **Executa health checks** (liveness probes, readiness probes) e reporta status de volta ao API server
- **Gerencia o ciclo de vida de Pods** — inicia, para e reinicia containers conforme necessário
- **Reporta status do node** — CPU, memória, disco e informações de Pods em execução

Em um servidor Linux, o systemd lê unit files e garante que os processos certos estejam rodando. O kubelet lê PodSpecs e garante que os containers certos estejam rodando. O paralelo é direto — mas o kubelet recebe suas instruções do API server, não de arquivos locais (na maior parte — static Pods são a exceção).

### kube-proxy — O Gerenciador de Regras de Rede

**Analogia Linux: gerenciador de regras iptables / nftables**

O kube-proxy roda em cada node e mantém regras de rede que habilitam a abstração de Service do Kubernetes. Quando você cria um Service, o kube-proxy programa regras para que tráfego para o ClusterIP do Service seja encaminhado para um dos Pods que o respaldam.

O kube-proxy pode operar em diferentes modos:
- **Modo iptables** (comum) — programa regras iptables para cada Service; encaminhamento de pacotes em kernel-space
- **Modo IPVS** — usa o IP Virtual Server do kernel para melhor performance em escala
- **Modo nftables** (mais novo, alpha desde Kubernetes v1.29) — usa nftables como backend, substituindo regras iptables

Em um servidor Linux, você escreve regras iptables para rotear tráfego. No Kubernetes, o kube-proxy escreve essas regras para você baseado nos Services que você define. O mecanismo é idêntico — a camada de automação é o que há de novo.

### Container Runtime — O Motor de Containers

**Analogia Linux: gerenciador de pacotes + lançador de processos**

O container runtime puxa imagens e inicia containers. No Kubernetes moderno (v1.24+), o runtime padrão é **containerd**, que se comunica com o kubelet através da Container Runtime Interface (CRI).

A pilha em cada node é:

```
kubelet → (CRI) → containerd → runc → kernel Linux
```

Esta é a mesma pilha do Capítulo 2, menos o Docker. O Kubernetes não precisa da CLI ou daemon do Docker — ele fala diretamente com o containerd.

## Como Tudo Se Encaixa

Aqui está o fluxo quando você executa `kubectl run nginx --image=nginx`:

1. **kubectl** envia um HTTPS POST ao **API server** com a especificação do Pod
2. **API server** te autentica, verifica autorização RBAC, executa admission controllers e persiste o Pod no **etcd** (com status "Pending")
3. **Scheduler** percebe o novo Pod não atribuído (via watch no API server), avalia nodes e atualiza o Pod com o nome do node selecionado
4. **kubelet** no node escolhido percebe que tem um novo Pod atribuído (via seu watch), diz ao **containerd** para puxar a imagem nginx e iniciar o container
5. **kubelet** reporta o status do Pod como "Running" ao API server, que o persiste no etcd
6. **kube-proxy** em cada node atualiza regras de rede para que o Pod seja alcançável via Services

Cada passo passa pelo API server. O API server é o hub; todo o resto é um spoke. Isso é por design — significa que há um único ponto de estrangulamento auditável e autenticado para todas as operações do cluster.

## Tabela de Comparação de Arquitetura Linux ↔ Kubernetes

| Componente Linux | Equivalente K8s | Papel |
|----------------|----------------|------|
| Diretório `/etc/` | etcd | Armazenamento de configuração e estado (mas distribuído, versionado e com consenso) |
| systemd (PID 1) | kubelet | Garante que processos/containers estejam rodando conforme declarado |
| Sistema init (processo de boot) | kube-scheduler | Decide o que roda onde baseado em recursos disponíveis |
| D-Bus / `systemctl` | kube-apiserver | Interface central de comando e comunicação |
| iptables / nftables | kube-proxy | Gerenciamento de regras de roteamento de rede |
| Gerenciador de pacotes (apt, yum) | Container runtime (containerd) | Puxa e executa pacotes de software/imagens |
| Drivers de hardware | cloud-controller-manager | Abstrai diferenças de infraestrutura subjacente |
| Unit files do systemd | PodSpecs (manifestos YAML) | Descrições declarativas do que deve rodar |
| `/var/log/` + journald | Logs de auditoria do API server + logs de Pod | Registro centralizado de atividade e eventos |

> **Onde a Analogia com Linux Quebra**
>
> - **etcd não é apenas `/etc/`.** É um sistema de consenso distribuído usando o protocolo Raft. Perder a maioria dos nodes etcd significa perder a capacidade de escrever estado no cluster. Perder todos os dados do etcd sem backup significa perder o cluster inteiramente. Em um servidor Linux, corromper `/etc/` é ruim mas recuperável — você pode dar boot por um disco de resgate. Falha do etcd em escala é um desastre muito maior porque afeta cada node e workload no cluster.
>
> - **O API server não é apenas `systemctl`.** Ele é simultaneamente um gateway de autenticação, um motor de autorização (RBAC), um pipeline de admission controller, uma REST API e o único componente que fala com o etcd. Nada acontece no cluster sem passar por ele. No Linux, você pode contornar o systemd e rodar processos manualmente. No Kubernetes, contornar o API server significa que o cluster não sabe que a workload existe.
>
> - **O fluxo de controle é invertido.** Em um servidor Linux, você faz SSH e executa comandos — você empurra instruções para a máquina. No Kubernetes, o kubelet **puxa** instruções do API server. Você nunca faz SSH em um node para iniciar um container. Você submete um manifesto ao API server, e o kubelet no node apropriado o pega. Esse modelo baseado em pull é o que permite ao Kubernetes gerenciar milhares de nodes sem precisar de acesso SSH a nenhum deles.

## Lab de Diagnóstico: Explorando a Arquitetura do Kubernetes

Este lab usa um cluster Kind para explorar os componentes reais que formam um cluster Kubernetes. Você verá que cada componente do control plane é um processo Linux real rodando dentro do container Kind.

### Pré-requisitos

- Docker instalado e rodando
- Kind e kubectl instalados (veja o lab do Capítulo 1)

### Exercício 1: Criar um Cluster Kind

```bash
kind create cluster --name arch-lab
```

Aguarde o cluster ficar pronto:

```bash
kubectl cluster-info --context kind-arch-lab
```

> **Fonte:** [Kind — Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)

### Exercício 2: Explorar Componentes do Control Plane

No Kind, todos os componentes do control plane rodam como static Pods no namespace `kube-system`:

```bash
kubectl get pods -n kube-system
```

Você deve ver algo como:

```
NAME                                         READY   STATUS    RESTARTS   AGE
coredns-...                                  1/1     Running   0          2m
coredns-...                                  1/1     Running   0          2m
etcd-arch-lab-control-plane                  1/1     Running   0          2m
kindnet-...                                  1/1     Running   0          2m
kube-apiserver-arch-lab-control-plane        1/1     Running   0          2m
kube-controller-manager-arch-lab-control-plane  1/1  Running   0          2m
kube-proxy-...                               1/1     Running   0          2m
kube-scheduler-arch-lab-control-plane        1/1     Running   0          2m
```

Cada componente que discutimos está ali — rodando como container (que é apenas um processo Linux, como você aprendeu no Capítulo 2).

### Exercício 3: Verificar a Saúde do Cluster

A forma moderna de verificar a saúde do API server usa os endpoints `/readyz` e `/livez`:

```bash
# Verificação de readiness (o API server está pronto para servir tráfego?)
kubectl get --raw='/readyz?verbose'
```

Você verá uma lista detalhada de health checks, cada um reportando `ok` ou um motivo de falha. Isso diz quais subsistemas estão saudáveis.

```bash
# Verificação de liveness (o API server está vivo?)
kubectl get --raw='/livez?verbose'
```

> **Nota:** Você pode ver `kubectl get componentstatuses` referenciado em tutoriais mais antigos. Este comando e sua API subjacente foram depreciados desde o Kubernetes v1.19 e podem produzir resultados incompletos ou enganosos. Use os endpoints `/readyz` e `/livez` em vez disso.
>
> **Fonte:** [Kubernetes API Health Endpoints](https://kubernetes.io/docs/reference/using-api/health-checks/)

### Exercício 4: Inspecionar Logs do kubelet

O kubelet roda como um serviço systemd dentro do container Kind (que é ele próprio um container Docker). Vamos ver seus logs:

```bash
docker exec arch-lab-control-plane journalctl -u kubelet --no-pager | tail -20
```

Você verá o kubelet iniciando, registrando o node e sincronizando estados de Pod — exatamente como ler `journalctl -u nginx` em um servidor Linux normal.

Para uma visão mais ampla da atividade do kubelet:

```bash
docker exec arch-lab-control-plane journalctl -u kubelet --no-pager | grep -i "started" | tail -10
```

### Exercício 5: Espiar Dentro do etcd

Todo o estado do cluster vive no etcd. Vamos provar consultando o etcd diretamente de dentro do Pod etcd:

```bash
# Listar algumas chaves armazenadas no etcd — estas SÃO seus objetos do cluster
kubectl exec -n kube-system etcd-arch-lab-control-plane -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get / --prefix --keys-only --limit=20
```

Você verá chaves como:
```
/registry/configmaps/kube-system/coredns
/registry/namespaces/default
/registry/pods/kube-system/etcd-arch-lab-control-plane
/registry/services/specs/default/kubernetes
```

Cada objeto Kubernetes — Pods, Services, ConfigMaps, Secrets — é armazenado como uma chave no etcd sob o prefixo `/registry/`. Esta é a fonte da verdade do cluster.

> **Fonte:** [etcd documentation](https://etcd.io/docs/) | [Kubernetes — Operating etcd clusters](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)

### Exercício 6: Rastrear as Chamadas de API

Quando você executa um comando kubectl, ele está fazendo chamadas REST API ao API server. Vamos vê-las:

```bash
kubectl get pods -n kube-system -v=6
```

No nível de verbosidade 6, você verá saída como:

```
I0101 12:00:00.000000  loader.go:395] Config loaded from file: /home/user/.kube/config
I0101 12:00:00.100000  round_trippers.go:553] GET https://127.0.0.1:PORT/api/v1/namespaces/kube-system/pods?limit=500 200 OK in 15 milliseconds
```

Isso revela a requisição HTTP real: `GET /api/v1/namespaces/kube-system/pods`. Kubernetes é apenas uma REST API — `kubectl` é um cliente HTTP sofisticado. Você poderia fazer a mesma requisição com `curl`:

```bash
# Obter a URL do API server e usar as credenciais do kubectl para fazer uma requisição raw
kubectl get --raw /api/v1/namespaces/kube-system/pods | python3 -m json.tool | head -20
```

Entender que o Kubernetes é uma REST API torna o debugging muito mais intuitivo — você está apenas lidando com endpoints HTTP que retornam JSON.

> **Fonte:** [Kubernetes API Overview](https://kubernetes.io/docs/concepts/overview/kubernetes-api/) | [kubectl Verbosity and Debugging](https://kubernetes.io/docs/reference/kubectl/quick-reference/#kubectl-output-verbosity-and-debugging)

### Limpeza

```bash
kind delete cluster --name arch-lab
```

## Principais Conclusões

1. **A arquitetura do Kubernetes se divide em control plane (cérebro) e worker nodes (músculo).** O control plane toma decisões; worker nodes as executam.
2. **O API server é o hub de tudo.** Toda comunicação entre componentes passa por ele — não há conversa direta entre componentes. Isso torna o sistema auditável e seguro.
3. **etcd armazena todo o estado do cluster** como um armazenamento distribuído de chave-valor fortemente consistente. Perder dados do etcd significa perder o cluster. Faça backup.
4. **O scheduler atribui Pods a nodes** baseado em disponibilidade de recursos, restrições e políticas — assim como o escalonador de CPU do Linux atribui processos a cores.
5. **O kubelet é o systemd para containers.** Ele roda em cada node, recebe PodSpecs do API server e garante que os containers certos estejam rodando e saudáveis.
6. **kube-proxy gerencia regras de rede** (iptables, IPVS ou nftables) para implementar a abstração de Service — mesmas funcionalidades do kernel que você já conhece, automatizadas em escala de cluster.
7. **Tudo é uma chamada REST API.** Quando você executa `kubectl get pods`, está fazendo uma requisição HTTP GET. Entender isso torna o debugging transparente — adicione `-v=6` a qualquer comando kubectl para ver as chamadas de API reais.

## Leitura Adicional

- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)
- [API Health Endpoints](https://kubernetes.io/docs/reference/using-api/health-checks/)
- [Operating etcd Clusters](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
- [Kind — Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [Kubernetes API Concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/)

---

**Anterior:** [Capítulo 2 — Containers Desmistificados](02-containers-demystified.md)

**Próximo:** [Capítulo 4 — Como o Kubernetes Pensa](04-how-kubernetes-thinks.md) — Estado desejado, loops de reconciliação, controllers, labels, selectors — o modelo mental que faz tudo se encaixar.
