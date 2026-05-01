# Capítulo 5: Seu Primeiro Cluster

*"A melhor maneira de aprender um novo sistema é quebrá-lo de propósito."*

---

## Da Teoria ao Terminal

Você passou quatro capítulos construindo um modelo mental — o que são containers, como o Kubernetes é arquitetado, e como ele pensa em termos de desired state e reconciliação. Agora é hora de colocar a mão na massa.

Este capítulo é o seu capítulo "fique confortável". Se os capítulos anteriores foram estudar o mapa, este é a trilha. Vamos construir um cluster Kind multi-node, aprender a falar kubectl fluentemente, e explorar cada canto de um cluster em execução da mesma forma que um admin Linux explora um novo servidor — cutucando processos, verificando logs, olhando conexões de rede e testando o que acontece quando as coisas quebram.

Em um servidor Linux, a primeira coisa que você provavelmente faz depois de conectar via SSH é executar `w`, `top`, `df -h`, e talvez `systemctl list-units`. Vamos fazer o equivalente Kubernetes de tudo isso, e ao final deste capítulo, `kubectl` deve parecer tão natural quanto `systemctl`.

---

## Kind em Profundidade

No Capítulo 1, você criou um cluster Kind de node único. Isso era suficiente para um "hello world", mas clusters Kubernetes reais têm múltiplos nodes — um control plane e um ou mais workers. Kind permite simular isso no seu laptop.

### Configuração de Cluster Multi-Node

Kind usa um arquivo de configuração YAML para definir a topologia do seu cluster:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

Isso lhe dá um cluster de 3 nodes: um control plane e dois workers. Cada "node" é na verdade um container Docker rodando um node Kubernetes completo (kubelet, kube-proxy, container runtime).

Salve isso como `kind-config.yaml` e crie o cluster:

```bash
kind create cluster --name first-cluster --config kind-config.yaml
```

### Mapeamento de Portas

Se você quer acessar Services a partir da sua máquina host (digamos, para acessar um servidor nginx no cluster pelo seu navegador), você precisa mapear portas do container Docker para o seu host:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30080
    hostPort: 8080
    protocol: TCP
- role: worker
- role: worker
```

Isso mapeia a porta 30080 dentro do cluster (comumente usada com Services NodePort) para a porta 8080 na sua máquina local. Usaremos isso em capítulos posteriores.

### O Que o Kind Cria Por Baixo dos Panos

Após criar o cluster, espie o que o Docker está realmente executando:

```bash
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
```

Você verá três containers — estes são seus "nodes." Cada um executa um kubelet completo, kube-proxy e runtime containerd. O node do control plane adicionalmente executa etcd, kube-apiserver, kube-scheduler e kube-controller-manager.

---

## Kubeconfig Explicado

Quando você criou o cluster Kind, ele configurou o kubectl para se comunicar com ele. Mas *como*? A resposta é o arquivo kubeconfig, que fica em `~/.kube/config` por padrão.

### A Analogia com SSH Config

Se você já usou `~/.ssh/config`, kubeconfig vai parecer familiar:

| Conceito SSH Config | Equivalente Kubeconfig |
|--------------------|----------------------|
| Entrada Host | Entrada Cluster |
| Hostname + Porta | URL do Server |
| IdentityFile (chave privada) | Certificado de cliente / token |
| User | Entrada User |
| Um bloco Host completo | Context (cluster + user + namespace) |

### Estrutura do Kubeconfig

Um arquivo kubeconfig tem três seções principais:

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: https://127.0.0.1:PORT
    certificate-authority-data: <base64-encoded CA cert>
  name: kind-first-cluster
users:
- name: kind-first-cluster
  user:
    client-certificate-data: <base64-encoded client cert>
    client-key-data: <base64-encoded client key>
contexts:
- context:
    cluster: kind-first-cluster
    user: kind-first-cluster
  name: kind-first-cluster
current-context: kind-first-cluster
```

- **Clusters** — onde conectar (URL do server + certificado CA a confiar)
- **Users** — como autenticar (certificados, tokens ou plugins de auth)
- **Contexts** — um atalho que vincula cluster + user + namespace padrão opcional

### Trabalhando com Contexts

