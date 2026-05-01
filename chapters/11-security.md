# Capítulo 11: Segurança

*"O único sistema verdadeiramente seguro é aquele que está desligado, envolto em um bloco de concreto, e selado em uma sala forrada de chumbo com guardas armados — e mesmo assim eu tenho minhas dúvidas."*
— Gene Spafford

---

## De chmod a RBAC: Mesmo Objetivo, Escala Diferente

Em um servidor Linux, a segurança é física e íntima. Você sabe quem tem acesso root porque gerencia o `/etc/passwd` e `/etc/shadow`. Você controla o que processos podem fazer com permissões de arquivo (`chmod 640`), o que usuários podem executar com `sudoers`, e qual tráfego de rede é permitido com `iptables`. Toda decisão de segurança se resume às mesmas três perguntas: **quem** está solicitando acesso, **o que** estão tentando fazer, e **em qual recurso**?

O Kubernetes faz exatamente as mesmas três perguntas — mas em escala de cluster.

Em vez de usuários Linux, você tem **ServiceAccounts** — identidades que pods usam para se autenticar no API server. Em vez de `/etc/sudoers`, você tem **Roles** e **ClusterRoles** — manifestos YAML que definem quais verbos de API (get, list, create, delete) são permitidos em quais recursos (pods, services, secrets). Em vez de adicionar um usuário ao grupo `sudo`, você cria um **RoleBinding** que vincula uma Role a uma ServiceAccount.

O modelo mental é o mesmo: *quem pode fazer o que em quais recursos.* O mecanismo é diferente porque o Kubernetes é um sistema distribuído orientado a API, não uma única máquina com um sistema de arquivos. Toda ação passa pelo API server, e toda requisição é autenticada, autorizada e admitida (ou rejeitada) antes de qualquer coisa acontecer.

Para um administrador Linux, a boa notícia é que seus instintos de segurança — menor privilégio, defesa em profundidade, negação explícita — todos se aplicam. Você só precisa aprender o vocabulário Kubernetes.

---

## RBAC: Controle de Acesso Baseado em Funções

RBAC é a espinha dorsal de autorização do Kubernetes. Foi promovido a GA no Kubernetes 1.8 e está habilitado por padrão em todo cluster moderno. Se você entende usuários, grupos e sudoers do Linux, já compreende a base conceitual.

### ServiceAccounts — Identidade para Pods

Todo pod no Kubernetes roda com uma ServiceAccount. Se você não especificar uma, ele usa a ServiceAccount `default` no namespace. Pense em ServiceAccounts como usuários de sistema no Linux — são identidades para processos, não pessoas.

```bash
# Todo namespace recebe uma ServiceAccount 'default' automaticamente
kubectl get serviceaccounts
```

O ponto crítico: a ServiceAccount `default` frequentemente tem mais permissões do que você esperaria, especialmente em clusters mais antigos. A primeira regra de segurança do Kubernetes é **não usar a ServiceAccount default para workloads de produção.** Crie ServiceAccounts dedicadas com apenas as permissões que cada workload precisa.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: default
```

### Roles e ClusterRoles — Definições de Permissão

Uma **Role** define quais ações são permitidas em quais recursos, com escopo para um único namespace. Uma **ClusterRole** faz o mesmo mas em todo o cluster.

```yaml
# Uma Role que permite acesso somente leitura a pods em um namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]            # "" = grupo de API core
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

Pense nisso como uma regra sudoers: "Esta identidade pode executar `get`, `list` e `watch` em pods no namespace default." Sem create, sem delete, sem update.

Uma **ClusterRole** funciona da mesma forma mas sem escopo de namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
```

### RoleBindings e ClusterRoleBindings — Vinculando Permissões

Roles definem permissões. Bindings vinculam essas permissões a identidades. Um **RoleBinding** concede uma Role dentro de um namespace específico. Um **ClusterRoleBinding** concede uma ClusterRole em todos os namespaces.

```yaml
# Vincular a Role pod-reader à ServiceAccount my-app
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: my-app
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

A lista `subjects` é *quem* recebe a permissão. O `roleRef` é *qual* conjunto de permissões. Você pode vincular a mesma Role a múltiplas ServiceAccounts, ou vincular múltiplas Roles à mesma ServiceAccount via RoleBindings separados.

### Roles Integradas

O Kubernetes vem com várias ClusterRoles padrão que cobrem casos de uso comuns:

