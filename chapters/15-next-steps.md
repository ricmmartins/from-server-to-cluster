# Capítulo 15: Próximos Passos

*"O especialista em qualquer coisa já foi um iniciante. A diferença é que ele nunca parou."*

---

## De "Aprendi Kubernetes" para "Trabalho Com Kubernetes"

Parabéns. Se você leu até aqui — e melhor ainda, fez os laboratórios — você viajou do gerenciamento de servidores Linux individuais para entender como a orquestração de containers funciona em um nível fundamental. Você não apenas aprendeu comandos; construiu um modelo mental. Você entende *por que* o Kubernetes funciona da forma que funciona, porque viu as fundações Linux por baixo.

Aqui está o detalhe sobre o Linux: você aprendeu uma vez e o conhecimento central serviu por uma década. Processos, filesystems, rede, permissões — os fundamentos são estáveis. Kubernetes é diferente. Os conceitos centrais (pods, services, controllers, estado desejado) são estáveis, mas o ecossistema ao redor deles evolui rapidamente. Ferramentas são substituídas. APIs são depreciadas. Melhores práticas mudam conforme a comunidade aprende. Aprendizado contínuo não é uma sugestão — é como você se mantém relevante.

Este capítulo final mapeia para onde ir a partir daqui: certificações que validam suas habilidades, caminhos de carreira que as aproveitam, comunidades que apoiam seu crescimento e ambientes de prática que mantêm suas mãos afiadas.

---

## Certificações Kubernetes

A Cloud Native Computing Foundation (CNCF) oferece quatro certificações Kubernetes, cada uma direcionada a um nível e foco diferente:

### Comparação de Certificações

| Certificação | Nível | Formato | Duração | Custo (USD) | Pré-requisitos |
|--------------|-------|--------|----------|------------|---------------|
| **KCNA** | Entrada | Múltipla escolha | 90 min | $250 | Nenhum |
| **CKA** | Intermediário | Lab hands-on | 2 horas | $445 | Conhecimento básico de K8s |
| **CKAD** | Intermediário | Lab hands-on | 2 horas | $445 | Alguma experiência com K8s |
| **CKS** | Avançado | Lab hands-on | 2 horas | $445 | Certificação CKA ativa |

Todos os preços incluem uma retentativa gratuita. Certificações são válidas por 2 anos.

### KCNA — Kubernetes and Cloud Native Associate

**O que é:** Um exame de entrada, múltipla escolha, que testa seu entendimento dos conceitos de Kubernetes e do ecossistema cloud-native.

**Para quem é:** Alguém que completou este livro e quer validar seu entendimento conceitual antes de mergulhar em certificações hands-on.

**O que cobre:**
- Fundamentos do Kubernetes (arquitetura, API, workloads)
- Conceitos de orquestração de containers
- Arquitetura cloud-native (microsserviços, auto-scaling)
- Observabilidade cloud-native (logging, monitoramento, tracing)
- Entrega de aplicações cloud-native (GitOps, CI/CD)

**Recursos de estudo:**
- Este livro cobre 80%+ dos tópicos do KCNA
- Currículo oficial: https://github.com/cncf/curriculum
- Página de certificação CNCF: https://www.cncf.io/certification/kcna/

### CKA — Certified Kubernetes Administrator

**O que é:** Um exame baseado em performance onde você resolve tarefas reais em um cluster Kubernetes ao vivo. Você tem um terminal, acesso kubectl e documentação — e precisa ser *rápido*.

**Para quem é:** Engenheiros de plataforma, SREs e profissionais de infraestrutura que gerenciam clusters Kubernetes.

**O que cobre:**
- Arquitetura, instalação e configuração do cluster (25%)
- Workloads e scheduling (15%)
- Services e networking (20%)
- Storage (10%)
- Troubleshooting (30%)

**Habilidades chave testadas:**
- Deploy e gerenciamento de clusters com kubeadm
- Configuração de networking (CNI, Services, Ingress)
- Gerenciamento de storage (PV, PVC, StorageClass)
- Troubleshooting de problemas de cluster e aplicação
- Configuração de RBAC e segurança
- Backup e restore do etcd

**Recursos de estudo:**
- Currículo oficial: https://github.com/cncf/curriculum
- Página de certificação CNCF: https://www.cncf.io/certification/cka/
- Simulador de exame: https://killer.sh (incluído no registro do exame)

