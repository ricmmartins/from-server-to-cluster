# Capítulo 9: Configuração e Secrets

*"Nunca codifique diretamente o que pode mudar. E tudo muda."*

---

## O /etc/ do Kubernetes

Todo administrador Linux tem uma relação profunda e pessoal com o `/etc/`. É onde a personalidade de um servidor vive — `nginx.conf`, `sshd_config`, `resolv.conf`, `my.cnf`, os arquivos `.env` que suas aplicações leem, os profiles de shell em `/etc/profile.d/`. Mude um arquivo no `/etc/`, recarregue o serviço, e o comportamento muda. O binário da aplicação permanece o mesmo; apenas a configuração difere entre ambientes.

Kubernetes pega essa ideia e a eleva a um padrão de primeira classe. Ao invés de fazer SSH em um servidor e editar arquivos, você declara configuração como objetos Kubernetes — ConfigMaps e Secrets — e os injeta em pods como variáveis de ambiente ou arquivos montados. A configuração vive no cluster, versionada e gerenciada junto com seus workloads. Você nunca faz login em um container para editar um arquivo de configuração. Você muda o ConfigMap, e o cluster propaga a atualização.

Esse é o princípio Twelve-Factor App em ação: "Armazene configuração no ambiente." Kubernetes não apenas suporta esse padrão — ele o impõe. A mesma imagem de container roda em dev, staging e produção. Apenas a configuração injetada muda.

Para um administrador Linux, pense da seguinte forma: ao invés de manter diferentes arquivos `/etc/nginx/nginx.conf` em diferentes servidores, você mantém diferentes ConfigMaps em diferentes namespaces (ou clusters). A imagem do nginx é idêntica em todo lugar. A configuração é externa.

---

## ConfigMaps — Configuração Gerenciada pelo Cluster

Um ConfigMap contém dados de configuração não confidenciais como pares chave-valor ou arquivos inteiros. É o equivalente Kubernetes de colocar arquivos de configuração no `/etc/` ou definir variáveis de ambiente no `/etc/profile.d/`.

### Criando ConfigMaps

**A partir de literais (pares chave-valor):**

```bash
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=DB_HOST=postgres \
  --from-literal=CACHE_TTL=300
```

**A partir de um arquivo:**

```bash
# Criar um arquivo de configuração primeiro
cat <<'EOF' > nginx.conf
server {
    listen 80;
    server_name localhost;
    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
    location /health {
        return 200 'OK';
        add_header Content-Type text/plain;
    }
}
EOF

kubectl create configmap nginx-config --from-file=nginx.conf
```

**A partir de um diretório (todos os arquivos viram chaves):**

```bash
mkdir config-dir
echo "info" > config-dir/log_level
echo "postgres" > config-dir/db_host
kubectl create configmap app-config-dir --from-file=config-dir/
```

**Declarativamente (YAML):**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: info
  DB_HOST: postgres
  CACHE_TTL: "300"
  app.properties: |
    database.host=postgres
    database.port=5432
    log.level=info
```

### Consumindo ConfigMaps

Existem duas formas principais de injetar dados de ConfigMap em um pod: como variáveis de ambiente ou como arquivos montados.

**Como variáveis de ambiente:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "env | sort && sleep 3600"]
    env:
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DB_HOST
```

Isso é como adicionar `export LOG_LEVEL=info` ao `/etc/profile.d/app.sh`.

**Injeção em massa com `envFrom`:**

```yaml
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "env | sort && sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-config
```

Isso injeta *toda* chave no ConfigMap como uma variável de ambiente. Como `source /etc/profile.d/app.sh` onde cada linha é um export.

**Como arquivos montados (volume):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-demo
spec:
  containers:
  - name: web
    image: nginx:1.27
    volumeMounts:
    - name: config-volume
      mountPath: /etc/nginx/conf.d
  volumes:
  - name: config-volume
    configMap:
      name: nginx-config