| ClusterRole | Permissões | Equivalente Linux |
|-------------|------------|------------------|
| `cluster-admin` | Tudo, em todos os lugares | `root` |
| `admin` | Acesso total dentro de um namespace (incluindo RBAC) | `sudo` no nível de namespace |
| `edit` | Leitura/escrita na maioria dos recursos, mas sem RBAC | Usuário avançado |
| `view` | Acesso somente leitura na maioria dos recursos | Usuário de auditoria / somente leitura |

**O princípio do menor privilégio se aplica.** Não distribua `cluster-admin` quando `view` é suficiente. Toda ServiceAccount com permissões excessivas é uma potencial expansão do raio de explosão.

---

## Pod Security Admission (PSA)

Se RBAC controla *quem pode criar workloads*, Pod Security Admission controla *que tipo de workloads podem rodar*. É o equivalente Kubernetes dos perfis AppArmor ou SELinux — restrições sobre o que um processo (pod) pode fazer em tempo de execução.

### Um Breve Histórico

O Kubernetes anteriormente usava PodSecurityPolicies (PSPs) para este propósito. PSPs foram descontinuadas no Kubernetes 1.21 e **removidas completamente no Kubernetes 1.25**. A substituição é o controlador Pod Security Admission integrado, que é mais simples, com escopo de namespace, e usa perfis de segurança predefinidos.

### Três Níveis de Segurança

Pod Security Admission define três perfis progressivamente restritivos, chamados **Pod Security Standards**:

| Nível | O Que Permite | Caso de Uso |
|-------|---------------|----------|
| **Privileged** | Vale tudo — sem restrições | Infraestrutura de nível de sistema (CNI, agentes de logging) |
| **Baseline** | Bloqueia escalações de privilégio conhecidas (hostNetwork, hostPID, containers privilegiados) | A maioria dos workloads de aplicação |
| **Restricted** | Exige non-root, remove todas as capabilities, proíbe escalação de privilégios, perfis seccomp | Workloads sensíveis à segurança |

### Três Modos de Aplicação

Para cada namespace, você pode configurar Pod Security Admission em três modos:

| Modo | Comportamento |
|------|----------|
| **enforce** | Rejeita pods que violam a política |
| **audit** | Permite o pod mas registra uma violação no log de auditoria |
| **warn** | Permite o pod mas exibe um aviso para o usuário |

Você aplica via labels no namespace:

```bash
# Aplicar o perfil restricted em um namespace
kubectl label namespace secure-ns \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest

# Avisar sobre violações do baseline (mas não bloquear)
kubectl label namespace dev-ns \
  pod-security.kubernetes.io/warn=baseline \
  pod-security.kubernetes.io/warn-version=latest
```

O padrão recomendado para migrar workloads existentes: comece com `warn` para ver o que quebraria, depois `audit` para rastrear violações sem interrupção, e então `enforce` quando estiver confiante.

---

## NetworkPolicy — Regras de Firewall para Pods

Por padrão, a rede do Kubernetes é **totalmente aberta**. Todo pod pode se comunicar com qualquer outro pod no cluster, independentemente do namespace. Não há firewall, sem segmentação, sem isolamento. É como rodar um servidor Linux com `iptables -P INPUT ACCEPT` e nenhuma regra.

Objetos **NetworkPolicy** são o firewall do Kubernetes. Eles definem quais pods podem se comunicar com quais outros pods, em quais portas, em qual direção.

### Como NetworkPolicies Funcionam

1. **Sem NetworkPolicy = sem restrições.** Pods aceitam tráfego de qualquer lugar.
2. **Uma vez que você aplica uma NetworkPolicy selecionando um pod, esse pod se torna isolado.** Apenas tráfego explicitamente permitido por uma política é aceito.
3. **Políticas são aditivas (modelo de whitelist).** Você não pode escrever uma regra "negar esta conexão específica". Você escreve "permitir estas conexões" e todo o resto é implicitamente negado.

### Anatomia de uma NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend            # Esta política se aplica a pods com app=backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend       # Permitir tráfego DE pods com app=frontend
    ports:
    - protocol: TCP
      port: 80                # Apenas na porta 80
