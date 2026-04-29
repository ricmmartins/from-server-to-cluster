# Chapter 7: Networking

*"On a single server, networking is configuration. In a cluster, networking is architecture."*

---

## From One Machine to a Flat Network

As a Linux admin, you know networking inside out. You've configured IP addresses with `ip addr`, set up routes with `ip route`, written iptables rules by hand, debugged DNS with `dig` and `nslookup`, and probably set up more than a few nginx reverse proxies. All of that knowledge directly applies here — Kubernetes networking is built on the same Linux primitives.

But there's one fundamental difference: on Linux, you manage the network of **one machine**. In Kubernetes, you're dealing with a **cluster-wide network** where every Pod gets its own IP address and can talk to every other Pod directly, regardless of which node it's on. No NAT, no port mapping, no manual routing tables.

If that sounds too good to be true — it kind of is. Somebody has to set up all those routes and rules, and that somebody is the Container Network Interface (CNI) plugin and kube-proxy. This chapter pulls back the curtain on how Kubernetes networking actually works, and how it maps to the Linux networking you already know.

---

## The Kubernetes Networking Model

Before we get into Services and Ingress, you need to understand the foundational rules. Kubernetes mandates a networking model with these requirements:

1. **Every Pod gets its own unique IP address** — not shared with other Pods, not NAT'd.
2. **All Pods can communicate with all other Pods** on any node without NAT.
3. **Agents on a node** (kubelet, system daemons) can communicate with all Pods on that node.