```

Cada chave no ConfigMap se torna um arquivo no caminho de montagem. Se o ConfigMap tem uma chave `nginx.conf`, o pod recebe um arquivo em `/etc/nginx/conf.d/nginx.conf`. Isso é exatamente como copiar arquivos de configuração para `/etc/` — exceto que o cluster os gerencia.

### A Ressalva da Propagação de Atualizações

Aqui está algo que pega todos que vêm do Linux:

- **ConfigMaps montados como volume auto-atualizam** — Quando você muda um ConfigMap, os arquivos nos volumes montados são eventualmente atualizados (o período de sincronização do kubelet é aproximadamente 60–90 segundos por padrão). Sua aplicação ainda precisa perceber a mudança (reler o arquivo, monitorar com inotify, etc.).
- **Variáveis de ambiente NUNCA auto-atualizam** — Se você injetou um valor de ConfigMap como variável de ambiente, mudar o ConfigMap tem zero efeito em pods rodando. Variáveis de ambiente são definidas no momento de início do container e são imutáveis durante a vida do container. Você deve reiniciar (ou recriar) o pod para pegar os novos valores.

No Linux, isso é como a diferença entre um app que lê `/etc/app.conf` a cada requisição (ele vê mudanças imediatamente) versus um app que lê `$LOG_LEVEL` uma vez na inicialização (ele nunca vê mudanças até ser reiniciado).

---

## Secrets — Configuração Sensível

Secrets são estruturalmente idênticos a ConfigMaps mas destinados a dados sensíveis: senhas, chaves de API, certificados TLS, chaves SSH. A API, padrões de consumo, e estrutura YAML são quase os mesmos.

### Criando Secrets

```bash
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=supersecret
```

Para certificados TLS:

```bash
kubectl create secret tls my-tls-secret \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem
```

Para credenciais de registro Docker:

```bash
kubectl create secret docker-registry my-registry \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass
```

### O Aviso Sobre Base64

Vamos ser absolutamente claros sobre isso: **codificação base64 NÃO é criptografia.** É codificação, como URL encoding. Qualquer pessoa com acesso ao objeto Secret pode decodificá-lo trivialmente:

```bash
# Criar um secret
kubectl create secret generic db-creds --from-literal=password=supersecret

# Ver o YAML bruto
kubectl get secret db-creds -o yaml
```

Você verá algo como:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
data:
  password: c3VwZXJzZWNyZXQ=
```

Aquele `c3VwZXJzZWNyZXQ=` é apenas base64:

```bash
echo 'c3VwZXJzZWNyZXQ=' | base64 --decode
# Output: supersecret
```

Qualquer pessoa que possa executar `kubectl get secret` pode ler suas senhas. Secrets fornecem uma *separação de responsabilidades* (config vs. config sensível), não *segurança*. Para segurança real, você precisa de:

- **Criptografia em repouso** — Criptografar o armazenamento do etcd para que Secrets não sejam armazenados em texto plano
- **RBAC** — Restringir quem pode ler objetos Secret
- **Gerenciadores de secrets externos** — Ferramentas como HashiCorp Vault, AWS Secrets Manager, ou Azure Key Vault, integradas via projetos como External Secrets Operator ou Sealed Secrets

### Consumindo Secrets

Secrets são consumidos exatamente como ConfigMaps — como variáveis de ambiente ou montagens de volume:

**Como variáveis de ambiente:**

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-creds
      key: password
```

**Como arquivos montados:**

```yaml
volumeMounts:
- name: secret-volume
  mountPath: /etc/db-creds
  readOnly: true
volumes:
- name: secret-volume
  secret:
    secretName: db-creds
