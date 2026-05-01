# Capítulo 2: Containers Desmistificados

*"Um container é apenas um processo Linux que pensa que está sozinho na máquina."*

---

## Não Existe "Container" de Verdade

Esse título pode parecer provocativo em um livro de Kubernetes, mas é a coisa mais importante que você vai aprender neste capítulo: **não existe uma primitiva container no kernel Linux.** Não há uma chamada de sistema `container_create()`. Nenhuma entrada "container" em `/proc`.

O que chamamos de "container" é na verdade um processo Linux comum com três coisas extras aplicadas:

1. **Namespaces** — para que o processo pense que tem sua própria árvore de PIDs, pilha de rede e sistema de arquivos
2. **cgroups** — para que o processo só possa usar uma quantidade definida de CPU e memória
3. **Um sistema de arquivos raiz** (de uma imagem) — para que o processo veja um `/` completamente diferente do host

É isso. Se você já usou `chroot`, já usou uma forma primitiva de containerização. Se já configurou cgroups para um serviço, já fez limitação de recursos. Docker, containerd e Kubernetes não inventaram essas funcionalidades — eles as tornaram *usáveis*.

Como profissional Linux, você já entende os blocos de construção. Este capítulo os desmonta, mostra como se encaixam e revela o que o Docker (e eventualmente o Kubernetes) realmente está fazendo por baixo dos panos.

## Namespaces Linux: A Camada de Isolamento

Namespaces são a funcionalidade do kernel que dá a um processo sua própria visão isolada do sistema. Pense neles como espelhos unidirecionais: o host pode ver tudo, mas o processo em namespace só vê seu próprio pequeno mundo.

O Linux oferece vários tipos de namespace, cada um isolando um recurso diferente do sistema:

| Namespace | Flag | O Que Isola | Analogia Linux |
|-----------|------|------------------|---------------|
| **PID** | `CLONE_NEWPID` | IDs de processo — o container vê seu entrypoint como PID 1 | Como um boot limpo onde seu processo é o `init` |
| **NET** | `CLONE_NEWNET` | Interfaces de rede, endereços IP, tabelas de roteamento, portas | Como uma máquina separada com seu próprio `eth0` |
| **MNT** | `CLONE_NEWNS` | Pontos de montagem — o container tem sua própria árvore de sistema de arquivos | Como `chroot` turbinado |
| **UTS** | `CLONE_NEWUTS` | Hostname e nome de domínio | Como definir um hostname diferente por serviço |
| **IPC** | `CLONE_NEWIPC` | Comunicação entre processos (memória compartilhada, semáforos) | Como domínios System V IPC separados |
| **USER** | `CLONE_NEWUSER` | IDs de usuário e grupo — root dentro pode não ser root fora | Como ter um `/etc/passwd` separado por processo |
| **cgroup** | `CLONE_NEWCGROUP` | Diretório raiz de cgroup — o container vê sua própria hierarquia de cgroup como raiz | Como uma visão privada de `/sys/fs/cgroup/` |

Quando o Docker inicia um container, ele está chamando as mesmas APIs do kernel que `unshare` e `clone()` usam. A "mágica" dos containers é apenas criação de namespace — algo que o Linux suporta desde o kernel 2.6.24 (2008) e que amadureceu com user namespaces no kernel 3.8 (2013).

### Como os Namespaces Trabalham Juntos

Um container típico combina todos os sete tipos de namespace. O processo interno:
- Se vê como PID 1 (PID namespace)
- Tem suas próprias interfaces `lo` e `eth0` (NET namespace)
- Vê um sistema de arquivos raiz completamente diferente (MNT namespace)
- Tem seu próprio hostname (UTS namespace)
- Não pode acessar a memória compartilhada do host (IPC namespace)
- Pode até pensar que está rodando como root, mas na verdade tem privilégios limitados no host (USER namespace)

Da **perspectiva do host**, esse mesmo processo é apenas mais uma entrada no `ps aux` com um PID normal. Não há nada especial nele — é um processo Linux normal com alguns metadados extras do kernel restringindo o que ele pode ver.

## cgroups v2: O Limitador de Recursos