Se você gerencia múltiplos clusters (dev, staging, production — ou múltiplos clusters Kind para testes), contexts permitem alternar entre eles:

```bash
# Ver todos os contexts disponíveis
kubectl config get-contexts

# Ver qual context está ativo atualmente
kubectl config current-context

# Mudar para um context diferente
kubectl config use-context kind-first-cluster

# Visualizar o kubeconfig completo (ocultando dados sensíveis)
kubectl config view
```

Isso é exatamente como aliases SSH. Em vez de digitar `ssh -i ~/.ssh/prod-key -p 2222 admin@10.0.1.5`, você define um bloco Host e simplesmente digita `ssh prod`. Em vez de passar certificados e URLs para o kubectl, você define um context e simplesmente digita `kubectl --context prod get pods`.

---

## kubectl: Sua Nova CLI

`kubectl` é o canivete suíço do Kubernetes — seu `systemctl`, `journalctl`, `ssh`, `top` e `ss` tudo em um.

### O Padrão Verbo-Recurso

Quase todo comando kubectl segue este padrão:

```
kubectl <verbo> <tipo-recurso> [nome] [flags]
```

Os verbos mais comuns:

| Verbo | O Que Faz | Equivalente Linux |
|------|-------------|------------------|
| `get` | Listar recursos | `ls`, `systemctl list-units` |
| `describe` | Mostrar informações detalhadas sobre um recurso | `systemctl status`, `ip addr show` |
| `apply` | Criar ou atualizar a partir de um arquivo (declarativo) | `ansible-playbook site.yml` |
| `create` | Criar um recurso (imperativo) | `systemctl start`, `useradd` |
| `delete` | Remover um recurso | `systemctl stop`, `rm` |
| `exec` | Executar um comando dentro de um container | `ssh`, `nsenter` |
| `logs` | Visualizar saída do container | `journalctl -u service` |
| `port-forward` | Encaminhar uma porta local para um Pod/Service | Túnel `ssh -L` |
| `edit` | Abrir um recurso no seu editor | `systemctl edit` |
| `scale` | Alterar a contagem de réplicas | — (sem equivalente direto) |
| `top` | Mostrar uso de recursos | `top`, `htop` |

### Tipos de Recursos Comuns

| Recurso | Nome Curto | O Que É |
|----------|-----------|------------|
| `pods` | `po` | A menor unidade implantável (um ou mais containers) |
| `deployments` | `deploy` | Gerencia ReplicaSets e rolling updates |
| `services` | `svc` | Endpoint de rede estável para um conjunto de Pods |
| `configmaps` | `cm` | Dados de configuração como pares chave-valor |
| `secrets` | — | Dados sensíveis (codificados em base64, não criptografados por padrão) |
| `namespaces` | `ns` | Partições lógicas do cluster |
| `nodes` | `no` | As máquinas (físicas ou virtuais) no cluster |
| `replicasets` | `rs` | Garante um número especificado de réplicas de pod |
| `events` | `ev` | Eventos do cluster (scheduling, erros, avisos) |

Nomes curtos economizam digitação: `kubectl get po` é o mesmo que `kubectl get pods`.

### Formatos de Saída

kubectl pode gerar saída em múltiplos formatos — isso é incrivelmente poderoso para scripts e debugging:

```bash
# Visualização em tabela padrão
kubectl get pods

# Visualização ampla — mais colunas (nome do node, IP, etc.)
kubectl get pods -o wide

# YAML completo — o objeto completo como armazenado no etcd
kubectl get pod <name> -o yaml

# JSON completo
kubectl get pod <name> -o json

# JSONPath — extrair campos específicos
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Colunas personalizadas — construa sua própria tabela
kubectl get pods -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName,STATUS:.status.phase'

# Ordenar por um campo
kubectl get pods --sort-by=.status.startTime

# Obter apenas os nomes (útil para scripts)
kubectl get pods -o name
```

A saída `-o yaml` é particularmente valiosa. Quando você precisa criar um manifesto YAML para algo que construiu imperativamente, basta exportá-lo:

```bash
kubectl get deployment nginx -o yaml > nginx-deployment.yaml
```

---

## Imperativo vs. Declarativo