### CKAD — Certified Kubernetes Application Developer

**O que é:** Baseado em performance, como o CKA, mas focado em deploy e gerenciamento de aplicações em vez de clusters.

**Para quem é:** Desenvolvedores que fazem deploy no Kubernetes, engenheiros DevOps construindo pipelines CI/CD.

**O que cobre:**
- Design e build de aplicações (20%)
- Deploy de aplicações (20%)
- Observabilidade e manutenção de aplicações (15%)
- Ambiente, configuração e segurança de aplicações (25%)
- Services e networking (20%)

**Habilidades chave testadas:**
- Projetar pods multi-container
- Usar Deployments, Jobs, CronJobs efetivamente
- Configurar probes, limites de recursos, ConfigMaps, Secrets
- Implementar NetworkPolicies
- Usar Helm para deploy de aplicações

**Recursos de estudo:**
- Currículo oficial: https://github.com/cncf/curriculum
- Página de certificação CNCF: https://www.cncf.io/certification/ckad/

### CKS — Certified Kubernetes Security Specialist

**O que é:** O exame avançado de segurança. Requer uma certificação CKA ativa como pré-requisito.

**Para quem é:** Engenheiros de segurança, times de plataforma responsáveis por hardening e compliance do cluster.

**O que cobre:**
- Configuração do cluster (10%)
- Hardening do cluster (15%)
- Hardening do sistema (15%)
- Minimizar vulnerabilidades de microsserviços (20%)
- Segurança da cadeia de suprimentos (20%)
- Monitoramento, logging e segurança em runtime (20%)

**Recursos de estudo:**
- Currículo oficial: https://github.com/cncf/curriculum
- Página de certificação CNCF: https://www.cncf.io/certification/cks/

### Dicas para o Exame

1. **Velocidade importa.** Você tem 2 horas para ~15-20 tarefas. Pratique com cronômetro. Se uma questão está demorando demais, marque e siga em frente.

2. **Domine atalhos do kubectl:**
   ```bash
   alias k=kubectl
   export do="--dry-run=client -o yaml"
   # Exemplo: k run nginx --image=nginx $do > pod.yaml
   ```

3. **Use `kubectl explain` durante o exame:**
   ```bash
   kubectl explain pod.spec.containers.livenessProbe
   kubectl explain deployment.spec.strategy
   ```

4. **Salve a documentação do Kubernetes nos favoritos.** Você tem acesso a https://kubernetes.io/docs durante o exame. Saiba onde as coisas estão antes do dia do exame.

5. **Pratique com killer.sh.** Seu registro de exame inclui duas sessões no simulador de exame. Use-as 1-2 semanas antes da data do exame.

6. **Aprenda comandos imperativos.** São mais rápidos que escrever YAML do zero:
   ```bash
   kubectl create deployment nginx --image=nginx --replicas=3
   kubectl expose deployment nginx --port=80 --type=ClusterIP
   kubectl create configmap my-config --from-literal=key=value
   kubectl create secret generic my-secret --from-literal=password=s3cr3t
   ```

---

## Caminhos de Carreira

Habilidades em Kubernetes abrem múltiplas trajetórias de carreira. Sua experiência com Linux dá uma vantagem em todas elas:

### Papel Linux → Evolução Kubernetes

| Papel Linux | Evolução K8s | Habilidades Chave Adicionadas |
|-----------|--------------|-----------------|
| Sysadmin | Engenheiro de Plataforma | K8s, IaC (Terraform), GitOps, observabilidade |
| Admin de Rede | Engenheiro de Rede Cloud | CNI, service mesh, Gateway API, load balancing |
| Admin de Segurança | Engenheiro de Segurança K8s | PSA, RBAC, OPA/Kyverno, scan de imagens, compliance |
| Arquiteto de Sistemas | Arquiteto Cloud | Multi-cloud, otimização de custos, padrões HA |
| DBA | Engenheiro de Confiabilidade de Banco de Dados | Operators, StatefulSets, automação de backup |
| Engenheiro de Release | DevOps/SRE | CI/CD, entrega progressiva, SLOs, resposta a incidentes |

### Engenheiro de Plataforma

Você constrói e mantém a plataforma Kubernetes que outros times usam. Você é responsável por provisionamento de clusters, upgrades, infraestrutura de observabilidade, experiência do desenvolvedor e confiabilidade da plataforma.