This is called the **flat network model**, and it's fundamentally different from Docker's default bridge networking (where containers on different hosts can't talk to each other without explicit port publishing).

### How It Works Under the Hood

The Kubernetes project doesn't implement networking itself — it defines the requirements and delegates to **CNI (Container Network Interface) plugins**. Popular CNI plugins include:

- **Kindnet** — The default for Kind clusters. Simple, lightweight.
- **Calico** — Feature-rich, supports NetworkPolicy, BGP routing.
- **Cilium** — eBPF-based, high performance, advanced observability.
- **Flannel** — Simple overlay network using VXLAN.

Each CNI plugin takes a different approach (VXLAN tunnels, BGP routing, eBPF), but the result is the same: every Pod gets an IP, and every Pod can reach every other Pod.

### The Linux Analogy

Think of it this way: if you had 10 Linux servers and wanted every process on every server to talk to every process on every other server by IP — no NAT, no port conflicts — you'd need to set up routing between all the machines, assign non-overlapping IP ranges, and maintain it all. That's exactly what a CNI plugin does, automatically.

---

## Services: The Stable Endpoint

Pods come and go. Deployments scale up and down. Pod IPs change every time a Pod is recreated. So how do you connect to something reliably?

**Services** provide a stable network endpoint — a fixed IP address and DNS name — that routes traffic to a set of Pods selected by labels. Think of a Service as a DNS-based load balancer that automatically updates its backend list as Pods appear and disappear.

### ClusterIP (Default)

A **ClusterIP** Service gets a virtual IP address that's only reachable from inside the cluster. It's the default Service type.

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

**Linux analogy:** Like a VIP managed by keepalived, but instead of failover between two machines, it load-balances across all matching Pods.

When another Pod in the cluster calls `nginx-svc` (or `nginx-svc.default.svc.cluster.local`), the request hits the ClusterIP, which distributes it to one of the backend Pods.

### NodePort

A **NodePort** Service exposes the service on a static port on every node's IP address. External clients can reach the service by hitting `<any-node-ip>:<node-port>`.

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

**Linux analogy:** Like running `iptables -t nat -A PREROUTING -p tcp --dport 30080 -j DNAT --to-destination <backend-ip>:80` on every server in your fleet. Any machine can receive the request and forward it to the right backend.

NodePort range is 30000-32767 by default. This is rarely used in production (LoadBalancer or Ingress are better), but it's handy for development and local clusters.

### LoadBalancer

A **LoadBalancer** Service requests an external load balancer from the cloud provider (or a local solution like MetalLB or cloud-provider-kind). It's the standard way to expose services to the internet.

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

**Linux analogy:** Like having an HAProxy or nginx reverse proxy sitting in front of your servers, automatically configured with the correct backend list. You don't manage the proxy — the cloud provider does.

In Kind clusters, you can use [cloud-provider-kind](https://github.com/kubernetes-sigs/cloud-provider-kind) to enable LoadBalancer Services locally.

### Headless Services

Sometimes you don't want load balancing — you want to discover all individual Pod IPs directly. A **Headless Service** (`clusterIP: None`) does exactly this.

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

Instead of returning a single ClusterIP, a DNS lookup for `db-headless` returns the IP addresses of all matching Pods. This is essential for StatefulSets where clients need to connect to specific Pods (e.g., `db-0.db-headless`).

**Linux analogy:** Like `/etc/hosts` entries pointing to every individual server, instead of a load balancer VIP.

---

## kube-proxy and iptables: How Services Actually Work

Services aren't magic — they're iptables rules. Let's demystify it.

**kube-proxy** runs on every node (as a DaemonSet) and watches the API server for Service and EndpointSlice changes. When a Service is created or updated, kube-proxy programs rules on the node to implement the load balancing.

### The iptables Chain

When a packet arrives destined for a ClusterIP, it follows this chain:

```
KUBE-SERVICES → KUBE-SVC-<hash> → KUBE-SEP-<hash> (one per endpoint)
```

1. **`KUBE-SERVICES`** — The top-level chain. Matches packets by destination IP (the ClusterIP) and jumps to the service-specific chain.
2. **`KUBE-SVC-*`** — The service chain. Uses `statistic` module with probability-based routing to distribute traffic across endpoints (poor man's round-robin).
3. **`KUBE-SEP-*`** — The endpoint chains. Each one DNATs the packet to a specific Pod IP:port.

If you've ever written iptables NAT rules, this will feel very familiar. The only difference is that kube-proxy generates and maintains hundreds or thousands of these rules automatically.

### iptables vs. IPVS Mode

kube-proxy supports two modes:

| Mode | How It Works | Best For |
|------|-------------|----------|
| **iptables** (default) | NAT rules in kernel netfilter. O(n) rule matching. | Small-to-medium clusters |
| **IPVS** | Linux Virtual Server (kernel module). Hash-based lookup. O(1). | Large clusters with many Services |

For most clusters, iptables mode works fine. IPVS becomes important when you have thousands of Services and the iptables rule count causes performance issues.

---

## DNS: CoreDNS

Every Kubernetes cluster runs **CoreDNS** — a DNS server deployed as a Deployment in the `kube-system` namespace. CoreDNS watches the API server and automatically creates DNS records for every Service.

### DNS Record Format

```
<service-name>.<namespace>.svc.cluster.local
```

For example, a Service named `nginx-svc` in the `default` namespace is reachable at:

```
nginx-svc.default.svc.cluster.local
```

But because every Pod has search domains configured in `/etc/resolv.conf`, you can use shorter names:

- **`nginx-svc`** — Works from Pods in the same namespace
- **`nginx-svc.default`** — Works from any namespace
- **`nginx-svc.default.svc.cluster.local`** — The fully qualified domain name (FQDN)

### Pod DNS for StatefulSets

StatefulSet Pods get individual DNS records when paired with a Headless Service:

```
<pod-name>.<headless-service>.<namespace>.svc.cluster.local
```

For example: `db-0.db-headless.default.svc.cluster.local`

This is how StatefulSets provide stable network identity without stable IPs.

**Linux analogy:** CoreDNS is like running `dnsmasq` for your entire infrastructure — automatic, always up to date, and everyone uses it.

---

## Ingress: HTTP Routing

Services handle L4 (TCP/UDP) traffic. But most web applications need L7 (HTTP) routing — routing based on hostnames, paths, and headers.

**Ingress** is a Kubernetes API object that defines HTTP routing rules. It routes external HTTP(S) traffic to internal Services based on the request's hostname and path.

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

**Linux analogy:** This is exactly like an nginx or Apache virtual host configuration:

```nginx
server {
    server_name app.example.com;
    location / {
        proxy_pass http://nginx-backend;
    }
}
```

The key difference: instead of editing nginx config files and running `nginx -s reload`, you submit a YAML document to the API server, and an **Ingress Controller** picks it up and configures itself automatically.

### Ingress Controllers

An Ingress resource by itself does nothing. You need an **Ingress Controller** — a Pod that watches Ingress resources and configures a reverse proxy accordingly. Popular choices:

- **ingress-nginx** — The most common. Runs nginx under the hood.
- **Traefik** — Auto-discovery, dashboard, Let's Encrypt integration.
- **HAProxy Ingress** — For HAProxy fans.
- **Cloud-native options** — AWS ALB Controller, GCE Ingress Controller.

> **Note:** The Kubernetes project now recommends Gateway API over Ingress. The Ingress API is stable but frozen — no new features will be added.

---

## Gateway API: The Modern Alternative

The **Gateway API** is the successor to Ingress. It solves several limitations of the Ingress API:

- **Role-oriented:** Separates concerns between infrastructure administrators (who manage Gateways) and application developers (who define routing rules via HTTPRoutes).
- **More expressive:** Supports header matching, traffic weighting, request mirroring, and more — without vendor-specific annotations.
- **Multi-protocol:** Supports HTTP, HTTPS, TCP, UDP, gRPC, and TLS natively.

The core resources:

| Resource | Who Manages It | Purpose |
|----------|---------------|---------|
| **GatewayClass** | Infrastructure provider | Defines the controller (like a StorageClass for networking) |
| **Gateway** | Cluster operator | The actual load balancer instance (IPs, ports, TLS) |
| **HTTPRoute** | Application developer | Routing rules (hostname, path → Service) |

Gateway API is not covered in depth here — it's a rich topic on its own. But as you move beyond development clusters, it's the direction the ecosystem is heading.

---

## Debugging Networking

Networking problems are among the most common Kubernetes issues. Here's your debugging toolkit:

### From Inside a Pod

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

### From Your Machine

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

## Linux ↔ Kubernetes Comparison

| Linux Concept | K8s Equivalent | Key Difference |
|---|---|---|
| `iptables -t nat` | kube-proxy / Service | Automated DNAT rules for load balancing across Pods |
| `/etc/hosts`, dnsmasq | CoreDNS | Automatic DNS for all Services — no manual records |
| haproxy / nginx reverse proxy | Ingress / Gateway API | Declarative HTTP routing defined as API objects |
| Bind to `0.0.0.0:port` | NodePort Service | Exposes on all nodes simultaneously, not just one host |
| VIP (keepalived) | ClusterIP | Virtual IP with automatic backend selection via iptables |
| `ss -tlnp` | `kubectl get svc`, `kubectl get endpoints` | Service → Endpoint mapping replaces port-to-process mapping |
| Firewall (ufw / iptables rules) | NetworkPolicy | Covered in Chapter 11 (Security) |
| `traceroute` | `kubectl exec -- traceroute` | Pod-to-pod path debugging across nodes |

---

> ### ⚠️ Where the Linux Analogy Breaks
>
> **The network is flat and cluster-wide.** On Linux, you manage one machine's network stack — its interfaces, routes, and firewall rules. In Kubernetes, the network spans the entire cluster: any Pod can reach any other Pod by IP, regardless of which node it's on. There's no Linux equivalent to a network where every process on every machine can directly reach every other process on every other machine — that's what the CNI plugin creates.
>
> **Services are NOT traditional load balancers.** There's no HAProxy process sitting somewhere handling connections. In iptables mode, kube-proxy programs DNAT rules *on every node*. When a Pod sends a packet to a ClusterIP, the local node's kernel rewrites the destination IP inline. It's distributed packet rewriting, not centralized proxying.
>
> **Pod IPs are ephemeral — never hardcode them.** This is the networking equivalent of the Pod ephemerality from Chapter 6. A Pod's IP changes every time it's recreated. Always use Service DNS names (`nginx-svc`, `nginx-svc.default.svc.cluster.local`), never raw IPs. In Linux terms: it's like a policy that says "never put IP addresses in config files — always use DNS" — except in Kubernetes, the system enforces it by design.

---

## Diagnostic Lab: Kubernetes Networking

### Prerequisites

Make sure your Kind cluster is running (with at least two worker nodes):

```bash
kind get clusters
```

If not, create one:

```bash
kind create cluster --name lab-cluster --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 80
    hostPort: 8080
    protocol: TCP
- role: worker
- role: worker
EOF
```

---

### Lab 1: Explore Pod Networking

**Step 1 — Deploy two Pods:**

```bash
kubectl run nginx --image=nginx:1.27 --port=80
kubectl run debug --image=busybox:1.36 --command -- sleep 3600
```

Wait for both Pods to be ready:

```bash
kubectl wait --for=condition=Ready pod/nginx pod/debug --timeout=60s
```

**Step 2 — Check Pod IPs:**

```bash
kubectl get pods -o wide
```

Expected output (IPs will differ):

```
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE
debug   1/1     Running   0          30s   10.244.1.3   lab-cluster-worker
nginx   1/1     Running   0          30s   10.244.2.5   lab-cluster-worker2
```

Notice: each Pod has its own unique IP, and they may be on different nodes.

**Step 3 — Test pod-to-pod connectivity:**

```bash
# Get the nginx Pod IP
NGINX_IP=$(kubectl get pod nginx -o jsonpath='{.status.podIP}')
echo "Nginx Pod IP: $NGINX_IP"

# From the debug pod, connect to nginx by IP
kubectl exec debug -- wget -qO- http://$NGINX_IP
```

You should see the nginx welcome page HTML. This works even though the Pods are on different nodes — that's the flat network model in action.

---

### Lab 2: Create and Test Services

**Step 1 — Create a ClusterIP Service:**

```bash
kubectl expose pod nginx --name=nginx-svc --port=80 --target-port=80
```

```bash
kubectl get svc nginx-svc
```

Expected output:

```
NAME        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
nginx-svc   ClusterIP   10.96.45.123   <none>        80/TCP    5s
```

**Step 2 — Access the Service from within the cluster:**

```bash
kubectl exec debug -- wget -qO- http://nginx-svc
```

This works because CoreDNS resolves `nginx-svc` to the ClusterIP.

**Step 3 — Check the endpoints:**

```bash
kubectl get endpoints nginx-svc
```

You'll see the nginx Pod's IP listed as a backend endpoint.

**Step 4 — Create a NodePort Service:**

```bash
kubectl expose pod nginx --name=nginx-nodeport --port=80 --target-port=80 --type=NodePort
```

```bash
kubectl get svc nginx-nodeport
```

Note the NodePort assigned (e.g., 30XXX).

**Step 5 — Access via port-forward (the Kind-friendly way):**

Since Kind nodes are Docker containers, the simplest way to reach NodePort services from your host is `kubectl port-forward`:

```bash
kubectl port-forward svc/nginx-nodeport 9090:80 &
sleep 2
curl -s http://localhost:9090 | head -5
kill %1 2>/dev/null
```

---

### Lab 3: Inspect kube-proxy iptables Rules

**Step 1 — Look at the KUBE-SERVICES chain:**

```bash
docker exec lab-cluster-worker iptables -t nat -L KUBE-SERVICES -n | head -20
```

You'll see entries like:

```
Chain KUBE-SERVICES (2 references)
target     prot opt source               destination
KUBE-SVC-XXXX  tcp  --  0.0.0.0/0       10.96.45.123    /* default/nginx-svc cluster IP */ tcp dpt:80
```

**Step 2 — Follow the chain to see endpoint selection:**

```bash
# Get the KUBE-SVC chain name for our service
docker exec lab-cluster-worker iptables -t nat -L -n | grep nginx-svc
```

**Step 3 — Understand the flow:**

```
KUBE-SERVICES (matches by ClusterIP)
  → KUBE-SVC-<hash> (the service — distributes across endpoints)
    → KUBE-SEP-<hash> (the endpoint — DNATs to a specific Pod IP)
```

This is standard iptables DNAT — the same mechanism you'd use to forward traffic from a public IP to a backend server on Linux. kube-proxy just automates the rule management.

---

### Lab 4: Test DNS Resolution

**Step 1 — Check DNS from a Pod:**

```bash
kubectl exec debug -- nslookup nginx-svc
```

Expected output:

```
Server:    10.96.0.10
Address:   10.96.0.10:53

Name:      nginx-svc.default.svc.cluster.local
Address:   10.96.45.123
```

**Step 2 — Try the fully qualified name:**

```bash
kubectl exec debug -- nslookup nginx-svc.default.svc.cluster.local
```

**Step 3 — Check the Pod's DNS configuration:**

```bash
kubectl exec debug -- cat /etc/resolv.conf
```

Expected output:

```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

The `search` line is why short names work. When you look up `nginx-svc`, the resolver appends each search domain in order:
1. `nginx-svc.default.svc.cluster.local` → **match!**
2. `nginx-svc.svc.cluster.local` → (would try next if #1 failed)
3. `nginx-svc.cluster.local` → (and so on)

The `ndots:5` option means any name with fewer than 5 dots is treated as a relative name and goes through the search list first. This is why Kubernetes DNS "just works" within a namespace.

---

### Lab 5: Deploy an Ingress Controller and Create an Ingress

**Step 1 — Install the NGINX Ingress Controller for Kind:**

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

**Step 2 — Wait for the controller to be ready:**

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

**Step 3 — Create an Ingress resource:**

```bash
cat <<'EOF' > ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
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

**Step 4 — Test the Ingress:**

The Kind cluster was created with port 80 mapped to host port 8080 (from our cluster config). The NGINX Ingress Controller uses hostPort 80, so:

```bash
curl -s http://localhost:8080 | head -10
```

You should see the nginx welcome page. The traffic flow is:

```
localhost:8080 → Kind container port 80 → Ingress Controller → nginx-svc → nginx Pod
```

**Step 5 — Inspect the Ingress:**

```bash
kubectl get ingress
kubectl describe ingress nginx-ingress
```

---

### Clean Up

```bash
kubectl delete ingress nginx-ingress
kubectl delete svc nginx-svc nginx-nodeport
kubectl delete pod nginx debug
rm -f ingress.yaml
```

---

## Key Takeaways

1. **Every Pod gets its own IP, and all Pods can reach all other Pods without NAT.** This flat network model is Kubernetes' most important networking rule, implemented by CNI plugins (Kindnet, Calico, Cilium, Flannel).

2. **Services provide stable endpoints for ephemeral Pods.** ClusterIP for internal traffic, NodePort for development access, LoadBalancer for production external access, and Headless for direct Pod discovery.

3. **kube-proxy implements Services using iptables DNAT rules on every node.** There's no central proxy — it's distributed packet rewriting at the kernel level. For large clusters, IPVS mode offers better performance.

4. **CoreDNS gives every Service an automatic DNS name.** Use `<svc>.<namespace>.svc.cluster.local` (or just `<svc>` within the same namespace). Never hardcode Pod IPs.

5. **Ingress provides L7 HTTP routing — hostname and path-based.** It requires an Ingress Controller (like ingress-nginx) to actually work. Gateway API is the modern replacement with more features and better role separation.

6. **DNS search domains make service discovery seamless.** The `search` line in `/etc/resolv.conf` lets you use short names like `nginx-svc` instead of the full FQDN. The `ndots:5` setting ensures relative names hit the search list first.

7. **When debugging networking: think in layers.** Pod IP connectivity first (can Pods reach each other?), then Service resolution (does DNS work?), then endpoint mapping (are Pods listed as endpoints?), then external access (Ingress/NodePort/LoadBalancer).

---

## Further Reading

- [The Kubernetes Network Model](https://kubernetes.io/docs/concepts/services-networking/)
- [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)
- [Kind — Ingress](https://kind.sigs.k8s.io/docs/user/ingress/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/deploy/)

---

**Previous:** [Chapter 6 — Pods and Workloads](06-pods-and-workloads.md)
**Next:** [Chapter 8 — Storage and Persistence](08-storage-and-persistence.md)