Existem duas formas de trabalhar com Kubernetes: imperativa e declarativa. Entender ambas — e saber quando usar cada uma — é essencial.

### Imperativo: Executar Comandos Diretamente

```bash
# Criar um deployment
kubectl create deployment nginx --image=nginx:1.27 --replicas=3

# Expô-lo como um service
kubectl expose deployment nginx --port=80 --type=NodePort

# Escalá-lo
kubectl scale deployment nginx --replicas=5

# Deletá-lo
kubectl delete deployment nginx
```

Isso parece natural para admins Linux — é como executar `systemctl start`, `useradd`, ou `iptables -A`. Você diz ao sistema o que fazer, passo a passo.

**Bom para:** Experimentos rápidos, debugging, tarefas únicas durante laboratórios.

### Declarativo: Aplicar Manifestos YAML

```bash
# Aplicar um manifesto (cria ou atualiza)
kubectl apply -f nginx-deployment.yaml

# Aplicar todos os manifestos em um diretório
kubectl apply -f ./manifests/

# Deletar tudo definido em um manifesto
kubectl delete -f nginx-deployment.yaml
```

Esta é a forma nativa do Kubernetes. Você define o desired state em um arquivo, armazena no controle de versão, e o `apply`. Se você mudar o arquivo e aplicar novamente, o Kubernetes descobre o diff e faz as mudanças mínimas necessárias.

**Bom para:** Produção, qualquer coisa que você queira reproduzir, colaboração em equipe, GitOps.

### Dry-Run e Diff

Antes de aplicar mudanças a um cluster ativo, você pode pré-visualizá-las:

```bash
# Dry-run: mostrar o que SERIA enviado ao servidor (sem realmente enviar)
kubectl apply -f nginx-deployment.yaml --dry-run=client -o yaml

# Dry-run do lado do servidor: validar contra o API server sem persistir
kubectl apply -f nginx-deployment.yaml --dry-run=server

# Diff: comparar um manifesto local com o que está rodando no cluster
kubectl diff -f nginx-deployment.yaml
```

A flag `--dry-run=client` também é ótima para *gerar* manifestos YAML:

```bash
# Gerar um YAML de Deployment sem criá-lo
kubectl create deployment nginx --image=nginx:1.27 --replicas=3 --dry-run=client -o yaml > nginx-deployment.yaml
```

Este é um fluxo de trabalho comum: criar imperativamente com dry-run para obter o template YAML, depois personalizá-lo e aplicar declarativamente.

---

## Tabela Comparativa Linux ↔ K8s

| Tarefa Linux | Equivalente kubectl | Notas |
|------------|-------------------|-------|
| `ssh user@server` | `kubectl exec -it <pod> -- /bin/sh` | Não é uma sessão persistente — Pod pode ser substituído a qualquer momento |
| `systemctl status nginx` | `kubectl get pods -l app=nginx` | Mostra o status de todos os Pods que correspondem ao label |
| `journalctl -u nginx -f` | `kubectl logs -f deployment/nginx` | Segue stdout/stderr do container |
| `scp file server:/path` | `kubectl cp file <pod>:/path` | Copia entre sistema de arquivos local e container |
| `ss -tlnp` | `kubectl get svc` | Lista Services e seus mapeamentos de porta |
| `cat /etc/systemd/system/nginx.service` | `kubectl get deployment nginx -o yaml` | Mostra a spec completa do desired-state |
| `top` | `kubectl top pods` | Requer Metrics Server (instalado no lab abaixo) |
| `systemctl restart nginx` | `kubectl rollout restart deployment/nginx` | Aciona um rolling restart de todos os Pods |
| `systemctl list-units` | `kubectl get all` | Lista tipos de recursos comuns em um namespace |
| `hostnamectl` | `kubectl get nodes -o wide` | Mostra nomes dos nodes, IPs, SO e versão do kernel |

---