Enquanto namespaces lidam com **isolamento** (o que um processo pode ver), cgroups lidam com **limites de recursos** (o que um processo pode usar). cgroups — abreviação de "control groups" — permitem definir limites rígidos em CPU, memória, I/O e mais.

Sistemas Linux modernos usam **cgroups v2** (hierarquia unificada), que organiza o controle de recursos sob uma única árvore em `/sys/fs/cgroup/`. Aqui está o que os principais controllers fazem:

| Controller | Arquivo de Interface | O Que Controla |
|-----------|---------------|-----------------|
| **CPU** | `cpu.max` | Tempo máximo de CPU (quota/período em microssegundos) |
| **CPU** | `cpu.weight` | Compartilhamento proporcional de CPU (substitui o `cpu.shares` do v1) |
| **Memory** | `memory.max` | Limite rígido de memória — OOM-killed se excedido |
| **Memory** | `memory.low` | Proteção de memória best-effort (garantia soft) |
| **I/O** | `io.max` | Limites de taxa de I/O por dispositivo (bytes/s, IOPS) |
| **PIDs** | `pids.max` | Número máximo de processos no grupo |

Quando você executa `docker run --memory=128m --cpus=0.5`, o Docker está fazendo exatamente isso por baixo dos panos:
- Escrevendo `134217728` (128 MB em bytes) em `memory.max`
- Escrevendo `50000 100000` em `cpu.max` (50ms de CPU a cada 100ms = 0.5 CPUs)

Não há mágica do Docker aqui — são interfaces puras do kernel.

## Imagens OCI: Sistemas de Arquivos Imutáveis e em Camadas

Agora que cobrimos isolamento (namespaces) e limites de recursos (cgroups), a terceira peça é o **sistema de arquivos**. Um container precisa ver um sistema de arquivos raiz (`/`) com todos os seus binários, bibliotecas e arquivos de configuração. É isso que uma imagem OCI (Open Container Initiative) fornece.

### Como as Camadas Funcionam

Uma imagem não é um único arquivo — é uma pilha de **camadas somente-leitura**, cada uma representando um conjunto de mudanças no sistema de arquivos. Pense nisso como controle de versão para sistemas de arquivos:

```
Layer 4: COPY app.py /app/           ← Código da sua aplicação
Layer 3: RUN pip install flask        ← Pacotes instalados
Layer 2: RUN apt-get install python3  ← Runtime do Python
Layer 1: FROM ubuntu:24.04            ← Sistema de arquivos base do SO
```

Cada camada armazena apenas o **diff** da camada abaixo. Este design tem enormes vantagens:

- **Camadas compartilhadas**: Se 10 containers usam `ubuntu:24.04`, essa camada base é armazenada uma vez no disco e compartilhada entre todos
- **Builds rápidos**: Mudou o código da aplicação? Apenas a Layer 4 é reconstruída — layers 1-3 estão em cache
- **Imutabilidade**: Camadas são somente-leitura. Quando um container escreve arquivos, essas escritas vão para uma fina **camada gravável** por cima (usando um sistema de arquivos union como OverlayFS)

### O Dockerfile: Construindo Imagens

Um Dockerfile é uma receita para construir uma imagem, camada por camada. Aqui estão as instruções principais:

| Instrução | Propósito | Analogia Linux |
|-------------|---------|---------------|
| `FROM` | Define a imagem base (sistema de arquivos inicial) | Como escolher qual distro instalar |
| `RUN` | Executa um comando durante o build (cria uma nova camada) | Como executar um comando em um script de provisionamento |
| `COPY` | Copia arquivos da sua máquina para dentro da imagem | Como `cp` ou `scp` durante a configuração |
| `ENV` | Define variáveis de ambiente | Como adicionar ao `/etc/environment` |
| `EXPOSE` | Documenta em qual porta a aplicação escuta | Como um comentário na sua config de firewall (não abre portas de verdade) |
| `CMD` | Comando padrão quando o container inicia | Como `ExecStart=` em uma unit do systemd |
| `ENTRYPOINT` | O executável principal (CMD se torna seus argumentos) | Como o caminho do binário em `ExecStart=`, com CMD como os argumentos |

