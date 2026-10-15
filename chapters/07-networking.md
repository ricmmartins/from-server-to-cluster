# Capítulo 7: Rede

*"Em um servidor único, rede é configuração. Em um cluster, rede é arquitetura."*

---

## De Uma Máquina para uma Rede Plana

Como administrador Linux, você conhece redes de cabo a rabo. Já configurou endereços IP com `ip addr`, definiu rotas com `ip route`, escreveu regras de iptables manualmente, debugou DNS com `dig` e `nslookup`, e provavelmente configurou mais de alguns reverse proxies nginx. Todo esse conhecimento se aplica diretamente aqui — a rede do Kubernetes é construída sobre os mesmos primitivos do Linux.

Mas há uma diferença fundamental: no Linux, você gerencia a rede de **uma máquina**. No Kubernetes, você está lidando com uma **rede abrangente do cluster** onde cada Pod recebe seu próprio endereço IP e pode se comunicar com qualquer outro Pod diretamente, independentemente de em qual node ele está. Sem NAT, sem mapeamento de portas, sem tabelas de roteamento manuais.

Se isso parece bom demais para ser verdade — é quase isso. Alguém precisa configurar todas essas rotas e regras, e esse alguém é o plugin Container Network Interface (CNI) e o kube-proxy. Este capítulo revela como a rede do Kubernetes realmente funciona, e como ela se mapeia para a rede Linux que você já conhece.

---

## O Modelo de Rede do Kubernetes

Antes de entrarmos em Services e Ingress, você precisa entender as regras fundamentais. O Kubernetes exige um modelo de rede com estes requisitos:

1. **Cada Pod recebe seu próprio endereço IP único** — não compartilhado com outros Pods, sem NAT.
2. **Todos os Pods podem se comunicar com todos os outros Pods** em qualquer node sem NAT.
3. **Agentes em um node** (kubelet, daemons do sistema) podem se comunicar com todos os Pods naquele node.

Isso é chamado de **modelo de rede plana (flat network model)**, e é fundamentalmente diferente da rede bridge padrão do Docker (onde containers em hosts diferentes não conseguem se comunicar sem publicação explícita de portas).

### Como Funciona Por Baixo dos Panos

O projeto Kubernetes não implementa a rede em si — ele define os requisitos e delega para **plugins CNI (Container Network Interface)**. Plugins CNI populares incluem:

- **Kindnet** — O padrão para clusters Kind. Simples, leve.
- **Calico** — Rico em funcionalidades, suporta NetworkPolicy, roteamento BGP.
- **Cilium** — Baseado em eBPF, alta performance, observabilidade avançada.
- **Flannel** — Rede overlay simples usando VXLAN.

Cada plugin CNI adota uma abordagem diferente (túneis VXLAN, roteamento BGP, eBPF), mas o resultado é o mesmo: cada Pod recebe um IP, e cada Pod pode alcançar qualquer outro Pod.

### A Analogia com Linux

Pense da seguinte forma: se você tivesse 10 servidores Linux e quisesse que cada processo em cada servidor conversasse com cada processo em cada outro servidor por IP — sem NAT, sem conflitos de porta — você precisaria configurar roteamento entre todas as máquinas, atribuir faixas de IP não sobrepostas, e manter tudo isso. Isso é exatamente o que um plugin CNI faz, automaticamente.

---

## Services: O Endpoint Estável

Pods vêm e vão. Deployments escalam para cima e para baixo. IPs de Pods mudam toda vez que um Pod é recriado. Então como você se conecta a algo de forma confiável?

**Services** fornecem um endpoint de rede estável — um endereço IP fixo e um nome DNS — que roteia tráfego para um conjunto de Pods selecionados por labels. Pense em um Service como um load balancer baseado em DNS que atualiza automaticamente sua lista de backends conforme Pods aparecem e desaparecem.

### ClusterIP (Padrão)

Um Service **ClusterIP** recebe um endereço IP virtual que só é alcançável de dentro do cluster. É o tipo de Service padrão.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