**Dia a dia:** Gerenciamento de clusters, Terraform/Pulumi para infraestrutura, construção de plataformas internas de desenvolvedor, configuração de pipelines CI/CD, gerenciamento de stacks de observabilidade (Prometheus, Grafana, Loki).

### Site Reliability Engineer (SRE)

Você garante que serviços sejam confiáveis, rápidos e disponíveis. Você define SLOs, constrói monitoramento, responde a incidentes e conduz melhorias de engenharia para reduzir toil.

**Dia a dia:** Definição de SLIs/SLOs, construção de dashboards e alertas, rotação de plantão, revisões pós-incidente, planejamento de capacidade, automação de tarefas operacionais repetitivas.

### Engenheiro DevOps

Você faz a ponte entre desenvolvimento e operações. Constrói pipelines CI/CD, automatiza testes e deploy, gerencia infraestrutura como código e permite que desenvolvedores entreguem mais rápido e com mais segurança.

**Dia a dia:** Desenvolvimento de pipelines (GitHub Actions, GitLab CI), workflows GitOps (ArgoCD, Flux), gerenciamento de imagens de container, provisionamento de ambientes, estratégias de deploy.

### Arquiteto Cloud

Você projeta soluções cloud-native no nível de sistema. Toma decisões sobre estratégia multi-cloud, adoção de service mesh, otimização de custos, recuperação de desastres e seleção de tecnologia.

**Dia a dia:** Revisões de arquitetura, provas de conceito, escrita de ADRs (Architecture Decision Records), mentoria de times, avaliação de novas tecnologias, modelagem de custos.

### Engenheiro de Segurança (Cloud-Native)

Você protege toda a plataforma: hardening de cluster, segurança de workloads, proteção da cadeia de suprimentos, enforcement de políticas e automação de compliance.

**Dia a dia:** Escrita de políticas (OPA/Kyverno), gerenciamento de RBAC, scan de vulnerabilidades de imagens, design de network policies, relatórios de compliance, modelagem de ameaças.

---

## O Ecossistema Cloud-Native (CNCF)

A Cloud Native Computing Foundation hospeda centenas de projetos. Não tente aprender todos — foque nas áreas relevantes para seu caminho de carreira.

### Principais Projetos Graduados

Estes são projetos prontos para produção e testados em batalha:

| Projeto | Categoria | O Que Faz |
|---------|----------|-------------|
| **Kubernetes** | Orquestração | Orquestração de containers (você já conhece este) |
| **Prometheus** | Monitoramento | Coleta de métricas e alertas |
| **Envoy** | Networking | Proxy de alta performance (usado por service meshes) |
| **Helm** | Empacotamento | Gerenciador de pacotes Kubernetes |
| **containerd** | Runtime | Container runtime (usado pelo K8s) |
| **CoreDNS** | Networking | Servidor DNS (embutido no K8s) |
| **etcd** | Storage | Armazenamento distribuído chave-valor (control plane K8s) |
| **Fluentd** | Logging | Camada de logging unificada |
| **Argo** | Entrega | GitOps e automação de workflows |
| **Cilium** | Networking | Networking e segurança baseada em eBPF |
| **OPA** | Política | Motor de política de propósito geral |

### Áreas para Explorar em Seguida

Com base no que você aprendeu neste livro, aqui estão os próximos passos naturais:

| Área | Projetos | Quando Aprender |
|------|----------|---------------|
| **Service Mesh** | Istio, Linkerd | Quando você precisar de mTLS, gerenciamento de tráfego, observabilidade entre serviços |
| **GitOps** | ArgoCD, Flux | Quando quiser deploys declarativos, orientados por Git |
| **Política** | OPA/Gatekeeper, Kyverno | Quando precisar enforçar padrões entre clusters |
| **Serverless** | Knative, KEDA | Quando quiser scale-to-zero ou workloads orientados por eventos |
| **Entrega Progressiva** | Argo Rollouts, Flagger | Quando precisar de deploys canary/blue-green |
| **Gerenciamento de Secrets** | External Secrets, Sealed Secrets | Quando precisar de tratamento de secrets em nível de produção |
| **Gerenciamento de Custos** | OpenCost, Kubecost | Quando precisar entender os gastos do cluster |

---

## Ambientes de Prática

Prática hands-on é inegociável. Aqui está onde conseguir:

### Ambientes Gratuitos

