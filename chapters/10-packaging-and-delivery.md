# Capítulo 10: Empacotamento e Entrega

*"Ninguém quer gerenciar mil arquivos YAML manualmente. Isso não é DevOps — isso é sofrimento."*

---

## De YAML Artesanal a Deploys Repetíveis

Em um servidor Linux, você não instala software copiando binários manualmente e escrevendo arquivos de configuração do zero. Você usa um gerenciador de pacotes — `apt install nginx`, `dnf install postgresql`, `brew install redis`. O gerenciador de pacotes cuida de dependências, versionamento, upgrades, rollbacks e desinstalações. É civilizado. É repetível. É como profissionais trabalham.

Agora olhe o que fizemos no Kubernetes nos últimos nove capítulos: escrever arquivos YAML individuais à mão, aplicá-los um por vez com `kubectl apply -f`. Para um único serviço com um Deployment, um Service, um ConfigMap e um Secret, são quatro arquivos YAML. Adicione um Ingress, um PVC, uma ServiceAccount e regras RBAC, e você tem oito. Multiplique por três ambientes (dev, staging, produção) e terá vinte e quatro arquivos para gerenciar. Para um serviço.

Em uma arquitetura real de microsserviços com dezenas de serviços, estamos falando de centenas de arquivos YAML. Ambientes diferentes precisam de contagens de réplicas diferentes, tags de imagem diferentes, limites de recursos diferentes, valores de ConfigMap diferentes. Copiar e editar arquivos YAML entre diretórios é o equivalente no Kubernetes a gerenciar servidores fazendo SSH e editando arquivos de configuração manualmente — funciona até que não funciona mais, e então falha espetacularmente.

Este capítulo apresenta as ferramentas que resolvem esse problema: **Helm** (o gerenciador de pacotes), **Kustomize** (o motor de patches) e o conceito de **GitOps** (o modelo de deploy). Juntos, eles transformam o Kubernetes de "artesanato YAML" em um pipeline de entrega repetível, auditável e pronto para produção.

---

## Helm — O Gerenciador de Pacotes do Kubernetes

Helm está para o Kubernetes assim como o `apt` está para o Debian ou o `dnf` para o Fedora. Ele empacota múltiplos recursos Kubernetes em uma única unidade versionada e configurável chamada **chart**.

### Conceitos Fundamentais

| Termo Helm | Equivalente Linux | O Que É |
|-----------|-----------------|------------|
| **Chart** | Pacote `.deb` ou `.rpm` | Um bundle de manifestos Kubernetes com templates |
| **Release** | Um pacote instalado | Uma instância específica de um chart rodando em um cluster |
| **Values** | Respostas do `debconf` / dpkg-reconfigure | Parâmetros de configuração passados no momento da instalação |
| **Repository** | Lista de fontes APT (`/etc/apt/sources.list`) | Um servidor que hospeda charts |
| **Template** | Templates Jinja2 / ERB | Templates Go que geram YAML a partir de values |
| **Revision** | Histórico de versões de pacotes | Cada `helm upgrade` cria uma nova revision |

### Como o Helm Funciona

Um chart Helm é um diretório com uma estrutura específica:

```
my-chart/
├── Chart.yaml          # Metadata (nome, versão, descrição)
├── values.yaml         # Valores de configuração padrão
├── templates/          # Arquivos de template Go que geram manifestos K8s
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── _helpers.tpl    # Funções auxiliares de template
└── charts/             # Dependências (sub-charts)
```

O diretório `templates/` contém manifestos Kubernetes com placeholders de template Go. Por exemplo, `templates/deployment.yaml` pode ser assim:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.port }}
```

E o `values.yaml` fornece os valores padrão:

```yaml
replicaCount: 1
image:
  repository: nginx
  tag: "1.27"
service:
  port: 80
  type: ClusterIP