**Analogia Linux:** Como um VIP gerenciado pelo keepalived, mas ao invés de failover entre duas máquinas, ele faz balanceamento de carga entre todos os Pods correspondentes.

Quando outro Pod no cluster chama `nginx-svc` (ou `nginx-svc.default.svc.cluster.local`), a requisição atinge o ClusterIP, que a distribui para um dos Pods backend.

### NodePort

Um Service **NodePort** expõe o serviço em uma porta estática no endereço IP de cada node. Clientes externos podem acessar o serviço acessando `<ip-de-qualquer-node>:<node-port>`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
```

**Analogia Linux:** Como executar `iptables -t nat -A PREROUTING -p tcp --dport 30080 -j DNAT --to-destination <ip-backend>:80` em cada servidor da sua frota. Qualquer máquina pode receber a requisição e encaminhá-la para o backend correto.

A faixa de NodePort é 30000-32767 por padrão. Isso raramente é usado em produção (LoadBalancer ou Ingress são melhores), mas é prático para desenvolvimento e clusters locais.

### LoadBalancer

Um Service **LoadBalancer** solicita um load balancer externo do provedor de nuvem (ou uma solução local como MetalLB ou cloud-provider-kind). É a forma padrão de expor serviços para a internet.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

**Analogia Linux:** Como ter um HAProxy ou nginx reverse proxy na frente dos seus servidores, automaticamente configurado com a lista correta de backends. Você não gerencia o proxy — o provedor de nuvem faz isso.

Em clusters Kind, você pode usar [cloud-provider-kind](https://github.com/kubernetes-sigs/cloud-provider-kind) para habilitar Services LoadBalancer localmente.

### Headless Services

Às vezes você não quer balanceamento de carga — você quer descobrir todos os IPs individuais dos Pods diretamente. Um **Headless Service** (`clusterIP: None`) faz exatamente isso.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None
  selector:
    app: db
  ports:
  - port: 5432
```

Ao invés de retornar um único ClusterIP, uma consulta DNS para `db-headless` retorna os endereços IP de todos os Pods correspondentes. Isso é essencial para StatefulSets onde clientes precisam se conectar a Pods específicos (ex: `db-0.db-headless`).

**Analogia Linux:** Como entradas no `/etc/hosts` apontando para cada servidor individual, ao invés de um VIP de load balancer.

---

## kube-proxy e Regras de Pacotes do Linux: Como os Services Realmente Funcionam

Services não são mágica: um plano de dados implementa seus IPs virtuais. Com kube-proxy no Linux, isso pode significar iptables, nftables ou o backend legado IPVS. Alguns CNIs substituem kube-proxy por um plano de dados eBPF. Vamos desmistificar primeiro o exemplo com iptables.

**kube-proxy** roda em cada node (como um DaemonSet) e observa o API server para mudanças em Services e EndpointSlices. Quando um Service é criado ou atualizado, o kube-proxy programa regras no node para implementar o balanceamento de carga.

### A Cadeia iptables

Quando um pacote chega destinado a um ClusterIP, ele segue esta cadeia:

```
KUBE-SERVICES → KUBE-SVC-<hash> → KUBE-SEP-<hash> (um por endpoint)
```

1. **`KUBE-SERVICES`** — A cadeia de nível superior. Corresponde pacotes pelo IP de destino (o ClusterIP) e pula para a cadeia específica do serviço.
2. **`KUBE-SVC-*`** — A cadeia do serviço. Usa o módulo `statistic` com roteamento baseado em probabilidade para distribuir tráfego entre endpoints (um round-robin simplificado).
3. **`KUBE-SEP-*`** — As cadeias de endpoint. Cada uma faz DNAT do pacote para um IP:porta específico de um Pod.

Se você já escreveu regras NAT de iptables, isso vai parecer muito familiar. A única diferença é que o kube-proxy gera e mantém centenas ou milhares dessas regras automaticamente.

