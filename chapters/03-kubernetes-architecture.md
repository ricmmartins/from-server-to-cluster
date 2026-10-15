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

O control plane executa os componentes de tomada de decisão e gerenciamento de estado. Em um cluster de produção, ele tipicamente roda em máquinas dedicadas (ou múltiplas máquinas para alta disponibilidade). Em um cluster Kind de um único node, o control plane e as workloads compartilham um container Docker; clusters Kind com múltiplos nodes usam um container por node.

### kube-apiserver — A Interface Central de Comando

**Analogia Linux: systemctl / D-Bus**

O API server é o ponto central para os objetos da API Kubernetes e o estado desejado. Quando você executa `kubectl get pods`, está fazendo uma chamada HTTPS REST ao API server. Quando o scheduler precisa atribuir um Pod a um node, ele fala com o API server. Quando o kubelet reporta o status de um node, ele fala com o API server.

O API server é:
- **O ponto de entrada para operações da API Kubernetes** — clientes e controllers leem e atualizam os objetos do cluster aqui, em vez de editar o etcd diretamente
- **Um gateway de autenticação e autorização** — cada requisição é autenticada (certificados, tokens) e autorizada (RBAC)
- **Um admission controller** — requisições podem ser validadas, mutadas ou rejeitadas por admission webhooks antes de serem persistidas
- **Uma REST API** — cada recurso Kubernetes (Pods, Services, Deployments) é um endpoint REST que você pode consultar com `curl`

Em um servidor Linux, `systemctl` é sua interface para gerenciar serviços, e o D-Bus é o backbone de comunicação entre componentes do sistema. O API server serve ambos os papéis para o cluster inteiro.

Isso não o torna um canal universal de comunicação. O kubelet fala **diretamente com seu runtime local via CRI**, e o tráfego da aplicação flui entre Pods e Services sem passar pelo API server. No sentido inverso, o API server se conecta ao **endpoint HTTPS do kubelet** para operações como logs, exec/attach e port-forward.

> **Fonte:** [Comunicação entre Nodes e o Control Plane](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/)

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
- **Disponibilidade de recursos** — os requests do Pod cabem na CPU e memória alocáveis do node depois de contabilizar os requests existentes?
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
- **EndpointSlice controller** — mantém EndpointSlices que descrevem quais Pods respondem por um Service (a API Endpoints antiga está descontinuada no Kubernetes v1.33 e posteriores)

Pense no systemd no Linux: ele lê unit files (estado desejado) e garante que os serviços correspondentes estejam rodando (estado real). Se um serviço cai, o systemd o reinicia. Controllers do Kubernetes fazem o mesmo, mas em escala — se um Pod gerenciado é perdido, o ReplicaSet controller cria um substituto. Se um node cai, o node controller o marca como não saudável; após o tratamento de evictions e tolerations, os controllers das workloads criam Pods substitutos e o scheduler tenta posicioná-los em outro lugar. Pods existentes não mudam de node. A recuperação leva tempo e exige capacidade adequada (veja o Capítulo 4).

### cloud-controller-manager — A Camada de Integração com a Nuvem

**Analogia Linux: drivers de kernel específicos de hardware**

Este componente opcional integra o Kubernetes com APIs de provedores de nuvem. Ele cuida de:
- **Gerenciamento de nodes** — detectando quando uma VM de nuvem é deletada e removendo o objeto Node correspondente
- **Load balancers** — provisionando load balancers de nuvem quando você cria um Service do tipo `LoadBalancer`
- **Rotas** — configurando rotas de rede em nuvem entre nodes, quando a rede do provedor exige isso

O provisionamento moderno de volumes em nuvem é feito por **drivers CSI (Container Storage Interface) e seus sidecars de controller**, incluindo o provisionador externo — não pelo cloud-controller-manager.

Assim como o Linux tem drivers de kernel que abstraem diferenças de hardware (você usa o mesmo comando `mount` independente do tipo de disco), o cloud controller manager abstrai diferenças de nuvem para que o Kubernetes funcione da mesma forma em todo lugar.

O cluster Kind padrão destes labs não tem cloud-controller-manager. Clusters bare-metal também costumam omiti-lo, a menos que uma integração de infraestrutura forneça um.