**Distinção importante:** `CMD` fornece padrões que podem ser sobrescritos em tempo de execução (`docker run myimage /bin/sh`). `ENTRYPOINT` define o executável que sempre roda, e `CMD` apenas fornece argumentos padrão para ele. A maioria das imagens de produção usa `ENTRYPOINT` para o binário principal e `CMD` para flags padrão.

## Container Runtimes: A Pilha

Quando você executa `docker run nginx`, um número surpreendente de componentes está envolvido. Entender a pilha ajuda a desmistificar o que está realmente acontecendo:

```
┌──────────────────────────────────────────────────┐
│  Docker CLI (docker)                             │
│  Você digita comandos aqui                       │
├──────────────────────────────────────────────────┤
│  Docker Daemon (dockerd)                         │
│  Gerencia imagens, redes, volumes                │
├──────────────────────────────────────────────────┤
│  containerd                                      │
│  Runtime de alto nível: gerenciamento de imagens,│
│  ciclo de vida de containers, CRI para Kubernetes│
├──────────────────────────────────────────────────┤
│  runc                                            │
│  Runtime OCI de baixo nível: realmente chama     │
│  clone(), configura namespaces, cgroups, exec's  │
│  o processo do container                         │
├──────────────────────────────────────────────────┤
│  Kernel Linux                                    │
│  namespaces, cgroups, seccomp, capabilities      │
└──────────────────────────────────────────────────┘
```

Aqui está o insight principal: **Kubernetes não usa Docker.** Desde o Kubernetes v1.24, o suporte ao Docker (dockershim) foi removido. O Kubernetes fala diretamente com o **containerd** via Container Runtime Interface (CRI). O containerd então usa **runc** para criar o processo do container.

Docker é uma ferramenta de desenvolvimento. containerd é o runtime de produção. runc é o motor que fala com o kernel. Entender essa pilha importa porque quando você faz troubleshooting de problemas com containers no Kubernetes, está debugando containerd e runc — não Docker.

## Tabela de Comparação Linux ↔ Container

| Conceito Linux | Equivalente em Container | Como Se Mapeia |
|---------------|---------------------|-------------|
| `chroot` | Mount namespace + rootfs | Containers usam um mount namespace completo, não apenas uma raiz alterada |
| `unshare` | Criação de namespace | Docker/runc chama `clone()` com flags de namespace, assim como `unshare` |
| cgroups (`/sys/fs/cgroup/`) | Limites de recursos (`--memory`, `--cpus`) | Mesma funcionalidade do kernel — Docker apenas escreve nos arquivos de cgroup para você |
| Processo (`/proc/<pid>/`) | Container | Um container *é* um processo — ele tem um PID no host |
| Sistema de arquivos (`/`, `/usr`, `/etc`) | Camadas de imagem (OverlayFS) | Em camadas e somente-leitura, diferente de um sistema de arquivos tradicional mutável |
| PID 1 (`init`/`systemd`) | Entrypoint do container | O processo entrypoint deve continuar rodando — se ele sair, o container para |
| `fork()`/`exec()` | `docker run` | Cria um novo processo com isolamento, apenas com mais passos |

> **Onde a Analogia com Linux Quebra**
>
> - **Imagens são imutáveis e em camadas.** Em um servidor Linux, você pode modificar qualquer arquivo diretamente. Imagens de container são construídas a partir de camadas somente-leitura — escritas em tempo de execução vão para uma camada gravável separada que é descartada quando o container para. Essa imutabilidade é uma funcionalidade, não uma limitação: ela garante que todo container da mesma imagem inicia de forma idêntica.
>
> - **O entrypoint deve continuar rodando.** Em um servidor Linux, se um serviço cai, o systemd o reinicia. Em um container, se o PID 1 sai, o container morre imediatamente. Não há sistema init dentro para reiniciar. (Esse é o trabalho do Kubernetes — ele reinicia o *container*, não o processo dentro dele.)
>
> - **A rede é completamente isolada por padrão.** Processos do host compartilham a mesma pilha de rede — todos podem fazer bind em portas, ver as mesmas interfaces e se alcançar via `localhost`. Um container recebe seu próprio network namespace com suas próprias interfaces. Para alcançar o container a partir do host, você precisa de mapeamento explícito de portas (`-p 8080:80`) ou uma rede de containers.