```

Isso se lê: "Pods com label `app: backend` aceitam tráfego TCP de entrada na porta 80, mas apenas de pods com label `app: frontend`. Todo outro ingress para pods `backend` é negado."

### Políticas de Negação Padrão

O padrão de NetworkPolicy mais importante é a **negação padrão** — bloquear todo o tráfego, depois liberar seletivamente:

```yaml
# Negar padrão TODA entrada no namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}             # Selector vazio = aplica a TODOS os pods
  policyTypes:
  - Ingress                   # Bloquear todo tráfego de entrada
```

```yaml
# Negar padrão TODA saída no namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress                    # Bloquear todo tráfego de saída
```

> **Importante:** NetworkPolicies requerem um plugin CNI que as suporte. Nem todos os CNIs suportam. O **kindnet** (CNI padrão do Kind) tem suporte limitado a NetworkPolicy. Para produção ou testes completos, use **Calico** ou **Cilium**.

---

## Proteção de Secrets

O Capítulo 9 introduziu Secrets como um recurso Kubernetes. Mas armazenar um Secret no cluster é apenas metade da batalha — você também precisa protegê-lo.

### O Problema com Secrets Padrão

Por padrão, Kubernetes Secrets são:

1. **Codificados em Base64, não criptografados.** Qualquer pessoa que possa ler o Secret pode decodificá-lo instantaneamente com `base64 -d`.
2. **Armazenados sem criptografia no etcd.** Se alguém ganhar acesso ao diretório de dados do etcd, pode ler todos os Secrets do cluster.
3. **Acessíveis a qualquer pessoa com permissões RBAC de `get secret`.** Um RoleBinding mal configurado pode expor todas as senhas e chaves de API em um namespace.

### Criptografia em Repouso

O Kubernetes suporta criptografia de dados de Secret no etcd via um recurso `EncryptionConfiguration`. Isso garante que mesmo se alguém acessar o armazenamento etcd diretamente, os valores dos Secrets estarão criptografados:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}       # Fallback: ler dados não criptografados escritos antes da habilitação da criptografia
```

Isso é configurado no API server via a flag `--encryption-provider-config`. Em serviços Kubernetes gerenciados (AKS, EKS, GKE), a criptografia em repouso é tipicamente habilitada por padrão.

### Gerenciamento Externo de Secrets

Para ambientes de produção, a melhor prática é armazenar secrets *fora* do cluster inteiramente:

- **External Secrets Operator (ESO):** Sincroniza secrets de armazenamentos externos (AWS Secrets Manager, Azure Key Vault, HashiCorp Vault, GCP Secret Manager) em Kubernetes Secrets automaticamente.
- **Sealed Secrets:** Criptografa Secrets no lado do cliente para que a versão criptografada possa ser armazenada com segurança no Git. Apenas o cluster pode descriptografá-los.

A regra de ouro: **nunca armazene secrets em texto puro no Git.** Nem em manifestos, nem em arquivos de values do Helm, nem em definições de variáveis de ambiente. Use gerenciamento externo de secrets ou soluções com criptografia em repouso.

---

## Security Context — Permissões por Pod e por Container

Um SecurityContext define configurações de privilégio e controle de acesso para um pod ou container individual. É aqui que o Kubernetes mapeia mais diretamente para primitivos de segurança do Linux — porque ele literalmente configura os recursos de segurança Linux do container runtime.

### Campos Principais

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:                    # Configurações no nível do pod
    runAsNonRoot: true                # Recusar iniciar se a imagem rodar como root
    runAsUser: 1000                   # Rodar como UID 1000
    runAsGroup: 3000                  # Rodar como GID 3000
    fsGroup: 2000                     # Grupo suplementar para montagem de volumes
    seccompProfile:
      type: RuntimeDefault            # Aplicar o filtro seccomp padrão do container runtime
  containers:
  - name: app
    image: nginx:1.27
    securityContext:                   # Configurações no nível do container (sobrescrevem o nível do pod)
      allowPrivilegeEscalation: false # Bloquear bits setuid/setgid
      readOnlyRootFilesystem: true    # Montar sistema de arquivos raiz como somente leitura
      capabilities:
        drop:
        - ALL                         # Remover todas as capabilities Linux
        add:
        - NET_BIND_SERVICE            # Adicionar de volta apenas o necessário