> **Onde a Analogia com Linux Quebra**
>
> **`kubectl exec` não é SSH.** Parece SSH, mas o Pod pode ser substituído a qualquer momento por um controller. Nunca faça mudanças em produção via exec — mude o manifesto. Qualquer coisa que você faça dentro de um container via exec é *efêmera* e será perdida quando o Pod for recriado.
>
> **`kubectl logs` mostra stdout/stderr do container, não logs do sistema.** Não existe um equivalente ao `journalctl` para o cluster em si a partir do kubectl. Logs do kubelet, logs do API server e outros logs de nível de sistema ficam nos nodes e requerem acesso em nível de node para inspeção (ou uma solução de agregação de logs como Fluentd ou Loki).
>
> **Você nunca "loga em um node" para operações normais.** Diferente do SSH onde você conecta a um servidor específico, kubectl conecta ao API server, que faz proxy de tudo. O API server é o ponto de entrada único — você interage com o cluster, não com máquinas individuais. Isso é por design: nodes são gado, não animais de estimação.

---

## Laboratório Diagnóstico: Ficando Confortável com kubectl e Kind

Este é o capítulo prático. Leve seu tempo com este laboratório — o objetivo é que kubectl pareça natural ao final.

### Passo 1: Crie um Cluster Kind Multi-Node

Crie o arquivo de configuração do cluster:

```bash
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF
```

Crie o cluster:

```bash
kind create cluster --name first-cluster --config kind-config.yaml
```

---

### Passo 2: Explore o Cluster

Familiarize-se com o que você está trabalhando:

```bash
# Ver todos os nodes e seus detalhes
kubectl get nodes -o wide

# O que está rodando em todo o cluster?
kubectl get pods -A

# Informações do endpoint do cluster
kubectl cluster-info

# Quais tipos de recursos este cluster suporta?
kubectl api-resources | head -20

# Quantos tipos de recursos estão disponíveis?
kubectl api-resources --no-headers | wc -l
```

Demore um momento com `kubectl get pods -A`. Estes são os componentes do sistema que você aprendeu no Capítulo 3 — etcd, kube-apiserver, kube-scheduler, kube-controller-manager, CoreDNS, kube-proxy — todos rodando como Pods no namespace `kube-system`.

---

### Passo 3: Faça Deploy de uma Aplicação Imperativamente

Crie um Deployment e exponha-o:

```bash
# Criar um Deployment com 3 réplicas
kubectl create deployment nginx --image=nginx:1.27 --replicas=3

# Expô-lo como um Service
kubectl expose deployment nginx --port=80 --type=NodePort
```

Verifique que tudo foi criado:

```bash
kubectl get all
```

Você deve ver o Deployment, ReplicaSet, 3 Pods e um Service.

---

### Passo 4: Inspecione Tudo

Agora vamos explorar como um admin Linux faria:

```bash
# Visão detalhada do Deployment
kubectl describe deployment nginx

# Ver em qual node cada Pod está rodando
kubectl get pods -o wide

# Verificar a linha do tempo de eventos (seu log do sistema para o cluster)
kubectl get events --sort-by=.metadata.creationTimestamp

# Ver a spec completa do Deployment
kubectl get deployment nginx -o yaml

# Extrair apenas a imagem do container com JSONPath
kubectl get deployment nginx -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Olhe a saída do `kubectl get pods -o wide`. Note como os 3 Pods estão distribuídos entre os worker nodes? Esse é o scheduler fazendo seu trabalho.

---

### Passo 5: Depure Como um Admin Linux

**Exec em um Pod (seu substituto do SSH):**

```bash
# Obter um shell dentro de um container em execução
kubectl exec -it $(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}') -- /bin/sh
```

Uma vez dentro:

```sh
# Verificar a lista de processos (você está dentro de um container)
ps aux

# Verificar se o nginx está rodando
curl -s localhost

# Verificar o sistema de arquivos
ls /etc/nginx/

