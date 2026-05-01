# Capítulo 1: Do Servidor ao Cluster

*"Tudo o que você sabe sobre Linux ainda se aplica. Você só precisa ver de uma altitude maior."*

---

## Você Já Sabe Mais do Que Imagina

Se você passou anos gerenciando servidores Linux — configurando serviços com systemd, depurando rede com ss e tcpdump, gerenciando usuários e permissões, escrevendo shell scripts que mantêm a produção funcionando — talvez olhe para o Kubernetes e sinta que está começando do zero.

Não está.

O Kubernetes não surgiu do nada. Ele foi construído por pessoas que gerenciavam servidores Linux em escala massiva (o sistema Borg do Google rodava em Linux). Cada conceito central do Kubernetes tem um ancestral direto no Linux:

| O Que Você Já Sabe (Linux) | O Que Vai Aprender (Kubernetes) |
|------------------------|-------------------------------|
| Processos | Pods |
| systemd units | Deployments |
| iptables / nftables | Services e kube-proxy |
| /etc/fstab, mount | Volumes, PersistentVolumeClaims |
| /etc/ arquivos de configuração | ConfigMaps e Secrets |
| users, groups, chmod | RBAC, ServiceAccounts |
| cron | CronJobs |
| systemctl restart | Rolling updates |
| top, htop | kubectl top, Metrics Server |
| journalctl | kubectl logs |
| namespaces (unshare) | Kubernetes Namespaces |
| cgroups | Resource requests e limits |

Esta tabela não é um truque — é a tese deste livro inteiro. Você não está aprendendo algo alienígena. Está aprendendo como conceitos familiares escalam além de uma única máquina.

## Por Que Kubernetes?

Sejamos honestos: se você está gerenciando 3 servidores, não precisa do Kubernetes. Um bom conjunto de playbooks Ansible e units do systemd vai te atender bem.

Mas no momento em que você precisa:

- **Executar a mesma aplicação em 50 máquinas** sem configurar cada uma manualmente
- **Lidar com falhas automaticamente** — se um processo morre, ele reinicia; se uma máquina morre, a carga de trabalho é movida
- **Escalar para cima e para baixo** baseado na demanda real, não no seu melhor palpite às 2h da manhã
- **Implantar novas versões** sem downtime e fazer rollback se algo quebrar
- **Dar a diferentes equipes** ambientes isolados em infraestrutura compartilhada

...é quando gerenciar servidores individuais para de escalar e você começa a pensar em termos de clusters.

O Kubernetes é o que acontece quando você pega as melhores ideias da administração de sistemas Linux — gerenciamento de processos, rede, armazenamento, segurança — e as aplica em uma frota de máquinas em vez de uma só.

### Uma Breve História (Para Contexto, Não Curiosidade)

O Kubernetes não foi a primeira tentativa de orquestração de containers, mas foi a que vingou:

- **2003-2013**: Google roda o Borg internamente — gerenciando milhões de containers em sua infraestrutura, tudo em Linux
- **2013**: Docker torna containers acessíveis a todos (antes do Docker, você precisava de conhecimento profundo de Linux para usar namespaces e cgroups diretamente)
- **2014**: Google abre o código do Kubernetes, destilando lições do Borg em um projeto comunitário
- **2015**: Kubernetes 1.0 é lançado; Cloud Native Computing Foundation (CNCF) é formada
- **2018-presente**: Kubernetes se torna o padrão de fato para orquestração de containers; todos os principais provedores de nuvem oferecem Kubernetes gerenciado (AKS, EKS, GKE)

O insight principal: o Kubernetes nasceu do Linux. As pessoas que o construíram eram administradores de sistemas Linux e engenheiros de kernel. Quando você aprende Kubernetes, está aprendendo a próxima evolução de habilidades que já possui.

## A Mudança Mental: Do Servidor ao Cluster

A parte mais difícil de aprender Kubernetes não é a tecnologia — é a mudança de mentalidade.

### Em um Servidor Linux, Você Pensa:

- "Preciso instalar o nginx **nesta máquina**"
- "Preciso abrir a porta 80 no firewall **desta máquina**"
- "Preciso montar **/dev/sdb1** em **/var/www**"
- "Se o serviço cair, vou configurar **uma política de restart no systemd**"

### No Kubernetes, Você Pensa:

- "Preciso de **3 cópias** do nginx rodando **em algum lugar** no cluster"
- "Preciso de um **Service** que roteie tráfego para qualquer Pod nginx saudável"
- "Preciso de **armazenamento persistente** que sobreviva a reinícios de Pod, independente de em qual node ele rode"
- "Se um Pod cair, o **Deployment controller** vai substituí-lo automaticamente"

Perceba a mudança: você para de se preocupar com *qual máquina específica* roda sua carga de trabalho. Você declara *o que quer* (3 réplicas do nginx, armazenamento persistente, um endpoint de rede), e o Kubernetes descobre o *onde* e o *como*.

Esta é a mudança fundamental: **de gerenciamento imperativo de servidores individuais para gerenciamento declarativo de um cluster.**