### Modos de Proxy Linux na Base Kubernetes 1.35

kube-proxy suporta estes modos no Linux:

| Modo | Como Funciona | Melhor Para |
|------|--------------|-------------|
| **iptables** | Programa o netfilter pela API iptables; amplamente compatível e padrão no Kubernetes 1.35. | Clusters existentes e a configuração explícita deste laboratório |
| **nftables** | Programa o netfilter pela API nftables; estável desde o Kubernetes **1.33**. | Nodes Linux modernos, após verificar a compatibilidade do kernel e do CNI |
| **IPVS** | Usa a API IPVS do kernel junto com iptables; **marcado como obsoleto no Kubernetes 1.35**. | Instalações legadas, não recomendado para novos clusters |

Não deduza o modo ativo pela presença do comando `iptables`. Inspecione a configuração e os logs do kube-proxy. nftables exige um kernel suficientemente recente (Linux 5.13 ou mais novo) e componentes de rede compatíveis; é o caminho de substituição do IPVS, não uma mudança automática neste laboratório. O frontend de comando `iptables-nft` também não é a mesma coisa que o modo `nftables` do kube-proxy.

---

## DNS: CoreDNS

Todo cluster Kubernetes executa o **CoreDNS** — um servidor DNS implantado como um Deployment no namespace `kube-system`. O CoreDNS observa o API server e cria automaticamente registros DNS para cada Service.

### Formato dos Registros DNS

```
<nome-do-service>.<namespace>.svc.cluster.local
```

Por exemplo, um Service chamado `nginx-svc` no namespace `default` é acessível em:

```
nginx-svc.default.svc.cluster.local
```

Mas como cada Pod tem domínios de busca configurados no `/etc/resolv.conf`, você pode usar nomes mais curtos:

- **`nginx-svc`** — Funciona a partir de Pods no mesmo namespace
- **`nginx-svc.default`** — Funciona a partir de qualquer namespace
- **`nginx-svc.default.svc.cluster.local`** — O nome de domínio totalmente qualificado (FQDN)

### DNS de Pods para StatefulSets

Pods de StatefulSets recebem registros DNS individuais quando pareados com um Headless Service:

```
<nome-do-pod>.<headless-service>.<namespace>.svc.cluster.local
```

Por exemplo: `db-0.db-headless.default.svc.cluster.local`

É assim que StatefulSets fornecem identidade de rede estável sem IPs estáveis.

**Analogia Linux:** CoreDNS é como executar `dnsmasq` para toda a sua infraestrutura — automático, sempre atualizado, e todos o utilizam.

---

## Ingress: Roteamento HTTP

Services lidam com tráfego L4 (TCP/UDP). Mas a maioria das aplicações web precisa de roteamento L7 (HTTP) — roteamento baseado em hostnames, caminhos e headers.