## Lab de Diagnóstico: Containers do Zero

Este lab mostra que containers são apenas funcionalidades do Linux — nenhuma mágica necessária. Você vai criar namespaces manualmente, construir uma imagem, inspecionar suas camadas e provar que containers são apenas processos.

### Pré-requisitos

- Uma máquina Linux (ou WSL2 no Windows)
- Docker instalado e rodando
- Acesso root ou sudo (para a seção de `unshare`)

### Exercício 1: Criar um Namespace Manualmente

Este exercício prova que "containers" são apenas funcionalidades do kernel Linux. Usaremos `unshare` para criar namespaces isolados — a mesma coisa que o Docker faz por baixo dos panos.

**Crie um novo PID e mount namespace:**

```bash
sudo unshare --pid --mount --fork bash
```

Dentro deste novo namespace, você está em um espaço de PID isolado. Vamos provar:

```bash
# Monte um /proc limpo para este PID namespace
mount -t proc proc /proc

# Verifique — seu shell é PID 1 neste namespace!
echo $$

# Liste todos os processos — você só verá processos DESTE namespace
ps aux
```

Você deve ver muito poucos processos — apenas seu shell e o próprio `ps`. Enquanto isso, no host (em outro terminal), a lista completa de processos é visível. Isso é exatamente o que um container vê.

**Saia do namespace:**

```bash
exit
```