```

### Linux Capabilities no Kubernetes

Linux capabilities são um mapeamento direto. O campo `capabilities` em um SecurityContext é o exato mesmo mecanismo que `setcap` e `capsh` no Linux:

| Capability | O Que Permite |
|-----------|---------------|
| `NET_BIND_SERVICE` | Vincular a portas abaixo de 1024 |
| `SYS_PTRACE` | Rastrear/depurar outros processos |
| `SYS_ADMIN` | Um pacote de operações privilegiadas (mount, operações de namespace) |
| `NET_RAW` | Usar raw sockets (necessário para `ping`) |
| `CHOWN` | Alterar proprietário de arquivos |

A melhor prática: **remover TODAS as capabilities, depois adicionar de volta apenas o que o container realmente precisa.** Este é o equivalente de capabilities do menor privilégio.

### Perfis Seccomp

Seccomp (Secure Computing Mode) é um recurso do kernel Linux que restringe quais chamadas de sistema um processo pode fazer. O Kubernetes expõe isso via o campo `seccompProfile`:

- `RuntimeDefault` — Usa o perfil seccomp padrão do container runtime (baseline recomendado)
- `Localhost` — Referencia um perfil seccomp personalizado no node
- `Unconfined` — Sem filtragem seccomp (não recomendado)

---

## Comparação Linux ↔ Kubernetes

| Conceito Linux | Equivalente Kubernetes | Notas |
|---------------|----------------------|-------|
| Usuários e grupos | ServiceAccounts | Identidade para processos/pods |
| `/etc/sudoers` | Role / ClusterRole | Quais ações são permitidas |
| Pertencer ao grupo `sudo` | RoleBinding / ClusterRoleBinding | Conceder permissões a uma identidade |
| `chmod`, `chown` | SecurityContext | Permissões por pod/container |
| Chains `iptables` INPUT/OUTPUT | NetworkPolicy ingress/egress | Filtragem de tráfego |
| Perfis AppArmor/SELinux | Pod Security Admission | Restringir o que pods podem fazer |
| Capabilities (`capsh`, `setcap`) | securityContext.capabilities | Controle de privilégios granular |
| Permissões do `/etc/shadow` | Secrets + RBAC | Restringir acesso a dados sensíveis |

---

> ### 🔒 Onde a Analogia com Linux Quebra
>
> - **Segurança Linux tem escopo de máquina. RBAC Kubernetes tem escopo de cluster.** No Linux, um usuário mal configurado afeta uma máquina. No Kubernetes, um ClusterRoleBinding mal configurado pode conceder permissões em *todos* os namespaces do cluster. Um único binding `cluster-admin` dado à ServiceAccount errada é equivalente a dar `root` em todos os servidores simultaneamente. O raio de explosão é o cluster inteiro.
>
> - **Não existe `su` ou `sudo` no Kubernetes.** Você não pode elevar privilégios temporariamente. Sua ServiceAccount ou tem permissão ou não tem — não existe "digite a senha para executar este comando como admin." O Kubernetes suporta impersonação (flag `--as=`), mas usá-la requer grants RBAC explícitos para os verbos de impersonação. É uma escolha de design deliberada: sem autoridade implícita, sem caminhos de escalação de privilégio.
>
> - **NetworkPolicies são aditivas (modelo de whitelist).** No Linux, o padrão do `iptables` frequentemente é "aceitar tudo, bloquear tráfego específico" — você adiciona regras DROP para coisas que quer bloquear. No Kubernetes, uma vez que você aplica uma política de negação padrão, **nada** pode se comunicar até que você explicitamente permita. Você constrói a partir do zero, não reduz a partir de tudo. Isso pega as pessoas de surpresa: aplique uma negação padrão, e de repente todo seu namespace fica silencioso.

---

## Laboratório Diagnóstico: Segurança Kubernetes na Prática

### Pré-requisitos

Crie um cluster Kind com Calico para suporte completo a NetworkPolicy:

```bash
cat <<'EOF' > kind-security-lab.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: "192.168.0.0/16"
nodes:
- role: control-plane
- role: worker
EOF

kind create cluster --name security-lab --config kind-security-lab.yaml
```

Instale o CNI Calico:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.3/manifests/calico.yaml
```

Aguarde os pods do Calico ficarem prontos:

```bash
kubectl wait --for=condition=Ready pods -l k8s-app=calico-node -n kube-system --timeout=120s
kubectl get nodes
```

Todos os nodes devem mostrar status `Ready`.

### Lab 1: RBAC — ServiceAccounts, Roles e RoleBindings

**Passo 1 — Criar uma ServiceAccount:**

```bash
kubectl create serviceaccount dev-user
```