**Ingress** é um objeto da API do Kubernetes que define regras de roteamento HTTP. Ele roteia tráfego HTTP(S) externo para Services internos baseado no hostname e caminho da requisição.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
```

**Analogia Linux:** Isso é exatamente como uma configuração de virtual host do nginx ou Apache:

```nginx
server {
    server_name app.example.com;
    location / {
        proxy_pass http://nginx-backend;
    }
}
```

A diferença principal: ao invés de editar arquivos de configuração do nginx e executar `nginx -s reload`, você submete um documento YAML para o API server, e um **Ingress Controller** o pega e se configura automaticamente.

### Ingress Controllers

Um recurso Ingress sozinho não faz nada. Você precisa de um **Ingress Controller** — um Pod que observa recursos Ingress e configura um reverse proxy correspondente. Opções populares:

- **Traefik** — Controller com manutenção ativa e providers para Kubernetes Ingress e Gateway API; usado neste capítulo.
- **HAProxy Ingress** — Para fãs do HAProxy.
- **Opções cloud-native** — AWS ALB Controller, GCE Ingress Controller.

> **Nota:** O projeto Kubernetes agora recomenda Gateway API ao invés de Ingress. A API Ingress é estável mas congelada — nenhuma nova funcionalidade será adicionada.

**Não instale o controller comunitário `ingress-nginx`, que foi aposentado.** A manutenção terminou em março de 2026; não haverá novas versões nem correções de segurança. Ele é diferente da imagem da aplicação NGINX usada como backend de teste. Veja o [anúncio oficial de aposentadoria](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/).

---

## Gateway API: A Alternativa Moderna

A **Gateway API** é a sucessora do Ingress. Ela resolve várias limitações da API Ingress:

- **Orientada por papéis:** Separa responsabilidades entre administradores de infraestrutura (que gerenciam Gateways) e desenvolvedores de aplicação (que definem regras de roteamento via HTTPRoutes).
- **Mais expressiva:** Suporta correspondência de headers, ponderação de tráfego, espelhamento de requisições, e mais — sem annotations específicas de fornecedor.
- **Multi-protocolo:** Suporta HTTP, HTTPS, TCP, UDP, gRPC, e TLS nativamente.

Os recursos principais:

| Recurso | Quem Gerencia | Propósito |
|---------|--------------|-----------|
| **GatewayClass** | Provedor de infraestrutura | Define o controller (como um StorageClass para rede) |
| **Gateway** | Operador do cluster | A instância real do load balancer (IPs, portas, TLS) |
| **HTTPRoute** | Desenvolvedor de aplicação | Regras de roteamento (hostname, caminho → Service) |

Gateway API não é coberta em profundidade aqui — é um tópico rico por si só. Mas conforme você avança além de clusters de desenvolvimento, é a direção para onde o ecossistema está caminhando.

---

## Debugando Rede

Problemas de rede estão entre os mais comuns no Kubernetes. Aqui está seu kit de ferramentas para debug:

### De Dentro de um Pod

```bash
# Run a debug pod with networking tools
kubectl run debug --image=busybox:1.36 --rm -it --restart=Never -- sh

# Test connectivity to a service
wget -qO- http://nginx-svc
wget -qO- http://nginx-svc.default.svc.cluster.local

# Check DNS resolution
nslookup nginx-svc
nslookup nginx-svc.default.svc.cluster.local

# Check the resolv.conf search domains
cat /etc/resolv.conf
```

### Da Sua Máquina

```bash
# Port-forward to access a service locally
kubectl port-forward svc/nginx-svc 8080:80

# Then in another terminal:
curl http://localhost:8080

# Check Service endpoints (which Pods back the Service)
kubectl get endpoints nginx-svc
kubectl get endpointslices -l kubernetes.io/service-name=nginx-svc