```

Quando montados como arquivos, cada chave se torna um arquivo. A chave `password` se torna `/etc/db-creds/password`. Isso é como colocar credenciais em um arquivo com permissões restritas — exceto que no Kubernetes, o controle de acesso é no nível de RBAC, não no nível do sistema de arquivos.

### Tipos de Secret

| Tipo | Propósito |
|------|-----------|
| `Opaque` | Dados genéricos chave-valor (padrão) |
| `kubernetes.io/tls` | Certificados TLS (deve conter `tls.crt` e `tls.key`) |
| `kubernetes.io/dockerconfigjson` | Credenciais de registro Docker |
| `kubernetes.io/basic-auth` | Autenticação básica (username e password) |
| `kubernetes.io/ssh-auth` | Autenticação SSH (ssh-privatekey) |
| `kubernetes.io/service-account-token` | Tokens de service account (criados automaticamente) |

---

## Variáveis de Ambiente — Injeção Direta

Além de ConfigMaps e Secrets, você pode definir variáveis de ambiente diretamente na spec do pod:

```yaml
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: APP_MODE
      value: "production"
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: password
```

Você pode misturar valores diretos, referências a ConfigMaps, e referências a Secrets no mesmo bloco `env:`. Isso dá máxima flexibilidade — codifique diretamente o que é verdadeiramente estático, referencie o que é compartilhado, e proteja o que é sensível.

---

## A Downward API — Metadados do Pod como Configuração

Às vezes sua aplicação precisa saber sobre *si mesma* — seu nome de pod, namespace, endereço IP, ou limites de recursos. A Downward API expõe esses metadados como variáveis de ambiente ou arquivos, sem a aplicação precisar chamar a API do Kubernetes.

Pense nisso como `/proc/self/status` para pods. No Linux, um processo pode ler `/proc/self/status` para saber seu PID, uso de memória e capabilities. No Kubernetes, um pod pode usar a Downward API para saber seu nome, namespace, labels e alocações de recursos.

**Como variáveis de ambiente:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-demo
  labels:
    app: demo
    version: v2
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "env | grep -E 'POD_|NODE_' && sleep 3600"]
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: POD_MEMORY_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: limits.memory
```

Campos `fieldRef` disponíveis incluem:
- `metadata.name` — nome do pod
- `metadata.namespace` — namespace do pod
- `metadata.uid` — UID do pod
- `metadata.labels['<KEY>']` — valor de um label específico
- `metadata.annotations['<KEY>']` — valor de uma annotation específica
- `spec.nodeName` — o node onde o pod está rodando
- `spec.serviceAccountName` — a service account
- `status.podIP` — o endereço IP do pod

Isso é incrivelmente útil para logging (incluir o nome do pod em cada linha de log), métricas (marcar métricas com namespace e node), e descoberta de serviços (conhecer seu próprio IP).

---

## ConfigMaps e Secrets Imutáveis

A partir do Kubernetes v1.21 (estável), você pode marcar ConfigMaps e Secrets como imutáveis:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: static-config
immutable: true
data:
  VERSION: "2.5.0"
  FEATURE_FLAGS: "dark-mode,new-checkout"