> **O que acabou de acontecer?** Você usou `unshare` para criar novos PID e mount namespaces — as mesmas funcionalidades do kernel que Docker e containerd usam. A única diferença é que o Docker também configura cgroups, rede, um sistema de arquivos raiz baseado em imagem, e faz tudo através de uma API conveniente.
>
> **Fonte:** [unshare(1) — Linux manual page](https://man7.org/linux/man-pages/man1/unshare.1.html)

### Exercício 2: Construir uma Imagem Docker

Crie um diretório de trabalho e um Dockerfile simples:

```bash
mkdir -p ~/container-lab && cd ~/container-lab
```

Crie um Dockerfile:

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
COPY <<EOF /app/hello.sh
#!/bin/bash
echo "Hello from container! Hostname: $(hostname), PID: $$"
sleep infinity
EOF
RUN chmod +x /app/hello.sh
CMD ["/app/hello.sh"]
```

Construa a imagem:

```bash
docker build -t my-container-lab:v1 .
```

> **Fonte:** [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)

### Exercício 3: Inspecionar Camadas da Imagem

Cada instrução do Dockerfile que modifica o sistema de arquivos cria uma nova camada. Vamos vê-las:

```bash
# Mostrar o histórico de camadas — cada linha é uma camada
docker history my-container-lab:v1
```

Você verá cada camada com seu tamanho e o comando que a criou. Note como `FROM ubuntu:24.04` é a maior camada e seu comando `COPY` é minúsculo.

Agora inspecione os metadados completos da imagem:

```bash
# Mostrar informações detalhadas da imagem incluindo digests das camadas
docker inspect my-container-lab:v1 --format '{{json .RootFS.Layers}}' | python3 -m json.tool
```

Cada hash SHA256 representa uma camada. Imagens compartilhando a mesma imagem base compartilharão as mesmas camadas inferiores — é assim que o Docker economiza espaço em disco.

> **Fonte:** [docker history](https://docs.docker.com/reference/cli/docker/image/history/) | [docker inspect](https://docs.docker.com/reference/cli/docker/inspect/)

### Exercício 4: Limites de Recursos com cgroups

Execute um container com limites de memória e CPU — o Docker traduz estes para configurações de cgroup:

```bash
docker run -d --name limited-container \
  --memory=128m \
  --cpus=0.5 \
  my-container-lab:v1
```

Agora inspecione os limites de cgroup que o Docker configurou:

```bash
# Verificar o limite de memória (em bytes — 128MB = 134217728)
docker exec limited-container cat /sys/fs/cgroup/memory.max

# Verificar a quota de CPU (50000 100000 = 50ms a cada 100ms = 0.5 CPUs)
docker exec limited-container cat /sys/fs/cgroup/cpu.max
```

Estes são os mesmos arquivos de cgroup que você escreveria manualmente no host. O Docker apenas fornece uma interface conveniente.

Limpeza:

```bash
docker stop limited-container && docker rm limited-container
```

> **Fonte:** [docker run — resource constraints](https://docs.docker.com/reference/cli/docker/container/run/#memory) | [cgroups v2 — Linux kernel docs](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)

### Exercício 5: Containers São Apenas Processos

Este é o momento "eureka". Execute um container e prove que é apenas um processo no host:

```bash
# Execute um container
docker run -d --name process-proof my-container-lab:v1

# Dentro do container, o entrypoint é PID 1
docker exec process-proof ps aux
```

Agora olhe a lista de processos do **host**:

```bash
# Encontre o processo do container no HOST — ele tem um PID normal!
docker top process-proof
```

A saída do `docker top` mostra os processos do container como o **host os vê** — com PIDs reais do host. O processo do container pensa que é PID 1, mas o host sabe seu PID real.

Você pode até encontrá-lo com um simples `ps`:

```bash
# Obter o PID do container no host
docker inspect process-proof --format '{{.State.Pid}}'

# Verificar — este é um processo normal visível no host
ps -p $(docker inspect process-proof --format '{{.State.Pid}}') -o pid,ppid,cmd
```

**Esta é a revelação fundamental:** Um container não é uma VM. Não é um kernel separado. É um processo Linux com namespaces e cgroups aplicados. O kernel do host roda tudo.

Limpeza:

```bash
docker stop process-proof && docker rm process-proof
rm -rf ~/container-lab
```

> **Fonte:** [docker top](https://docs.docker.com/reference/cli/docker/container/top/) | [docker inspect](https://docs.docker.com/reference/cli/docker/inspect/)

## Principais Conclusões

1. **Um container é um processo Linux** com namespaces (isolamento), cgroups (limites de recursos) e um sistema de arquivos raiz baseado em imagem. Não existe uma primitiva "container" no nível do kernel.
2. **Namespaces Linux** fornecem seis tipos de isolamento: PID, NET, MNT, UTS, IPC e USER. Cada um esconde um aspecto diferente do sistema host do processo containerizado.
3. **cgroups v2** impõem limites de recursos. Quando você define `--memory=128m`, o Docker escreve no mesmo arquivo `/sys/fs/cgroup/memory.max` que você poderia escrever manualmente.
4. **Imagens OCI são em camadas e imutáveis.** Cada instrução do Dockerfile cria uma camada somente-leitura. Camadas base compartilhadas economizam espaço em disco, e a imutabilidade garante deploys consistentes.
5. **A pilha do container runtime é Docker CLI → dockerd → containerd → runc → kernel Linux.** O Kubernetes pula o Docker inteiramente e fala com o containerd via CRI.
6. **Entrypoints de container são PID 1.** Se o processo entrypoint sai, o container para. Não há sistema init dentro para reiniciar serviços — essa responsabilidade passa para o orquestrador (Kubernetes).
7. **Entender esses fundamentos Linux te dá um superpoder de debugging.** Quando um container se comporta mal, você não está lutando contra mágica — está debugando namespaces, cgroups e comportamento de processos, que você já entende.

## Leitura Adicional

- [Linux Namespaces — man7.org](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [unshare(1) — Linux manual page](https://man7.org/linux/man-pages/man1/unshare.1.html)
- [Control Group v2 — Linux kernel docs](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
- [OCI Image Specification](https://github.com/opencontainers/image-spec)
- [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec)
- [Dockerfile Reference](https://docs.docker.com/reference/dockerfile/)
- [containerd — CNCF Project](https://containerd.io/)
- [Docker CLI Reference](https://docs.docker.com/reference/cli/docker/)

---

**Anterior:** [Capítulo 1 — Do Servidor ao Cluster](01-from-server-to-cluster.md)

**Próximo:** [Capítulo 3 — Arquitetura do Kubernetes](03-kubernetes-architecture.md) — Os componentes do control plane e worker node, mapeados para os serviços e daemons Linux que você já conhece.