# Describe a Service for full details
kubectl describe svc nginx-svc
```

---

## Comparação Linux ↔ Kubernetes

| Conceito Linux | Equivalente K8s | Diferença Principal |
|---|---|---|
| `iptables -t nat` | kube-proxy / Service | Regras DNAT automatizadas para balanceamento de carga entre Pods |
| `/etc/hosts`, dnsmasq | CoreDNS | DNS automático para todos os Services — sem registros manuais |
| haproxy / nginx reverse proxy | Ingress / Gateway API | Roteamento HTTP declarativo definido como objetos da API |
| Bind em `0.0.0.0:port` | NodePort Service | Expõe em todos os nodes simultaneamente, não apenas em um host |
| VIP (keepalived) | ClusterIP | IP virtual com seleção automática de backend via iptables |
| `ss -tlnp` | `kubectl get svc`, `kubectl get endpoints` | Mapeamento Service → Endpoint substitui mapeamento porta-para-processo |
| Firewall (ufw / regras iptables) | NetworkPolicy | Coberto no Capítulo 11 (Segurança) |
| `traceroute` | `kubectl exec -- traceroute` | Debug de caminho pod-para-pod entre nodes |

---

> ### ⚠️ Onde a Analogia com Linux Quebra
>
> **A rede é plana e abrange todo o cluster.** No Linux, você gerencia a pilha de rede de uma máquina — suas interfaces, rotas e regras de firewall. No Kubernetes, a rede abrange o cluster inteiro: qualquer Pod pode alcançar qualquer outro Pod por IP, independentemente de em qual node ele está. Não existe equivalente no Linux a uma rede onde cada processo em cada máquina pode alcançar diretamente cada outro processo em cada outra máquina — isso é o que o plugin CNI cria.
>
> **Services NÃO são load balancers tradicionais.** Não existe um processo HAProxy sentado em algum lugar tratando conexões. No modo iptables, o kube-proxy programa regras DNAT *em cada node*. Quando um Pod envia um pacote para um ClusterIP, o kernel do node local reescreve o IP de destino inline. É reescrita distribuída de pacotes, não proxy centralizado.
>
> **IPs de Pods são efêmeros — nunca os codifique diretamente.** Isso é o equivalente em rede da efemeridade dos Pods do Capítulo 6. O IP de um Pod muda toda vez que ele é recriado. Sempre use nomes DNS de Services (`nginx-svc`, `nginx-svc.default.svc.cluster.local`), nunca IPs brutos. Em termos Linux: é como uma política que diz "nunca coloque endereços IP em arquivos de configuração — sempre use DNS" — exceto que no Kubernetes, o sistema impõe isso por design.

---

## Laboratório Diagnóstico: Rede no Kubernetes

### Pré-requisitos

Use **Kind v0.31.0**, **kubectl v1.35.0**, Docker em execução e a porta TCP local **8080** livre. A imagem Kubernetes **v1.35.0** abaixo é uma referência didática com versão fixada; em produção, use o patch mais recente com suporte da versão secundária escolhida.

Crie o cluster dedicado `network-lab` abaixo, não um cluster existente do Capítulo 5 ou 6. O mapeamento de portas, o label de node e o modo de proxy fazem parte do laboratório, não são configurações opcionais. Primeiro, confira os nomes existentes:

```bash
kind get clusters
```

Se `network-lab` já existir, pare e inspecione-o. Não o reutilize sem verificar: depois de aceitar a perda dos dados desse laboratório, exclua-o com `kind delete cluster --name network-lab` e recrie-o com esta configuração exata. O Kind não consegue adicionar mapeamentos de portas do host a um cluster existente.

```bash
kind create cluster --name network-lab --image kindest/node:v1.35.0@sha256:452d707d4862f52530247495d180205e029056831160e22870e37e3f6c1ac31f --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  kubeProxyMode: iptables
nodes:
- role: control-plane
  labels:
    ingress-ready: "true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 8080
    listenAddress: "127.0.0.1"
    protocol: TCP
- role: worker
- role: worker
EOF
kubectl config use-context kind-network-lab
kubectl wait --for=condition=Ready nodes --all --timeout=120s
kubectl get nodes -L ingress-ready
```

Somente `network-lab-control-plane` deve ter `ingress-ready=true`. O Pod Traefik do Lab 5 é agendado explicitamente nele, com uma toleration para o control plane e `hostPort: 80`, correspondendo ao mapeamento do Kind. Todos os comandos abaixo usam o contexto `kind-network-lab` e o namespace default, salvo indicação em contrário.

---

### Lab 1: Explorar Rede de Pods

**Passo 1 — Implante dois Pods:**

```bash
kubectl run nginx --image=nginx:1.27 --port=80
kubectl run debug --image=busybox:1.36 --command -- sleep 3600
```

Aguarde ambos os Pods estarem prontos:

```bash
kubectl wait --for=condition=Ready pod/nginx pod/debug --timeout=60s
```

**Passo 2 — Verifique os IPs dos Pods:**

```bash
kubectl get pods -o wide
```

Saída esperada (IPs vão diferir):

```
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE
debug   1/1     Running   0          30s   10.244.1.3   network-lab-worker
nginx   1/1     Running   0          30s   10.244.2.5   network-lab-worker2
```

Observe: cada Pod tem seu próprio IP único, e eles podem estar em nodes diferentes.

**Passo 3 — Teste conectividade pod-para-pod:**

```bash
# Get the nginx Pod IP
NGINX_IP=$(kubectl get pod nginx -o jsonpath='{.status.podIP}')
echo "Nginx Pod IP: $NGINX_IP"