> **Onde a Analogia com Linux Quebra**
>
> Em um servidor Linux, você faz SSH, executa comandos e vê resultados imediatos. É imperativo: você diz ao sistema exatamente o que fazer, passo a passo.
>
> O Kubernetes é declarativo: você descreve o estado final desejado, e um conjunto de controllers trabalha continuamente para fazer a realidade corresponder à sua descrição. Não existe "SSH no cluster e instalar o nginx." Em vez disso, você envia um manifesto YAML que diz "Eu quero 3 Pods nginx," e o Kubernetes os cria, monitora e substitui se falharem.
>
> Essa mudança de "eu faço coisas" para "eu descrevo o que quero" é o maior ajuste mental. Todo o resto deriva disso.

## O Que Este Livro Cobre

Este livro está organizado como uma jornada progressiva, começando do que você já sabe e construindo em direção a um Kubernetes pronto para produção:

**Parte I — Fundamentos (Capítulos 1-5)**

Você vai fazer a ponte do Linux para containers, entender a arquitetura do Kubernetes, aprender como o Kubernetes "pensa" diferente de um servidor único, e colocar seu primeiro cluster para rodar.

**Parte II — Conceitos Centrais (Capítulos 6-10)**

A essência do Kubernetes: workloads, rede, armazenamento, configuração e empacotamento. Cada capítulo mapeia diretamente para algo que você já faz em servidores Linux.

**Parte III — Operações (Capítulos 11-15)**

Segurança, escalabilidade, observabilidade, troubleshooting e prontidão para produção. É aqui que os instintos operacionais de Linux realmente compensam — e onde as diferenças mais importam.

Cada capítulo inclui:
- **Tabelas de comparação Linux-para-K8s** — referência rápida para mapeamento de conceitos
- **Caixas "Onde a Analogia Quebra"** — honestas sobre os limites das analogias com Linux
- **Labs de diagnóstico** — exercícios práticos usando Kind (Kubernetes local)
- **Principais conclusões** — o que lembrar de cada capítulo

## Lab: Verifique Seus Pré-requisitos

Antes de mergulhar, vamos garantir que seu ambiente está pronto. Você vai precisar de Docker e Kind instalados — ambos rodam em Linux (ou WSL2 no Windows).

### Passo 1: Verificar o Docker

```bash
docker version
```

Você deve ver as versões do Client e do Server. Se o Docker não estiver instalado, siga o [guia oficial de instalação](https://docs.docker.com/engine/install/).

### Passo 2: Instalar o Kind

Kind (Kubernetes IN Docker) roda um cluster Kubernetes completo dentro de containers Docker. É a maneira mais rápida de ter um cluster local:

```bash
# Para Linux (amd64)
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.31.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Verifique a instalação:

```bash
kind version
```

> **Fonte:** [Kind Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)

### Passo 3: Instalar o kubectl

kubectl é a ferramenta de linha de comando para interagir com clusters Kubernetes — pense nele como seu `systemctl` para o cluster:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
```

Verifique:

```bash
kubectl version --client
```

> **Fonte:** [Instalar kubectl no Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

### Passo 4: Criar Seu Primeiro Cluster

```bash
kind create cluster --name from-server-to-cluster
```

Isso cria um cluster Kubernetes de um único node rodando dentro de um container Docker. Você deve ver uma saída terminando com:

```
Set kubectl context to "kind-from-server-to-cluster"
```

### Passo 5: Verificar o Cluster

```bash
kubectl cluster-info --context kind-from-server-to-cluster
```

Saída esperada:

```
Kubernetes control plane is running at https://127.0.0.1:<port>
CoreDNS is running at https://127.0.0.1:<port>/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

Agora explore o que está rodando — este é um cluster Kubernetes completo, e seus componentes são processos Linux reais:

```bash
kubectl get nodes
kubectl get pods -A
```

Você verá velhos conhecidos: `etcd`, `kube-apiserver`, `kube-scheduler`, `kube-controller-manager`, `coredns`, `kube-proxy` — todos rodando como containers, que são apenas processos Linux com namespaces e cgroups.

### Limpeza (Opcional)

Se quiser deletar o cluster e recriá-lo depois:

```bash
kind delete cluster --name from-server-to-cluster
```

## Principais Conclusões

1. **Kubernetes é uma evolução das habilidades Linux**, não uma substituição. Todo conceito central mapeia para algo que você já conhece.
2. **A mudança fundamental** é de imperativo (SSH e executar comandos) para declarativo (descrever o estado desejado em YAML).
3. **Você para de pensar em servidores individuais** e começa a pensar em clusters — o que você quer rodando, não onde roda.
4. **Kind** te dá um cluster Kubernetes completo localmente para aprendizado — sem necessidade de conta em nuvem.
5. **A abordagem deste livro**: começar com o que você sabe em Linux, mapear para Kubernetes, e honestamente sinalizar onde a analogia quebra.

## Leitura Adicional

- [Kubernetes Documentation — Overview](https://kubernetes.io/docs/concepts/overview/)
- [Kind — Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [kubectl Installation — Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
- [The History of Kubernetes (CNCF)](https://www.cncf.io/blog/2018/07/19/the-history-of-kubernetes-the-community-behind-it/)

---

**Próximo:** [Capítulo 2 — Containers Desmistificados](02-containers-demystified.md) — Os fundamentos Linux (namespaces, cgroups) que tornam containers possíveis, e por que entendê-los te dá uma vantagem.