# Sair do shell
exit
```

**Visualizar logs do container (seu substituto do journalctl):**

```bash
# Logs de um Pod específico
kubectl logs $(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')

# Seguir logs em tempo real (como journalctl -f)
kubectl logs -f deployment/nginx

# Pressione Ctrl+C para parar de seguir
```

**Port-forward para acessar o Service (seu substituto do túnel SSH):**

```bash
# Encaminhar porta local 8080 para a porta 80 do Service
kubectl port-forward svc/nginx 8080:80 &

# Testar
curl -s localhost:8080

# Parar o port-forward
kill %1
```

---

### Passo 6: Mude para Declarativo

**Exporte o Deployment existente para YAML:**

```bash
kubectl get deployment nginx -o yaml > nginx-deployment.yaml
```

**Edite o arquivo — mude replicas de 3 para 5:**

```bash
sed -i 's/replicas: 3/replicas: 5/' nginx-deployment.yaml
```

> **Nota:** No macOS com BSD sed, use `sed -i '' 's/replicas: 3/replicas: 5/' nginx-deployment.yaml` em vez disso.

**Aplique a mudança declarativamente:**

```bash
kubectl apply -f nginx-deployment.yaml
```

**Verifique o escalonamento:**

```bash
kubectl get pods -l app=nginx
```

Você deve ver 5 Pods.

**Execute diff — não deve haver diferença já que acabamos de aplicar:**

```bash
kubectl diff -f nginx-deployment.yaml
```

Se não há saída (ou apenas diferenças de metadados como `resourceVersion`), o estado do cluster corresponde ao arquivo. Este é o fluxo declarativo: o arquivo é a verdade, e `apply` faz o cluster corresponder.

**Gere um manifesto limpo do zero (a abordagem recomendada):**

Em vez de exportar de um recurso em execução (que inclui muitos campos de runtime), use dry-run para gerar YAML limpo:

```bash
kubectl create deployment nginx-clean --image=nginx:1.27 --replicas=3 --dry-run=client -o yaml > nginx-clean.yaml
```

Veja a diferença:

```bash
wc -l nginx-deployment.yaml nginx-clean.yaml
```

A versão com dry-run é muito mais curta — ela só tem o que você realmente precisa.

Limpe o manifesto de teste:

```bash
rm -f nginx-clean.yaml
```

---

### Passo 7: Limpeza

```bash
# Deletar usando o manifesto (declarativo)
kubectl delete -f nginx-deployment.yaml

# Deletar o Service
kubectl delete svc nginx

# Limpar os arquivos locais
rm -f nginx-deployment.yaml kind-config.yaml

# Deletar o cluster Kind
kind delete cluster --name first-cluster
```

---

## Pontos-Chave

1. **Kind permite rodar clusters multi-node localmente.** Control plane + workers, mapeamento de portas, configurações personalizadas — tudo em containers Docker na sua máquina.

2. **Kubeconfig é o seu SSH config para clusters.** Ele define onde conectar (clusters), como autenticar (users) e atalhos para alternar entre eles (contexts).

3. **kubectl segue um padrão consistente de verbo-recurso.** Aprenda os verbos (`get`, `describe`, `apply`, `delete`, `exec`, `logs`) e os tipos de recurso (`pods`, `deployments`, `services`), e você pode navegar qualquer cluster.

4. **Formatos de saída são poderosos.** `-o wide` para debugging rápido, `-o yaml` para specs completas, `-o jsonpath` para scripts. Domine estes e você nunca se sentirá perdido.

5. **Imperativo para exploração, declarativo para produção.** Use `kubectl create` para experimentar rápido. Use `kubectl apply -f` com YAML versionado para qualquer coisa que importa.

6. **`--dry-run=client -o yaml` é seu gerador de manifestos.** Não escreva YAML do zero — deixe o kubectl gerar o template, depois personalize.

7. **`kubectl exec` e `kubectl logs` são seu SSH e journalctl.** São bons o suficiente para debugging, mas lembre-se: o Pod é efêmero. O manifesto é a fonte da verdade, não o container em execução.

---

## Leitura Adicional

- [kubectl Overview](https://kubernetes.io/docs/reference/kubectl/)
- [kubectl Quick Reference (Cheat Sheet)](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- [Organizing Cluster Access Using kubeconfig Files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
- [Kind — Configuration](https://kind.sigs.k8s.io/docs/user/configuration/)
- [Kind — Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [Managing Resources with kubectl](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/)
- [Declarative Management of Kubernetes Objects Using Configuration Files](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/)

---

**Anterior:** [Capítulo 4 — Como o Kubernetes Pensa](04-how-kubernetes-thinks.md)
**Próximo:** [Capítulo 6 — Pods e Workloads](06-pods-and-workloads.md)