# From the debug pod, connect to nginx by IP
kubectl exec debug -- wget -qO- http://$NGINX_IP
```

Você deve ver o HTML da página de boas-vindas do nginx. Isso funciona mesmo que os Pods estejam em nodes diferentes — esse é o modelo de rede plana em ação.

---

### Lab 2: Criar e Testar Services

**Passo 1 — Crie um Service ClusterIP:**

```bash
kubectl expose pod nginx --name=nginx-svc --port=80 --target-port=80
```

```bash
kubectl get svc nginx-svc
```

Saída esperada:

```
NAME        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
nginx-svc   ClusterIP   10.96.45.123   <none>        80/TCP    5s
```

**Passo 2 — Acesse o Service de dentro do cluster:**

```bash
kubectl exec debug -- wget -qO- http://nginx-svc
```

Isso funciona porque o CoreDNS resolve `nginx-svc` para o ClusterIP.

**Passo 3 — Verifique os endpoints:**

```bash
kubectl get endpoints nginx-svc
```

Você verá o IP do Pod nginx listado como um endpoint de backend.

**Passo 4 — Crie um Service NodePort:**

```bash
kubectl expose pod nginx --name=nginx-nodeport --port=80 --target-port=80 --type=NodePort
```

```bash
kubectl get svc nginx-nodeport
```

Anote a NodePort atribuída (ex: 30XXX).

**Passo 5 — Acesse via port-forward (a forma amigável ao Kind):**

Como os nodes Kind são containers Docker, a forma mais simples de alcançar serviços NodePort do seu host é com `kubectl port-forward`:

```bash
kubectl port-forward svc/nginx-nodeport 9090:80 &
sleep 2
curl -s http://localhost:9090 | head -5
kill %1 2>/dev/null
```

---

### Lab 3: Inspecionar Regras iptables do kube-proxy

**Passo 1 — Olhe a cadeia KUBE-SERVICES:**

```bash
kubectl -n kube-system get configmap kube-proxy -o yaml
docker exec network-lab-worker iptables -t nat -L KUBE-SERVICES -n | head -20
```

Você verá entradas como:

```
Chain KUBE-SERVICES (2 references)
target     prot opt source               destination
KUBE-SVC-XXXX  tcp  --  0.0.0.0/0       10.96.45.123    /* default/nginx-svc cluster IP */ tcp dpt:80
```

**Passo 2 — Siga a cadeia para ver a seleção de endpoints:**

```bash
# Get the KUBE-SVC chain name for our service
docker exec network-lab-worker iptables -t nat -L -n | grep nginx-svc
```

**Passo 3 — Entenda o fluxo:**

```
KUBE-SERVICES (corresponde pelo ClusterIP)
  → KUBE-SVC-<hash> (o service — distribui entre endpoints)
    → KUBE-SEP-<hash> (o endpoint — faz DNAT para um IP de Pod específico)
```

Isso é DNAT padrão do iptables — o mesmo mecanismo que você usaria para encaminhar tráfego de um IP público para um servidor backend no Linux. O kube-proxy apenas automatiza o gerenciamento das regras.

---

### Lab 4: Testar Resolução DNS

**Passo 1 — Verifique o DNS de dentro de um Pod:**

```bash
kubectl exec debug -- nslookup nginx-svc
```

Saída esperada:

```
Server:    10.96.0.10
Address:   10.96.0.10:53

Name:      nginx-svc.default.svc.cluster.local
Address:   10.96.45.123
```

**Passo 2 — Tente o nome totalmente qualificado:**

```bash
kubectl exec debug -- nslookup nginx-svc.default.svc.cluster.local
```

**Passo 3 — Verifique a configuração DNS do Pod:**

```bash
kubectl exec debug -- cat /etc/resolv.conf
```

Saída esperada:

```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