```

Uma vez definido como `immutable: true`, os dados não podem ser alterados. Qualquer tentativa de atualizá-lo será rejeitada pelo API server. Você deve deletar e recriar o objeto para alterá-lo.

Por que você iria querer isso?

1. **Performance** — O kubelet para de observar ConfigMaps/Secrets imutáveis por mudanças, reduzindo a carga no API server. Em clusters com milhares de pods, isso importa.
2. **Segurança** — Previne modificações acidentais em configuração que não deveria mudar durante o tempo de vida de uma aplicação.

Pense nisso como montar um sistema de arquivos somente-leitura (`mount -o ro`). Você pode lê-lo, mas não pode acidentalmente (ou intencionalmente) modificá-lo enquanto está em uso.

---

## Comparação Linux ↔ Kubernetes

| Conceito Linux | Equivalente K8s | Notas |
|----------------|-----------------|-------|
| `/etc/nginx/nginx.conf` | ConfigMap montado como arquivo | Configuração gerenciada pelo cluster |
| `export ENV_VAR=value` | `env:` na spec do pod | Injeção de variável de ambiente |
| `/etc/shadow` (arquivo restrito) | Secret | Dados sensíveis (mas ainda precisa de criptografia!) |
| `/proc/self/status` | Downward API | Metadados do pod expostos ao container |
| `source /etc/profile.d/*.sh` | `envFrom: configMapRef` | Injeção em massa de ambiente |
| inotify (monitoramento de mudança de arquivo) | Atualizações de volume ConfigMap | Auto-atualização quando config muda (~60-90s) |
| `mount -o ro` (montagem somente-leitura) | `immutable: true` | Otimização de performance, previne mudanças |
| `chmod 600 /etc/shadow` | RBAC em objetos Secret | Controle de acesso (nível de namespace, não por chave) |

---

> ### 🔀 Onde a Analogia com Linux Quebra
>
> - **Atualizações de arquivo de config não disparam recarregamentos de serviço.** No Linux, você edita `/etc/nginx/nginx.conf` e executa `systemctl reload nginx`. No Kubernetes, você atualiza o ConfigMap, e... a montagem de volume eventualmente atualiza (60–90 segundos), mas a aplicação pode não perceber. Não há sinal `reload` embutido. Se você usou variáveis de ambiente ao invés de montagem de volume, os valores nunca atualizam — você precisa de um reinício do pod. Alguns projetos contornam isso com sidecar reloaders ou triggers de rollout baseados em hash, mas não é uma funcionalidade nativa.
>
> - **Secrets NÃO são seguros por padrão.** Eles são codificados em base64 (trivialmente reversível) e armazenados sem criptografia no etcd a menos que você explicitamente habilite criptografia em repouso. Qualquer pessoa com acesso a `kubectl get secret` pode ler suas senhas. Trate Secrets como "ligeiramente melhor que ConfigMaps" para separação de responsabilidades — segurança real requer gerenciadores de secrets externos e políticas RBAC adequadas.
>
> - **Não há controle de acesso por chave.** No Linux, você pode `chmod 600 /etc/shadow` e `chmod 644 /etc/passwd` — permissões no nível de arquivo. No Kubernetes, controle de acesso é no nível de namespace/RBAC. Se alguém pode ler Secrets em um namespace, pode ler *todos* os Secrets naquele namespace. Não há equivalente a definir permissões diferentes em chaves individuais dentro de um ConfigMap ou Secret.

---

## Laboratório Diagnóstico: Configuração Hands-On

### Pré-requisitos

```bash
kind create cluster --name config-lab
```

### Lab 1: Criar e Consumir um ConfigMap

**Passo 1 — Crie um ConfigMap a partir de literais:**

```bash
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=DB_HOST=postgres \
  --from-literal=CACHE_TTL=300
```

**Passo 2 — Crie um ConfigMap a partir de um arquivo:**

```bash
cat <<'EOF' > nginx.conf
server {
    listen 80;
    server_name localhost;
    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
    location /health {
        return 200 'healthy';
        add_header Content-Type text/plain;
    }
}
EOF

kubectl create configmap nginx-config --from-file=nginx.conf
```

**Passo 3 — Inspecione os ConfigMaps:**

```bash
kubectl get configmap app-config -o yaml
kubectl describe configmap nginx-config
```

**Passo 4 — Implante um pod consumindo ConfigMap como variáveis de ambiente:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: env-consumer
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo LOG_LEVEL=$LOG_LEVEL DB_HOST=$DB_HOST && sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-config
EOF

kubectl wait --for=condition=Ready pod/env-consumer --timeout=60s

# Verificar as variáveis de ambiente
kubectl exec env-consumer -- env | grep -E 'LOG_LEVEL|DB_HOST|CACHE_TTL'
```

**Passo 5 — Implante um pod consumindo ConfigMap como arquivo montado:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: volume-consumer
spec:
  containers:
  - name: web
    image: nginx:1.27
    volumeMounts:
    - name: nginx-conf
      mountPath: /etc/nginx/conf.d
  volumes:
  - name: nginx-conf
    configMap:
      name: nginx-config
EOF

kubectl wait --for=condition=Ready pod/volume-consumer --timeout=60s

# Verificar que o arquivo de config está montado
kubectl exec volume-consumer -- cat /etc/nginx/conf.d/nginx.conf

# Testar que o nginx usa nossa config personalizada (o endpoint /health)
kubectl exec volume-consumer -- curl -s http://localhost/health
```

### Lab 2: Criar e Consumir um Secret

**Passo 1 — Crie um Secret:**

```bash
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=supersecret
```

**Passo 2 — Prove que base64 não é criptografia:**

```bash
# Ver o secret bruto
kubectl get secret db-creds -o yaml

# Decodificar a senha (é por isso que base64 != criptografia)
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 --decode
echo  # newline para legibilidade
```

**Passo 3 — Implante um pod consumindo o Secret como montagem de volume:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-consumer
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo 'Username:' $(cat /etc/db-creds/username) && echo 'Password:' $(cat /etc/db-creds/password) && sleep 3600"]
    volumeMounts:
    - name: db-creds
      mountPath: /etc/db-creds
      readOnly: true
  volumes:
  - name: db-creds
    secret:
      secretName: db-creds
EOF

kubectl wait --for=condition=Ready pod/secret-consumer --timeout=60s

# Cada chave é um arquivo no caminho de montagem
kubectl exec secret-consumer -- ls /etc/db-creds/
kubectl exec secret-consumer -- cat /etc/db-creds/password
```

**Passo 4 — Consuma o Secret como variável de ambiente:**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-consumer
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo DB_PASSWORD=$DB_PASSWORD && sleep 3600"]
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: password
EOF

kubectl wait --for=condition=Ready pod/secret-env-consumer --timeout=60s
kubectl exec secret-env-consumer -- env | grep DB_PASSWORD
```

### Lab 3: Downward API

Implante um pod que expõe seus próprios metadados como variáveis de ambiente:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: downward-demo
  labels:
    app: demo
    version: v2
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "env | grep -E 'POD_|NODE_' && sleep 3600"]
    resources:
      requests:
        memory: "32Mi"
        cpu: "100m"
      limits:
        memory: "64Mi"
        cpu: "200m"
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: POD_MEMORY_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: limits.memory
EOF

kubectl wait --for=condition=Ready pod/downward-demo --timeout=60s

# O pod sabe sobre si mesmo sem chamar a API do Kubernetes
kubectl exec downward-demo -- env | grep -E 'POD_|NODE_'
```

Você deve ver saída como:

```
POD_NAME=downward-demo
POD_NAMESPACE=default
POD_IP=10.244.0.5
NODE_NAME=config-lab-control-plane
POD_MEMORY_LIMIT=67108864
```

O pod sabe seu próprio nome, namespace, IP, o node onde está rodando, e seu limite de memória — tudo sem nenhuma chamada à API do Kubernetes.

### Lab 4: Atualizar um ConfigMap e Ver a Propagação

**Passo 1 — Verifique o estado atual:**

```bash
# O pod volume-consumer do Lab 1 deve ainda estar rodando
kubectl exec volume-consumer -- cat /etc/nginx/conf.d/nginx.conf | head -3
```

**Passo 2 — Atualize o ConfigMap:**

```bash
cat <<'EOF' > nginx-updated.conf
server {
    listen 80;
    server_name localhost;
    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
    location /health {
        return 200 'healthy-v2';
        add_header Content-Type text/plain;
    }
    location /ready {
        return 200 'ready';
        add_header Content-Type text/plain;
    }
}
EOF

kubectl create configmap nginx-config --from-file=nginx.conf=nginx-updated.conf \
  --dry-run=client -o yaml | kubectl apply -f -
```

**Passo 3 — Aguarde a propagação e verifique:**

```bash
# Montagens de volume atualizam em aproximadamente 60-90 segundos
echo "Aguardando propagação do ConfigMap (até 90 segundos)..."
sleep 90

# Verificar se o arquivo foi atualizado (procure o bloco location /ready)
kubectl exec volume-consumer -- cat /etc/nginx/conf.d/nginx.conf
```

Você deve ver a configuração atualizada com o novo bloco location `/ready`.

**Importante:** O arquivo no disco foi atualizado, mas o nginx não sabe disso ainda. Você precisaria executar `nginx -s reload` dentro do container (ou reiniciar o pod) para que o nginx pegue a mudança. Kubernetes atualiza o arquivo; sua aplicação é responsável por perceber.

**Passo 4 — Mostre que variáveis de ambiente NÃO atualizam:**

```bash
# env-consumer ainda tem os valores originais
kubectl exec env-consumer -- env | grep LOG_LEVEL

# Mesmo se atualizarmos o ConfigMap...
kubectl patch configmap app-config -p '{"data":{"LOG_LEVEL":"debug"}}'

# ...a variável de ambiente no pod rodando não muda
sleep 5
kubectl exec env-consumer -- env | grep LOG_LEVEL
# Ainda mostra "info", não "debug"

# Apenas um reinício do pod pega o novo valor da variável de ambiente
kubectl delete pod env-consumer
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: env-consumer
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo LOG_LEVEL=$LOG_LEVEL && sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-config
EOF

kubectl wait --for=condition=Ready pod/env-consumer --timeout=60s
kubectl exec env-consumer -- env | grep LOG_LEVEL
# Agora mostra "debug"
```

### Limpeza

```bash
# Deletar o cluster Kind
kind delete cluster --name config-lab

# Remover arquivos temporários
rm -f nginx.conf nginx-updated.conf
```

---

## Principais Conclusões

1. **ConfigMaps externalizam configuração de aplicação.** Eles contêm dados chave-valor não sensíveis ou arquivos de configuração inteiros, injetados em pods como variáveis de ambiente ou volumes montados.

2. **Secrets são para dados sensíveis, mas base64 não é criptografia.** Qualquer pessoa que possa executar `kubectl get secret` pode decodificar suas senhas. Use RBAC, criptografia em repouso, e gerenciadores de secrets externos para segurança real.

3. **ConfigMaps montados como volume auto-atualizam; variáveis de ambiente nunca.** Se você precisa de atualizações de configuração em tempo real sem reinícios de pod, use montagens de volume e faça sua aplicação monitorar mudanças de arquivo.

4. **A Downward API permite que pods saibam sobre si mesmos.** Nome do pod, namespace, IP, node, labels e limites de recursos podem todos ser injetados como variáveis de ambiente ou arquivos — sem chamadas à API do Kubernetes necessárias.

5. **`envFrom` é injeção em massa; `env` é injeção seletiva.** Use `envFrom` para injetar todas as chaves de um ConfigMap ou Secret. Use `env` com `valueFrom` para pegar chaves específicas ou renomeá-las.

6. **ConfigMaps e Secrets imutáveis melhoram a performance.** Marque-os como `immutable: true` para parar o kubelet de monitorar mudanças — essencial em clusters com milhares de ConfigMaps montados.

7. **Separe configuração do código, sempre.** A mesma imagem de container deve rodar identicamente em dev, staging e produção. Apenas os ConfigMaps, Secrets e variáveis de ambiente injetados devem diferir.

---

## Leitura Complementar

- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Downward API](https://kubernetes.io/docs/concepts/workloads/pods/downward-api/)
- [Immutable Secrets and ConfigMaps](https://kubernetes.io/docs/concepts/configuration/secret/#secret-immutable)
- [Managing Secrets Using kubectl](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/)
- [Configure a Pod to Use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
- [Expose Pod Information to Containers Through Environment Variables](https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/)

---

**Anterior:** [Capítulo 8 — Armazenamento e Persistência](08-storage-and-persistence.md)
**Próximo:** [Capítulo 10 — Empacotamento e Entrega](10-packaging-and-delivery.md)