**Passo 2 — Criar uma Role que permite apenas leitura de pods:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
EOF
```

**Passo 3 — Criar um RoleBinding para vincular a Role à ServiceAccount:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-user-pod-reader
  namespace: default
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

**Passo 4 — Testar as permissões:**

```bash
# Deve retornar "yes" — dev-user pode listar pods
kubectl auth can-i list pods --as=system:serviceaccount:default:dev-user

# Deve retornar "yes" — dev-user pode obter pods
kubectl auth can-i get pods --as=system:serviceaccount:default:dev-user

# Deve retornar "no" — dev-user não pode deletar pods
kubectl auth can-i delete pods --as=system:serviceaccount:default:dev-user

# Deve retornar "no" — dev-user não pode criar deployments
kubectl auth can-i create deployments --as=system:serviceaccount:default:dev-user

# Deve retornar "no" — dev-user não pode obter secrets
kubectl auth can-i get secrets --as=system:serviceaccount:default:dev-user
```

Isso é menor privilégio em ação: `dev-user` pode ler pods e nada mais.

**Passo 5 — Explorar o que a ServiceAccount pode fazer de forma abrangente:**

```bash
kubectl auth can-i --list --as=system:serviceaccount:default:dev-user
```

Isso exibe uma lista completa de permissões — útil para auditoria.

### Lab 2: Pod Security Admission

**Passo 1 — Criar um namespace com o perfil PSA restricted:**

```bash
kubectl create namespace secure-ns

kubectl label namespace secure-ns \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest
```

**Passo 2 — Tentar fazer deploy de um pod privilegiado (deve ser REJEITADO):**

```bash
cat <<'EOF' | kubectl apply -n secure-ns -f -
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    securityContext:
      privileged: true
EOF
```

Você deve ver um erro como:
```
Error from server (Forbidden): error when creating "STDIN": pods "privileged-pod"
is forbidden: violates PodSecurity "restricted:latest"...
```

**Passo 3 — Fazer deploy de um pod em conformidade (deve TER SUCESSO):**

```bash
cat <<'EOF' | kubectl apply -n secure-ns -f -
apiVersion: v1
kind: Pod
metadata:
  name: compliant-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: busybox:1.37
    command: ["sleep", "3600"]
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
EOF
```

**Passo 4 — Verificar que o pod em conformidade está rodando:**

```bash
kubectl get pods -n secure-ns
kubectl exec -n secure-ns compliant-pod -- id
kubectl exec -n secure-ns compliant-pod -- whoami
```

O pod roda como UID 1000 — um usuário não-root.

### Lab 3: NetworkPolicy

**Passo 1 — Fazer deploy de duas aplicações (frontend e backend):**

```bash
# Deploy do backend
kubectl run backend --image=nginx:1.27 --labels="app=backend" --port=80
kubectl expose pod backend --port=80 --target-port=80

# Deploy do frontend (um pod de longa duração para testar conectividade)
kubectl run frontend --image=busybox:1.37 --labels="app=frontend" -- sleep 3600

# Aguardar os pods ficarem prontos
kubectl wait --for=condition=Ready pod/backend --timeout=60s
kubectl wait --for=condition=Ready pod/frontend --timeout=60s
```

**Passo 2 — Verificar que podem se comunicar (sem políticas ainda):**

```bash
kubectl exec frontend -- wget -qO- --timeout=5 http://backend
```

Você deve ver o HTML da página de boas-vindas do nginx. A rede está totalmente aberta.

**Passo 3 — Aplicar uma política de negação padrão para todo ingress:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF
```

**Passo 4 — Verificar que o frontend NÃO PODE MAIS alcançar o backend:**

```bash
kubectl exec frontend -- wget -qO- --timeout=5 http://backend
```

Isso deve dar timeout — a política de negação padrão bloqueia todo tráfego de entrada.

**Passo 5 — Adicionar uma política que permite frontend → backend na porta 80:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
EOF
```

**Passo 6 — Verificar que a comunicação foi restaurada:**

```bash
kubectl exec frontend -- wget -qO- --timeout=5 http://backend
```

Você deve ver a página de boas-vindas do nginx novamente. A política de whitelist permite seletivamente frontend → backend enquanto todo o resto permanece bloqueado.

**Passo 7 — Verificar que outros pods ainda não conseguem alcançar o backend:**

```bash
# Criar um pod sem a label frontend
kubectl run outsider --image=busybox:1.37 --labels="app=outsider" -- sleep 3600
kubectl wait --for=condition=Ready pod/outsider --timeout=60s