> **Fontes:** [Cloud Controller Manager](https://kubernetes.io/docs/concepts/architecture/cloud-controller/) | [Volumes CSI](https://kubernetes.io/docs/concepts/storage/volumes/#csi)

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
- **Modo IPVS** — usa o IP Virtual Server do kernel; está descontinuado na versão de referência dos labs, Kubernetes v1.35
- **Modo nftables** — usa nftables como backend; estável desde o Kubernetes v1.33 (inicialmente alpha no v1.29)

Em um servidor Linux, você escreve regras iptables para rotear tráfego. No Kubernetes, o kube-proxy escreve essas regras para você baseado nos Services que você define. O mecanismo é idêntico — a camada de automação é o que há de novo.

> **Fonte:** [Kubernetes v1.35 — IPs Virtuais e Proxies de Service](https://v1-35.docs.kubernetes.io/docs/reference/networking/virtual-ips/)

### Container Runtime — O Motor de Containers

**Analogia Linux: gerenciador de pacotes + lançador de processos**

O container runtime puxa imagens e inicia containers. O Kubernetes exige um runtime compatível com CRI, como **containerd** (usado nestes labs) ou CRI-O. O kubelet se comunica com ele através da Container Runtime Interface (CRI).

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
6. **Se um Service seleciona esse Pod**, o **EndpointSlice controller** atualiza seus endpoints conforme a prontidão muda; o **kube-proxy** observa Services e EndpointSlices e atualiza as regras de encaminhamento. `kubectl run` sozinho não cria um Service.

Leituras e escritas de objetos passam pelo API server, tornando-o o ponto central do estado desejado. O download de imagens e as chamadas de ciclo de vida dos containers acontecem entre o kubelet e o runtime via CRI, enquanto o tráfego de Service usa o plano de dados do node. Mantenha esses caminhos separados ao investigar problemas.

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
> - **O API server não é apenas `systemctl`.** Ele é simultaneamente um gateway de autenticação, um motor de autorização (RBAC), um pipeline de admission controller e uma REST API com dados persistidos no etcd. O gerenciamento normal de objetos Kubernetes passa por ele, mas reinícios locais de containers e tráfego do plano de dados não. Static Pods são outra exceção: o kubelet lê seus manifestos locais e reporta mirror Pods pela API quando ela está disponível.
>
> - **O fluxo de controle é invertido.** Em um servidor Linux, você faz SSH e executa comandos — você empurra instruções para a máquina. No Kubernetes, o kubelet **puxa** instruções do API server. Você nunca faz SSH em um node para iniciar um container. Você submete um manifesto ao API server, e o kubelet no node apropriado o pega. Esse modelo baseado em pull é o que permite ao Kubernetes gerenciar milhares de nodes sem precisar de acesso SSH a nenhum deles.

## Lab de Diagnóstico: Explorando a Arquitetura do Kubernetes

Este lab usa um cluster Kind para explorar os componentes reais que formam um cluster Kubernetes. Você verá que cada componente do control plane é um processo Linux real rodando dentro do container Kind.

### Pré-requisitos

- Docker instalado e rodando
- Kind e kubectl instalados (veja o lab do Capítulo 1)

### Exercício 1: Criar um Cluster Kind

```bash
kind create cluster --name arch-lab \
  --image kindest/node:v1.35.0@sha256:452d707d4862f52530247495d180205e029056831160e22870e37e3f6c1ac31f
```

Aguarde o cluster ficar pronto:

```bash
kubectl cluster-info --context kind-arch-lab
```

> **Fonte:** [Kind — Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)

### Exercício 2: Explorar Componentes do Control Plane

No Kind, o API server, o etcd, o scheduler e o controller manager rodam como static Pods em `kube-system`. O mesmo namespace também contém Pods de complementos gerenciados por Deployments ou DaemonSets:

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

Os componentes do control plane e de rede estão ali como containers. O kubelet roda separadamente como um serviço systemd, e o cluster Kind padrão não tem cloud-controller-manager.

### Exercício 3: Verificar a Saúde do Cluster

A forma moderna de verificar a saúde do API server usa os endpoints `/readyz` e `/livez`:

```bash
# Readiness check (is the API server ready to serve traffic?)
kubectl get --raw='/readyz?verbose'
```

Você verá uma lista detalhada de health checks, cada um reportando `ok` ou um motivo de falha. Isso diz quais subsistemas estão saudáveis.

```bash
# Liveness check (is the API server alive?)
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
# List some keys stored in etcd — these ARE your cluster objects
kubectl exec -n kube-system etcd-arch-lab-control-plane -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get / --prefix --keys-only --limit=20
```

As primeiras 20 chaves dependem dos objetos do cluster. No registro, as chaves usam caminhos como estes (não necessariamente todos neste resultado limitado):
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
# Use the current kubeconfig credentials to make a raw API request
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
2. **O API server é o ponto central para os objetos do cluster, não para todo o tráfego.** Kubelets chamam runtimes diretamente via CRI; o API server contata kubelets para logs, exec e port-forward; o tráfego da aplicação segue o plano de dados.
3. **etcd armazena todo o estado do cluster** como um armazenamento distribuído de chave-valor fortemente consistente. Perder dados do etcd significa perder o cluster. Faça backup.
4. **O scheduler atribui Pods a nodes** baseado em disponibilidade de recursos, restrições e políticas — assim como o escalonador de CPU do Linux atribui processos a cores.
5. **O kubelet é o systemd para containers.** Ele roda em cada node, recebe PodSpecs do API server e garante que os containers certos estejam rodando e saudáveis.
6. **kube-proxy gerencia regras de rede** (iptables, IPVS ou nftables) para implementar a abstração de Service — mesmas funcionalidades do kernel que você já conhece, automatizadas em escala de cluster.
7. **kubectl gerencia objetos pela API Kubernetes.** Quando você executa `kubectl get pods`, está fazendo uma requisição HTTP GET. Adicione `-v=6` para ver detalhes das requisições à API; isso não rastreia chamadas CRI nem pacotes da aplicação.

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