A linha `search` é o motivo pelo qual nomes curtos funcionam. Quando você consulta `nginx-svc`, o resolver adiciona cada domínio de busca em ordem:
1. `nginx-svc.default.svc.cluster.local` → **encontrado!**
2. `nginx-svc.svc.cluster.local` → (tentaria a seguir se o #1 falhasse)
3. `nginx-svc.cluster.local` → (e assim por diante)

A opção `ndots:5` significa que qualquer nome com menos de 5 pontos é tratado como um nome relativo e passa pela lista de busca primeiro. É por isso que o DNS do Kubernetes "simplesmente funciona" dentro de um namespace.

---

### Lab 5: Implantar um Ingress Controller e Criar um Ingress

**Passo 1 — Instale o controller Traefik com versão fixada:**

Este manifesto autocontido usa **`traefik:v3.7.15`** da [versão oficial](https://github.com/traefik/traefik/releases/tag/v3.7.15) e o [provider Kubernetes Ingress](https://doc.traefik.io/traefik/v3.7/reference/install-configuration/providers/kubernetes/kubernetes-ingress/). Não é necessário usar Helm, CRDs do Traefik, dashboard nem annotations específicas do nginx. As permissões de leitura no escopo do cluster são para este laboratório dedicado; em produção, o RBAC deve seguir seu modelo de namespaces e isolamento entre usuários.

Salve e aplique `traefik.yaml`:

```bash
cat <<'EOF' > traefik.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: traefik
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik
  namespace: traefik
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: network-lab-traefik
rules:
- apiGroups: [""]
  resources: ["services", "secrets", "nodes", "namespaces"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["discovery.k8s.io"]
  resources: ["endpointslices"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses", "ingressclasses"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses/status"]
  verbs: ["update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: network-lab-traefik
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: network-lab-traefik
subjects:
- kind: ServiceAccount
  name: traefik
  namespace: traefik
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: traefik
spec:
  controller: traefik.io/ingress-controller
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: traefik
  namespace: traefik
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik
      nodeSelector:
        ingress-ready: "true"
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: traefik
        image: traefik:v3.7.15
        args:
        - --providers.kubernetesingress=true
        - --providers.kubernetesingress.ingressclass=traefik
        - --entrypoints.web.address=:8080
        - --entrypoints.web.asdefault=true
        - --entrypoints.health.address=:8082
        - --ping=true
        - --ping.entrypoint=health
        ports:
        - name: web
          containerPort: 8080
          hostPort: 80
        - name: health
          containerPort: 8082
        readinessProbe:
          httpGet:
            path: /ping
            port: health
        securityContext:
          runAsNonRoot: true
          runAsUser: 65532
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
EOF
kubectl apply -f traefik.yaml
kubectl -n traefik rollout status deployment/traefik --timeout=180s
kubectl -n traefik get pods -o wide
```

**Passo 2 — Verifique o backend dos Labs 1–2:**

```bash
kubectl wait --for=condition=Ready pod/nginx --timeout=120s
kubectl get svc nginx-svc
kubectl get endpointslices -l kubernetes.io/service-name=nginx-svc
```

O Service deve expor a porta 80 e selecionar o Pod nginx pronto. O controller usa seu próprio `hostPort`, portanto não exige um Service LoadBalancer.

**Passo 3 — Crie um recurso Ingress:**

```bash
cat <<'EOF' > ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  ingressClassName: traefik
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
EOF
```

```bash
kubectl apply -f ingress.yaml
```

**Passo 4 — Teste o Ingress:**

O Kind mapeia `127.0.0.1:8080` do host para a porta 80 do control plane; o Pod Traefik agendado ali mapeia esse hostPort para seu entrypoint `web` na porta 8080 do container. A configuração de roteamento pode levar alguns segundos para se propagar:

```bash
curl --fail --show-error --retry 10 --retry-connrefused --retry-all-errors --retry-delay 2 http://127.0.0.1:8080/
```

Você deve ver a página de boas-vindas do nginx. O fluxo de tráfego é:

```
127.0.0.1:8080 → porta 80 do control plane Kind → Traefik :8080 → nginx-svc:80 → Pod nginx:80
```

**Passo 5 — Inspecione o Ingress:**

```bash
kubectl get ingress
kubectl describe ingress nginx-ingress
```

O campo `ADDRESS` do Ingress pode permanecer vazio neste laboratório que usa somente hostPort; não há um load balancer de nuvem. Verifique a resposta HTTP e os logs do controller, em vez de tratar esse campo isolado como um health check.

---

### Limpeza

```bash
kubectl delete ingress nginx-ingress
kubectl delete -f traefik.yaml
kubectl delete svc nginx-svc nginx-nodeport
kubectl delete pod nginx debug
kind delete cluster --name network-lab
rm -f ingress.yaml traefik.yaml
```

---

## Principais Conclusões

1. **Cada Pod recebe seu próprio IP, e todos os Pods podem alcançar todos os outros Pods sem NAT.** Este modelo de rede plana é a regra de rede mais importante do Kubernetes, implementada por plugins CNI (Kindnet, Calico, Cilium, Flannel).

2. **Services fornecem endpoints estáveis para Pods efêmeros.** ClusterIP para tráfego interno, NodePort para acesso de desenvolvimento, LoadBalancer para acesso externo em produção, e Headless para descoberta direta de Pods.

3. **kube-proxy programa o plano de dados dos Services em cada node.** No Linux, há suporte a iptables e nftables (estável desde a 1.33); IPVS foi marcado como obsoleto na 1.35. Este laboratório usa iptables explicitamente para que os comandos de inspeção de regras façam sentido.

4. **CoreDNS dá a cada Service um nome DNS automático.** Use `<svc>.<namespace>.svc.cluster.local` (ou apenas `<svc>` dentro do mesmo namespace). Nunca codifique IPs de Pods diretamente.

5. **Ingress fornece roteamento HTTP L7 — baseado em hostname e caminho.** Ele requer um controller com manutenção ativa, como o Traefik, e uma IngressClass adequada. O controller comunitário ingress-nginx foi aposentado. Gateway API é a substituição moderna com mais funcionalidades e melhor separação de papéis.

6. **Domínios de busca DNS tornam a descoberta de serviços transparente.** A linha `search` no `/etc/resolv.conf` permite usar nomes curtos como `nginx-svc` ao invés do FQDN completo. A configuração `ndots:5` garante que nomes relativos passem pela lista de busca primeiro.

7. **Ao debugar rede: pense em camadas.** Conectividade de IP de Pod primeiro (Pods conseguem se alcançar?), depois resolução de Service (o DNS funciona?), depois mapeamento de endpoints (os Pods estão listados como endpoints?), depois acesso externo (Ingress/NodePort/LoadBalancer).

---

## Leitura Complementar

- [The Kubernetes Network Model](https://kubernetes.io/docs/concepts/services-networking/)
- [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)
- [Kind — Ingress](https://kind.sigs.k8s.io/docs/user/ingress/)
- [Traefik v3.7 — Provider Kubernetes Ingress](https://doc.traefik.io/traefik/v3.7/reference/install-configuration/providers/kubernetes/kubernetes-ingress/)
- [Versão Traefik v3.7.15](https://github.com/traefik/traefik/releases/tag/v3.7.15)
- [IPs virtuais e modos de proxy de Services](https://kubernetes.io/docs/reference/networking/virtual-ips/)
- [Configuração do Kind — mapeamentos de portas e labels de nodes](https://kind.sigs.k8s.io/docs/user/configuration/)

---

**Anterior:** [Capítulo 6 — Pods e Workloads](06-pods-and-workloads.md)
**Próximo:** [Capítulo 8 — Armazenamento e Persistência](08-storage-and-persistence.md)