# Tentar alcançar o backend — deve dar timeout
kubectl exec outsider -- wget -qO- --timeout=5 http://backend
```

O pod outsider é bloqueado porque não corresponde ao selector de label `app: frontend`.

### Lab 4: Security Context

**Passo 1 — Fazer deploy de um pod com configurações de segurança estritas:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: busybox:1.37
    command: ["sleep", "3600"]
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
EOF
```

**Passo 2 — Verificar que as restrições de segurança estão ativas:**

```bash
# Verificar UID e GID — deve ser uid=1000, gid=3000
kubectl exec hardened-pod -- id

# Tentar escrever no sistema de arquivos raiz — deve falhar (somente leitura)
kubectl exec hardened-pod -- touch /test

# Tentar escrever em /tmp — também falha (todo o FS raiz é somente leitura)
kubectl exec hardened-pod -- touch /tmp/test
```

Os comandos `touch` devem falhar com "Read-only file system." O container só pode escrever em volumes montados explicitamente como graváveis (emptyDir, PVC).

**Passo 3 — Verificar que o pod tem as configurações de QoS mais estritas refletidas:**

```bash
kubectl get pod hardened-pod -o jsonpath='{.spec.securityContext}' | python -m json.tool
```

### Limpeza

```bash
# Deletar todos os recursos do lab
kubectl delete pod frontend backend outsider hardened-pod --ignore-not-found
kubectl delete svc backend --ignore-not-found
kubectl delete networkpolicy default-deny-ingress allow-frontend-to-backend --ignore-not-found
kubectl delete rolebinding dev-user-pod-reader --ignore-not-found
kubectl delete role pod-reader --ignore-not-found
kubectl delete serviceaccount dev-user --ignore-not-found
kubectl delete namespace secure-ns --ignore-not-found

# Deletar o cluster Kind
kind delete cluster --name security-lab

# Remover o arquivo de configuração do cluster
rm -f kind-security-lab.yaml
```

---

## Principais Conclusões

1. **RBAC é a base da segurança Kubernetes.** Todo pod roda com uma ServiceAccount, toda ação requer autorização, e o princípio do menor privilégio deve guiar cada Role e RoleBinding que você criar. Nunca use `cluster-admin` quando uma Role com escopo de namespace é suficiente.

2. **Pod Security Admission substitui as descontinuadas PodSecurityPolicies.** Use os três níveis predefinidos (Privileged, Baseline, Restricted) com labels no namespace. Comece com o modo `warn`, avance para `enforce` quando verificar a conformidade.

3. **A rede padrão do Kubernetes é totalmente aberta.** Sem NetworkPolicies, todo pod pode se comunicar com qualquer outro pod. Aplique políticas de negação padrão primeiro, depois libere seletivamente os caminhos de comunicação necessários — assim como configurar um firewall.

4. **NetworkPolicies requerem um CNI compatível.** O `kindnet` padrão do Kind tem suporte limitado. Para produção ou testes completos, use Calico ou Cilium.

5. **Security Context mapeia diretamente para primitivos de segurança Linux.** `runAsNonRoot`, `capabilities`, `readOnlyRootFilesystem` e `seccompProfile` estão todos configurando recursos do kernel Linux via objetos da API Kubernetes. Seu conhecimento de segurança Linux se aplica diretamente aqui.

6. **Secrets não são seguros por padrão.** São codificados em base64 (não criptografados), armazenados sem criptografia no etcd a menos que você habilite criptografia em repouso, e acessíveis a qualquer pessoa com as permissões RBAC certas. Trate Secrets como ponto de partida, não como linha de chegada — use gerenciamento externo de secrets para produção.

7. **Defesa em profundidade é a única estratégia.** RBAC controla quem pode fazer o quê. PSA controla quais workloads podem rodar. NetworkPolicy controla o que pode se comunicar com o quê. SecurityContext controla o que um container pode fazer no nível do SO. Use todos juntos — cada camada captura o que as outras deixam escapar.

---

## Leitura Complementar

- [RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Encrypting Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- [Declare Network Policy](https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/)

---

**Anterior:** [Capítulo 10 — Empacotamento e Entrega](10-packaging-and-delivery.md)
**Próximo:** [Capítulo 12 — Escalabilidade e Observabilidade](12-scaling-and-observability.md)