| Plataforma | O Que Oferece | Melhor Para |
|----------|---------------|----------|
| **Killercoda** (https://killercoda.com) | Labs interativos K8s gratuitos no navegador | Prática diária, reforço de conceitos |
| **Play with Kubernetes** (https://labs.play-with-k8s.com/) | Clusters multi-nó temporários | Experimentos rápidos, compartilhamento de demos |
| **Kind** (local) | Clusters K8s completos em Docker | Trabalho de lab, preparação CKA, testes CI |
| **k3s** (local) | K8s leve de binário único | Cenários edge, máquinas com poucos recursos |

### Preparação para Exame

| Plataforma | O Que Oferece | Melhor Para |
|----------|---------------|----------|
| **killer.sh** (https://killer.sh) | Simulador realista de exame CKA/CKAD/CKS | Preparação final (2 sessões gratuitas com registro) |
| **Kodekloud** | Cursos em vídeo + labs hands-on | Caminhos de aprendizado estruturados |
| **KodeKloud Engineer** | Cenários de tarefas do mundo real | Prática de resolução de problemas sob pressão de tempo |

### Tiers Gratuitos de Cloud

| Provedor | Oferta Gratuita |
|----------|---------------|
| **Azure (AKS)** | Cluster tier gratuito (sem cobranças de control plane) |
| **Google Cloud (GKE)** | Tier gratuito Autopilot + $300 de crédito para nova conta |
| **AWS (EKS)** | Sem tier gratuito para EKS (mas tier gratuito EC2 para autogerenciado) |

---

## Comunidade

Kubernetes tem uma das maiores e mais acolhedoras comunidades open-source:

### Onde Se Conectar

- **Kubernetes Slack** (https://slack.k8s.io/) — Milhares de canais. Comece com `#kubernetes-novice` e `#kubectl`.
- **CNCF Community Groups** — Meetups locais no mundo todo, tanto virtuais quanto presenciais.
- **KubeCon + CloudNativeCon** — A conferência principal (América do Norte, Europa, Ásia). Palestras são gratuitas no YouTube.
- **Reddit** — r/kubernetes para discussões e perguntas.
- **Stack Overflow** — Tag `kubernetes` para perguntas e respostas.

### Contribuindo

Kubernetes é construído por contribuidores de todo o mundo. Se você quer se envolver:

1. **Special Interest Groups (SIGs)** — Cada área do Kubernetes tem um SIG (SIG-Network, SIG-Storage, SIG-Auth, etc.). Participe de suas reuniões e canais Slack.
2. **Good first issues** — Procure labels `good-first-issue` no repositório kubernetes/kubernetes.
3. **Documentação** — A documentação sempre precisa de melhorias. É um ótimo ponto de entrada.
4. **Operators e ferramentas** — Construa e disponibilize como open-source seu próprio operator ou plugin kubectl.

---

## O Caminho de Aprendizado Revisitado

Este livro é parte de um ecossistema maior:

```
Linux Fundamentals         →  This Book              →  Hands-on Practice    →  Advanced Topics
(linuxhackathon.com)          (From Server            (k8shackathon.com)        (ai4infra.com)
                               to Cluster)
```

### Sequência Recomendada de Certificação

```
KCNA (conceitos) → CKA (admin) → CKAD (developer) → CKS (segurança)
     ↓                ↓               ↓                  ↓
  Valida           Valida          Valida             Valida
  conhecimento     operações       deploy de          habilidades
  do livro         de cluster      aplicações         de segurança
```

Você não precisa das quatro. Escolha baseado no seu caminho de carreira:
- **Engenheiro de Plataforma / SRE:** CKA → CKS
- **Desenvolvedor de Aplicações / DevOps:** CKAD → CKA
- **Mudança de carreira / Começando:** KCNA → CKA

---

> ### ⚠️ Onde a Analogia com Linux Quebra
>
> **Velocidade de mudança:** O ecossistema Linux é maduro e estável. Habilidades de 10 anos atrás ainda se aplicam — `grep`, `awk`, `systemd` e TCP/IP não mudaram fundamentalmente. O ecossistema Kubernetes se move rápido. APIs são depreciadas (PodSecurityPolicy → Pod Security Admission), ferramentas são substituídas (Docker shim → containerd) e melhores práticas evoluem. Aprendizado contínuo não é opcional — é sobrevivência.
>
> **Estilo de exame:** Certificações Linux (LFCS, RHCSA, RHCE) testam conhecimento acumulado com limites de tempo razoáveis. Certificações Kubernetes (CKA, CKAD, CKS) testam velocidade sob pressão em um ambiente ao vivo. Você precisa ser rápido com kubectl, não apenas correto. Uma resposta correta entregue em 15 minutos dá zero pontos se o orçamento de tempo é 7 minutos. Pratique exercícios cronometrados.
>
> **Amplitude vs. profundidade:** No Linux, você pode se especializar profundamente em uma área — tornar-se um mago de networking que conhece iptables por dentro, ou um especialista de storage que vive em LVM e ZFS. No Kubernetes, engenheiros de plataforma precisam de amplitude. Espera-se que você entenda networking, segurança, storage, observabilidade e ciclo de vida de aplicações em um nível funcional. Profundidade vem depois, uma vez que você tenha a amplitude.

---

## Laboratório Diagnóstico: Preparando-se para o Próximo Passo

### Lab 1: Configure Seu Ambiente de Prática

**Passo 1 — Criar um cluster multi-nó para prática estilo exame:**

```bash
kind create cluster --name exam-prep --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF
```

**Passo 2 — Configurar autocompletion do kubectl (essencial para velocidade no exame):**

```bash
# Para bash
source <(kubectl completion bash)
echo 'source <(kubectl completion bash)' >> ~/.bashrc

# Criar o alias essencial
alias k=kubectl
complete -o default -F __start_kubectl k
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
```

**Passo 3 — Configurar atalhos estilo exame:**

```bash
# Atalho de dry-run para gerar YAML rapidamente
export do="--dry-run=client -o yaml"

# Exemplos de criação rápida de recursos:
k run nginx --image=nginx:1.27 $do > pod.yaml
k create deployment web --image=nginx:1.27 --replicas=3 $do > deploy.yaml
k create service clusterip web --tcp=80:80 $do > svc.yaml

# Verificar o que foi gerado
cat pod.yaml
```

**Passo 4 — Praticar `kubectl explain` (disponível durante o exame):**

```bash
# Campos de nível superior do recurso
kubectl explain pod.spec

# Campos aninhados
kubectl explain pod.spec.containers.livenessProbe
kubectl explain deployment.spec.strategy.rollingUpdate

# Listar campos disponíveis em qualquer nível
kubectl explain pod.spec --recursive | head -40
```

---

### Lab 2: Autoavaliação Contra Domínios do CKA

Percorra este checklist. Para cada tópico, avalie-se honestamente:

```bash
cat <<'EOF'
=== CKA SELF-ASSESSMENT ===

Rate yourself 1-5 on each topic:
  1 = Never heard of it
  2 = Heard of it, can't do it
  3 = Can do it with documentation
  4 = Can do it from memory
  5 = Can do it fast under pressure

CLUSTER ARCHITECTURE & INSTALLATION (25%)
[ ] Manage RBAC (Role, ClusterRole, RoleBinding)
[ ] Use kubeadm to install a cluster
[ ] Manage a highly-available cluster
[ ] Provision infrastructure (certificate management)
[ ] Perform a version upgrade with kubeadm
[ ] Implement etcd backup and restore

WORKLOADS & SCHEDULING (15%)
[ ] Understand Deployments and rolling updates
[ ] Use ConfigMaps and Secrets
[ ] Configure resource limits and requests
[ ] Understand manifest management (Kustomize)
[ ] Configure pod scheduling (nodeSelector, affinity, taints/tolerations)

SERVICES & NETWORKING (20%)
[ ] Understand host networking on cluster nodes
[ ] Understand ClusterIP, NodePort, LoadBalancer
[ ] Configure and use CoreDNS
[ ] Configure Ingress and Ingress Controllers
[ ] Choose a CNI plugin

STORAGE (10%)
[ ] Understand PV, PVC, StorageClass
[ ] Configure volume modes, access modes, reclaim policies
[ ] Understand PV hostPath and CSI

TROUBLESHOOTING (30%)
[ ] Evaluate cluster and node logging
[ ] Monitor applications (probes, metrics)
[ ] Troubleshoot application failures
[ ] Troubleshoot cluster component failures
[ ] Troubleshoot networking
EOF
```

Tópicos onde você se avaliou 1-2 são suas prioridades de estudo. Tópicos em 3 precisam de prática cronometrada. Tópicos em 4-5 são seus pontos fortes.

---

### Lab 3: Explore o Landscape CNCF

**Passo 1 — Verificar quais projetos CNCF você já usou:**

```bash
cat <<'EOF'
=== CNCF PROJECTS FROM THIS BOOK ===

You've already worked with these CNCF projects:
✓ Kubernetes - Container orchestration (entire book)
✓ containerd - Container runtime (Chapter 2)
✓ etcd - Distributed key-value store (Chapter 3, 14)
✓ CoreDNS - Cluster DNS (Chapter 7)
✓ Helm - Package management (Chapter 10)
✓ Prometheus - Metrics (Chapter 12, mentioned)

Projects to explore next based on your interests:
→ ArgoCD/Flux - If you liked GitOps concepts (Chapter 10)
→ Cilium - If you found networking fascinating (Chapter 7)
→ cert-manager - If security resonated with you (Chapter 11)
→ OPA/Kyverno - If policy enforcement interests you
→ Velero - If disaster recovery is your focus (Chapter 14)
EOF
```

**Passo 2 — Visite o landscape CNCF:**

O landscape completo está em https://landscape.cncf.io/ — contém 1000+ projetos. Não se sinta sobrecarregado. Foque em:

1. **Projetos graduados** — Estes são prontos para produção e amplamente adotados
2. **Seu caminho de carreira** — Escolha 2-3 projetos alinhados com seu papel-alvo
3. **Suas necessidades imediatas** — O que ajudaria mais seu time atual?

**Passo 3 — Explorar CRDs existentes no seu cluster:**

```bash
# Todo cluster Kubernetes tem CRDs de componentes instalados
kubectl get crd

# Em um cluster Kind, você verá CRDs do local-path-provisioner
# Em um cluster de produção, você veria CRDs dos seus operators instalados
```

---

### Limpeza do Laboratório

```bash
kind delete cluster --name exam-prep
```

---

## Principais Conclusões

1. **Comece com KCNA, mire no CKA.** KCNA valida conhecimento conceitual (este livro). CKA valida habilidades hands-on de administração. Juntos provam que você entende Kubernetes tanto teórica quanto praticamente.

2. **O exame CKA testa velocidade, não apenas conhecimento.** Você precisa resolver 15-20 tarefas em 2 horas. Pratique com atalhos kubectl, comandos imperativos e `kubectl explain`. Memória muscular importa.

3. **Engenharia de Plataforma é a evolução natural do sysadmin Linux.** Ela adiciona Kubernetes, Infrastructure-as-Code, GitOps e observabilidade ao seu conjunto de habilidades existente. A fundação Linux que você tem é exatamente o ponto de partida certo.

4. **O ecossistema CNCF é vasto — seja seletivo.** Não tente aprender cada projeto. Escolha 2-3 que se alinhem com seu papel e vá fundo. Amplitude vem naturalmente ao longo do tempo conforme você encontra ferramentas em produção.

5. **Comunidade acelera o aprendizado.** Participe do Kubernetes Slack, assista palestras da KubeCon (gratuitas no YouTube), participe de reuniões de SIGs. A comunidade é acolhedora e o networking é inestimável.

6. **Ambientes de prática são gratuitos e abundantes.** Kind para prática local, Killercoda para labs no navegador, killer.sh para simulação de exame. Não há desculpa para não ter um cluster rodando.

7. **Você já está mais avançado do que pensa.** Se você completou este livro e fez os labs, você entende Kubernetes melhor do que a maioria das pessoas que "usam Kubernetes." Agora é sobre construir velocidade, profundidade e experiência do mundo real.

---

## Leitura Adicional

- [CKA Certification](https://www.cncf.io/certification/cka/)
- [CKAD Certification](https://www.cncf.io/certification/ckad/)
- [CKS Certification](https://www.cncf.io/certification/cks/)
- [KCNA Certification](https://www.cncf.io/certification/kcna/)
- [Exam Curriculum Repository](https://github.com/cncf/curriculum)
- [CNCF Landscape](https://landscape.cncf.io/)
- [Kubernetes Slack](https://slack.k8s.io/)
- [Killercoda Interactive Labs](https://killercoda.com)
- [killer.sh Exam Simulator](https://killer.sh)

---

**Anterior:** [Capítulo 14 — Operações em Produção](14-production-operations.md)
**Próximo:** [Apêndice A — Glossário Linux-para-Kubernetes](appendix-a-glossary.md)