```

Quando você executa `helm install`, o Helm renderiza os templates substituindo os valores, produzindo YAML puro do Kubernetes que é aplicado ao cluster. Você pode sobrescrever qualquer valor no momento da instalação com `--set` ou `--values`.

### Repositórios Helm e Artifact Hub

Charts são distribuídos através de repositórios — servidores HTTP (ou registros OCI) hospedando charts empacotados. Você adiciona um repositório, atualiza o índice e busca charts assim como faria com o `apt`:

| Comando APT | Equivalente Helm |
|-------------|----------------|
| `add-apt-repository ppa:...` | `helm repo add bitnami https://charts.bitnami.com/bitnami` |
| `apt update` | `helm repo update` |
| `apt search nginx` | `helm search repo nginx` |
| `apt show nginx` | `helm show chart bitnami/nginx` |
| `apt install nginx` | `helm install my-nginx bitnami/nginx` |

O [Artifact Hub](https://artifacthub.io/) é o diretório público para encontrar charts Helm — pense nele como o equivalente Kubernetes do `packages.ubuntu.com` ou `hub.docker.com` para charts. Ele indexa charts de múltiplos repositórios e fornece busca, documentação e histórico de versões.

### Comandos de Ciclo de Vida do Helm

O fluxo de trabalho principal:

```bash
# Instalar um chart (cria uma release)
helm install <release-name> <chart> [--set key=value] [--values custom.yaml]

# Listar releases instaladas
helm list

# Mostrar valores atuais de uma release
helm get values <release-name>

# Atualizar uma release (nova revision)
helm upgrade <release-name> <chart> [--set key=value]

# Rollback para uma revision anterior
helm rollback <release-name> <revision-number>

# Ver histórico da release
helm history <release-name>

# Desinstalar uma release
helm uninstall <release-name>

# Renderizar templates localmente sem instalar (ótimo para debug)
helm template <release-name> <chart> [--set key=value]
```

O comando `helm template` é particularmente útil — ele renderiza o chart em YAML puro sem tocar no cluster. É assim que você debugga problemas de template ou alimenta a saída do Helm em outras ferramentas (como Kustomize, que veremos a seguir).

---

## Kustomize — Patches Sem Templates

Kustomize adota uma abordagem fundamentalmente diferente do Helm. Em vez de templates com placeholders, o Kustomize trabalha com YAML puro e válido do Kubernetes e aplica patches por cima. Não há linguagem de templates, sem sintaxe especial — apenas transformações YAML.

### O Modelo Base + Overlays

O Kustomize usa uma estrutura em camadas:

```
app/
├── base/                    # Os manifestos base (YAML válido)
│   ├── kustomization.yaml   # Declara quais recursos incluir
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/                 # Patches específicos para dev
    │   └── kustomization.yaml
    └── prod/                # Patches específicos para prod
        └── kustomization.yaml
```

A **base** contém seus manifestos Kubernetes padrão — sem placeholders, sem sintaxe especial. São os mesmos arquivos YAML que você aplicaria com `kubectl apply -f`.

**Overlays** contêm patches que modificam a base para um ambiente específico. O overlay de dev pode reduzir réplicas para 1 e usar uma tag de imagem `latest`. O overlay de prod pode aumentar réplicas para 5, definir limites de recursos e adicionar regras de pod anti-affinity.

Isso é como alternatives e symlinks no Linux — o software base é o mesmo, mas a configuração específica do sistema (qual binário usar, qual arquivo de configuração carregar) varia.

### Como o Kustomize Funciona

O arquivo `kustomization.yaml` em cada diretório diz ao Kustomize quais recursos incluir e quais transformações aplicar:

**base/kustomization.yaml:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
```

**overlays/dev/kustomization.yaml:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
patches:
- patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-app
    spec:
      replicas: 1
namePrefix: dev-
namespace: dev
```

**overlays/prod/kustomization.yaml:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
patches:
- patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-app
    spec:
      replicas: 5
namePrefix: prod-
namespace: production
```

Compilar e aplicar:

```bash
# Pré-visualizar o que o overlay de dev produz
kubectl kustomize overlays/dev

# Pré-visualizar o que o overlay de prod produz
kubectl kustomize overlays/prod

# Aplicar o overlay de dev
kubectl apply -k overlays/dev

# Aplicar o overlay de prod
kubectl apply -k overlays/prod
```

A flag `-k` está integrada no kubectl — o Kustomize é integrado diretamente, sem necessidade de instalação separada. O comando `kubectl kustomize` renderiza a saída sem aplicar, o que é útil para revisão e debug.

### O Que o Kustomize Pode Fazer

| Transformação | Descrição |
|----------------|-------------|
| `namePrefix` / `nameSuffix` | Adicionar prefixo/sufixo a todos os nomes de recursos |
| `namespace` | Sobrescrever o namespace para todos os recursos |
| `commonLabels` | Adicionar labels a todos os recursos e selectors |
| `commonAnnotations` | Adicionar annotations a todos os recursos |
| `images` | Sobrescrever nomes e tags de imagens sem patching |
| `patches` | Strategic merge patches ou JSON patches |
| `configMapGenerator` | Gerar ConfigMaps a partir de arquivos ou literais |
| `secretGenerator` | Gerar Secrets a partir de arquivos ou literais |

A transformação `images` é particularmente limpa — você pode alterar tags de imagens em todos os deployments sem escrever um patch:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
images:
- name: my-app
  newTag: v2.1.0
```

---

## Helm vs. Kustomize — Quando Usar Qual

Ambas as ferramentas resolvem o problema de "gerenciar YAML entre ambientes", mas de ângulos diferentes:

| Aspecto | Helm | Kustomize |
|--------|------|-----------|
| **Abordagem** | Templates com substituição de valores | Patches sobre YAML puro |
| **Distribuição** | Charts são pacotes que você compartilha | Bases são apenas diretórios que você copia ou referencia |
| **Ciclo de vida** | Ciclo completo: instalar, atualizar, rollback, desinstalar | Apenas ferramenta de build — usa `kubectl apply/delete` |
| **Complexidade** | Maior — templates Go, helpers, hooks, dependências | Menor — YAML puro, sem linguagem de templates |
| **Melhor para** | Apps de terceiros, parametrização complexa | Apps internos, customização específica por ambiente |
| **Rastreamento de estado** | Helm rastreia histórico de releases (revisions) | Sem estado — depende do kubectl e Git |

**Use Helm quando:**
- Você está instalando software de terceiros (bancos de dados, stacks de monitoramento, ingress controllers)
- Você precisa distribuir pacotes reutilizáveis para outras equipes
- Você precisa de gerenciamento de ciclo de vida (rollback para revision 3, rastrear quem instalou o quê)
- A aplicação tem muitos parâmetros configuráveis

**Use Kustomize quando:**
- Você está customizando os manifestos da sua própria equipe entre ambientes
- Você quer evitar aprender uma linguagem de templates
- Você precisa fazer mudanças pequenas e direcionadas em YAML existente
- Você quer manter seus manifestos base como YAML Kubernetes válido e legível

**Use ambos juntos quando:**
- Você instala um chart Helm mas precisa de ajustes específicos por ambiente que não são expostos como Helm values
- Você renderiza um chart com `helm template` e aplica patches Kustomize por cima

---

## GitOps — Uma Breve Introdução

Cobrimos como empacotar (Helm) e customizar (Kustomize) seus manifestos Kubernetes. Mas como você os coloca no cluster? Até agora, estávamos executando `kubectl apply` dos nossos laptops. Em produção, isso é problemático:

- Quem aplicou o quê, e quando? Não há rastro de auditoria.
- Alguém fez uma mudança manual com `kubectl edit` que ninguém mais sabe?
- Como você reproduz o estado exato do cluster?

**GitOps** resolve isso tornando o Git a única fonte da verdade para o estado desejado do cluster. O fluxo de trabalho inverte de "push" para "pull":

**Tradicional (push):**
```
Desenvolvedor → kubectl apply → Cluster
```

**GitOps (pull):**
```
Desenvolvedor → git push → Repositório Git ← Controlador GitOps → Cluster
```

Um controlador GitOps (como [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) ou [Flux](https://fluxcd.io/)) roda dentro do cluster e observa continuamente um repositório Git. Quando detecta uma mudança (novo commit, valores atualizados), ele sincroniza o cluster para corresponder ao estado do Git. Se alguém fizer uma mudança manual com `kubectl`, o controlador reverte — o Git sempre vence.

Pense nisso como `ansible-pull` ou `puppet agent` para o Kubernetes — o sistema puxa sua configuração de uma fonte central em vez de ter um administrador empurrando.

### Por Que GitOps Importa

| Benefício | Como |
|---------|-----|
| **Rastro de auditoria** | Toda mudança é um commit Git com autor, timestamp e mensagem |
| **Reprodutibilidade** | O estado do cluster é definido em código, não no histórico do terminal de alguém |
| **Rollback** | `git revert` faz rollback do cluster, não apenas do código |
| **Segurança** | Desenvolvedores fazem push para o Git; não precisam de acesso `kubectl` ao cluster |
| **Detecção de drift** | O controlador alerta quando o estado do cluster difere do Git |

Não vamos construir um pipeline GitOps completo neste capítulo — esse é um tópico que merece seu próprio aprofundamento. Mas entender o conceito é essencial porque Helm e Kustomize são os *blocos de construção* do GitOps. ArgoCD e Flux entendem nativamente tanto charts Helm quanto overlays Kustomize, tornando tudo que cobrimos neste capítulo diretamente aplicável a um fluxo GitOps.

---

## Comparação Linux ↔ Kubernetes

| Conceito Linux | Equivalente K8s | Notas |
|---------------|----------------|-------|
| `apt` / `yum` / `dnf` | Helm | Gerenciador de pacotes com repos, install, upgrade, rollback |
| Pacote `.deb` / `.rpm` | Helm Chart | Bundle reutilizável e versionado de recursos |
| Lista de fontes APT (`sources.list`) | Helm Repository | Onde encontrar charts |
| `debconf` / `dpkg-reconfigure` | `values.yaml` / `--set` | Configuração no momento da instalação |
| `/etc/alternatives`, symlinks | Overlays Kustomize | Variantes específicas por ambiente da mesma base |
| Templates Puppet/Ansible | Templates Helm (Go) | Geração de configuração parametrizada |
| `git` + `ansible-pull` | GitOps (ArgoCD / Flux) | Deploy declarativo baseado em pull |
| `dpkg --list` | `helm list` | Ver o que está instalado |
| `apt rollback` (via snapshots) | `helm rollback` | Reverter para uma versão anterior |

---

> ### 🔀 Onde a Analogia com Linux Quebra
>
> - **Helm é mais que um gerenciador de pacotes.** O `apt` instala pacotes e rastreia o que está instalado. O Helm faz isso *mais* gerencia o estado das releases — ele conhece cada revision de cada release, quais valores foram usados, e pode fazer rollback para qualquer revision anterior. É um gerenciador de pacotes, um rastreador de deploys e um motor de rollback combinados. No Linux, você precisaria do `apt` + `etckeeper` + `timeshift` para ter funcionalidade comparável.
>
> - **Kustomize não "instala" nada.** Não existe `kustomize install` ou `kustomize uninstall`. Kustomize é uma ferramenta de build de YAML — ele transforma manifestos e produz YAML puro. Você ainda usa `kubectl apply` para criar recursos e `kubectl delete` para removê-los. Ele não tem conceito de releases, revisions ou rollback. Pense nele como um pré-processador, não um gerenciador de pacotes.
>
> - **GitOps inverte o modelo de deploy.** O deploy tradicional no Linux é baseado em push: você faz SSH, executa comandos, e o sistema muda. GitOps é baseado em pull: você faz push para o Git, e um controlador dentro do cluster detecta a mudança e a aplica. Se você fizer uma mudança manual com `kubectl edit`, o controlador GitOps vai revertê-la para corresponder ao Git. Esta é uma relação fundamentalmente diferente entre o operador e o sistema — o controlador é o operador, não você.

---

## Laboratório Diagnóstico: Empacotamento na Prática

### Pré-requisitos

Certifique-se de que o Helm está instalado:

```bash
# Verificar se o Helm está instalado
helm version

# Se não estiver instalado, veja https://helm.sh/docs/intro/install/
# No macOS: brew install helm
# No Linux: curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
# No Windows: choco install kubernetes-helm
```

Crie um cluster Kind:

```bash
kind create cluster --name packaging-lab
```

### Lab 1: Básico do Helm

**Passo 1 — Adicionar um repositório de charts:**

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

**Passo 2 — Buscar charts:**

```bash
helm search repo bitnami/nginx
```

Você verá as versões disponíveis do chart nginx da Bitnami.

**Passo 3 — Inspecionar o chart antes de instalar:**

```bash
# Mostrar metadata do chart
helm show chart bitnami/nginx

# Mostrar os valores padrão (isso é o que você pode configurar)
helm show values bitnami/nginx | head -50
```

**Passo 4 — Instalar o chart:**

```bash
helm install my-nginx bitnami/nginx --set service.type=ClusterIP
```

**Passo 5 — Explorar a release:**

```bash
# Listar releases instaladas
helm list

# Mostrar os valores usados para esta release
helm get values my-nginx

# Mostrar todos os recursos criados pela release
helm get manifest my-nginx | head -40

# Verificar os pods e services
kubectl get pods -l app.kubernetes.io/instance=my-nginx
kubectl get svc -l app.kubernetes.io/instance=my-nginx
```

**Passo 6 — Atualizar a release:**

```bash
helm upgrade my-nginx bitnami/nginx \
  --set service.type=ClusterIP \
  --set replicaCount=3

# Verificar que agora temos 3 réplicas
kubectl get pods -l app.kubernetes.io/instance=my-nginx

# Ver histórico da release
helm history my-nginx
```

**Passo 7 — Rollback:**

```bash
# Rollback para revision 1 (a instalação original)
helm rollback my-nginx 1

# Verificar que voltamos para 1 réplica
kubectl get pods -l app.kubernetes.io/instance=my-nginx

# O histórico agora mostra o rollback
helm history my-nginx
```

**Passo 8 — Desinstalar:**

```bash
helm uninstall my-nginx

# Verificar que tudo foi removido
kubectl get pods -l app.kubernetes.io/instance=my-nginx
helm list
```

### Lab 2: Básico do Kustomize

**Passo 1 — Criar os manifestos base:**

```bash
mkdir -p kustomize-demo/base kustomize-demo/overlays/dev kustomize-demo/overlays/prod
```

Crie o Deployment base. Salve como `kustomize-demo/base/deployment.yaml`:

```bash
cat <<'EOF' > kustomize-demo/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: nginx:1.27
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
EOF
```

Crie o Service base. Salve como `kustomize-demo/base/service.yaml`:

```bash
cat <<'EOF' > kustomize-demo/base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF
```

Crie o `kustomization.yaml` base:

```bash
cat <<'EOF' > kustomize-demo/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
EOF
```

**Passo 2 — Criar o overlay de dev:**

```bash
cat <<'EOF' > kustomize-demo/overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
namePrefix: dev-
commonLabels:
  environment: dev
patches:
- patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-app
    spec:
      replicas: 1
EOF
```

**Passo 3 — Criar o overlay de prod:**

```bash
cat <<'EOF' > kustomize-demo/overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
namePrefix: prod-
commonLabels:
  environment: production
patches:
- patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-app
    spec:
      replicas: 5
images:
- name: nginx
  newTag: "1.27-alpine"
EOF
```

**Passo 4 — Comparar as saídas:**

```bash
echo "=== OVERLAY DE DEV ==="
kubectl kustomize kustomize-demo/overlays/dev
echo ""
echo "=== OVERLAY DE PROD ==="
kubectl kustomize kustomize-demo/overlays/prod
```

Note as diferenças: dev tem 1 réplica com prefixo `dev-`; prod tem 5 réplicas com prefixo `prod-` e a tag de imagem alpine. Mesma base, resultados diferentes.

**Passo 5 — Aplicar o overlay de dev:**

```bash
# Criar o namespace de dev
kubectl create namespace dev

# Aplicar o overlay de dev
kubectl apply -k kustomize-demo/overlays/dev -n dev

# Verificar
kubectl get deployment -n dev
kubectl get pods -n dev
kubectl get svc -n dev
```

**Passo 6 — Aplicar o overlay de prod:**

```bash
# Criar o namespace de produção
kubectl create namespace production

# Aplicar o overlay de prod
kubectl apply -k kustomize-demo/overlays/prod -n production

# Verificar — note a diferença em réplicas e tag de imagem
kubectl get deployment -n production
kubectl get pods -n production
kubectl describe deployment prod-my-app -n production | grep Image
```

**Passo 7 — Limpar os recursos do Kustomize:**

```bash
kubectl delete -k kustomize-demo/overlays/dev -n dev
kubectl delete -k kustomize-demo/overlays/prod -n production
kubectl delete namespace dev production
```

### Lab 3: Combinando Helm e Kustomize

Às vezes você quer instalar um chart Helm mas aplicar customizações adicionais que os values do chart não expõem. O padrão: renderizar o chart com `helm template`, depois aplicar patches com Kustomize.

**Passo 1 — Renderizar um chart Helm em YAML:**

```bash
mkdir -p helm-kustomize-demo/base

helm template my-nginx bitnami/nginx \
  --set service.type=ClusterIP \
  > helm-kustomize-demo/base/nginx.yaml
```

**Passo 2 — Criar uma camada Kustomize por cima:**

```bash
cat <<'EOF' > helm-kustomize-demo/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- nginx.yaml
commonAnnotations:
  team: platform-engineering
  managed-by: helm-plus-kustomize
EOF
```

**Passo 3 — Pré-visualizar e aplicar:**

```bash
# Pré-visualizar — você verá que todos os recursos gerados pelo Helm agora têm nossas annotations customizadas
kubectl kustomize helm-kustomize-demo/base | head -30

# Aplicar
kubectl apply -k helm-kustomize-demo/base

# Verificar que as annotations estão presentes
kubectl get deployment -o yaml | grep -A2 "annotations"

# Limpar
kubectl delete -k helm-kustomize-demo/base
```

**O que acabou de acontecer:** O Helm renderizou o chart nginx em YAML puro. O Kustomize adicionou annotations que o chart original não suportava. Este padrão permite usar o Helm para o que ele faz bem (empacotar apps de terceiros) e o Kustomize para o que ele faz bem (ajustes específicos por ambiente).

### Limpeza

```bash
# Deletar o cluster Kind
kind delete cluster --name packaging-lab

# Remover os diretórios de demo
rm -rf kustomize-demo helm-kustomize-demo
```

---

## Principais Conclusões

1. **Gerenciar YAML bruto em escala não funciona.** Conforme seu cluster cresce, você precisa de templating (Helm), patching (Kustomize), ou ambos para manter os manifestos gerenciáveis entre ambientes.

2. **Helm é um gerenciador de pacotes completo.** Charts empacotam recursos, values os parametrizam, e o Helm rastreia releases com gerenciamento de ciclo de vida completo: install, upgrade, rollback e uninstall.

3. **Kustomize aplica patches em YAML puro sem templates.** O modelo base + overlays permite manter um único conjunto de manifestos válidos e customizá-los por ambiente usando patches, prefixos, substituição de imagens e injeção de labels.

4. **Helm e Kustomize são complementares, não concorrentes.** Use Helm para instalar charts de terceiros. Use Kustomize para customizar seus próprios manifestos. Use ambos juntos quando precisar do empacotamento Helm com o ajuste fino do Kustomize.

5. **`helm template` é sua ferramenta de debug.** Ele renderiza charts em YAML puro sem instalar, permitindo inspecionar exatamente o que seria aplicado ao cluster.

6. **Kustomize é integrado ao kubectl.** Não precisa de instalação separada — `kubectl apply -k` e `kubectl kustomize` funcionam nativamente.

7. **GitOps é o modelo de deploy para produção.** Faça push para o Git, deixe um controlador (ArgoCD, Flux) sincronizar o cluster. Charts Helm e overlays Kustomize são os blocos de construção que controladores GitOps consomem.

---

## Leitura Complementar

- [Helm — Using Helm](https://helm.sh/docs/intro/using_helm/)
- [Helm Install](https://helm.sh/docs/helm/helm_install/)
- [Helm — Getting Started](https://helm.sh/docs/chart_template_guide/getting_started/)
- [Kustomize — Managing Kubernetes Objects](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [kubectl kustomize](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_kustomize/)
- [Artifact Hub — Find Helm Charts](https://artifacthub.io/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/en/stable/)
- [Flux Documentation](https://fluxcd.io/flux/)

---

**Anterior:** [Capítulo 9 — Configuração e Secrets](09-configuration-and-secrets.md)
**Próximo:** [Capítulo 11 — Segurança](11-security.md)
