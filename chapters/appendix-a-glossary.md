# Appendix A: Linux-to-Kubernetes Glossary

*A quick-reference mapping of familiar Linux concepts to their Kubernetes equivalents.*

---

## Process Management

| Linux | Kubernetes | Notes |
|-------|-----------|-------|
| process | Pod | A Pod is the smallest deployable unit; it wraps one or more containers (analogous to a process group) |
| PID 1 / init / systemd | kubelet | The kubelet is the node-level agent that starts and supervises Pods |
| fork/exec | Container creation | The container runtime (containerd) spawns containers similar to how the kernel forks processes |
| process group | Pod (multi-container) | Containers in the same Pod share network and IPC namespaces, like processes in a group |
| daemon | DaemonSet | Ensures exactly one Pod runs on every (or selected) node in the cluster |
| cron | CronJob | Schedules Pods to run on a time-based pattern (cron syntax) |
| at (one-shot scheduled task) | Job | Runs a Pod to completion and tracks success/failure |
| SIGTERM | Pod termination (preStop + SIGTERM) | Kubernetes sends SIGTERM, then waits `terminationGracePeriodSeconds` before SIGKILL |
| SIGKILL | Force delete (`--grace-period=0 --force`) | Immediate Pod termination without graceful shutdown |
| nice / renice | Resource requests and limits | `resources.requests` sets scheduling priority; `resources.limits` caps usage |
| ulimit | LimitRange / ResourceQuota | LimitRange sets per-Pod defaults; ResourceQuota caps namespace-wide totals |
| systemd unit file | Pod spec / Deployment manifest | Declarative YAML describing the desired state of a workload |
| `Restart=always` | `restartPolicy: Always` | Default Pod restart policy — kubelet restarts failed containers automatically |
| `systemctl start/stop/restart` | `kubectl rollout restart`, `kubectl scale` | Rolling restarts or scaling replicas to zero and back |

## Networking

| Linux | Kubernetes | Notes |
|-------|-----------|-------|
| IP address (per host) | Pod IP (per Pod) | Every Pod gets a unique, routable IP address from the cluster CIDR |
| iptables / nftables | kube-proxy (iptables/IPVS mode) | kube-proxy programs packet-routing rules for Service ClusterIPs |
| NAT (SNAT/DNAT) | Service (ClusterIP / NodePort) | Services provide stable virtual IPs that DNAT traffic to backend Pods |
| port binding (`bind()`) | `containerPort` / Service `port` | Containers declare ports; Services map external ports to container ports |
| /etc/hosts | CoreDNS (in-cluster DNS) | Kubernetes auto-registers `<svc>.<namespace>.svc.cluster.local` entries |
| DNS resolver (/etc/resolv.conf) | Pod DNS policy / CoreDNS | Each Pod's resolv.conf points to the cluster DNS service |
| socket (`ss -tlnp`) | `kubectl get endpoints` | Endpoints list the IP:port pairs backing each Service |
| reverse proxy (nginx / haproxy) | Ingress / Gateway API | L7 load balancing with path-based routing and TLS termination |
| network namespace | Pod network namespace | All containers in a Pod share one network namespace (one IP, one loopback) |
| bridge (`brctl`) | CNI plugin bridge (e.g., Calico, Cilium) | The CNI plugin configures virtual bridges/routes for Pod networking |
| VLAN | NetworkPolicy + namespace isolation | Logical traffic segmentation via label selectors and policy rules |
| traceroute | `kubectl debug` + traceroute inside Pod | Run diagnostic Pods to trace network paths within the cluster |
| firewall rules (ufw / iptables) | NetworkPolicy | Declare allowed ingress/egress traffic per Pod using label selectors |

## Storage

| Linux | Kubernetes | Notes |
|-------|-----------|-------|
| /etc/fstab | PersistentVolume (PV) + StorageClass | PVs define available storage; StorageClass enables dynamic provisioning |
| mount | volumeMounts (in Pod spec) | Attach a volume at a specific path inside the container filesystem |
| block device (/dev/sda) | PersistentVolume (block mode) | `volumeMode: Block` exposes raw block devices to Pods |
| filesystem (ext4/xfs) | PersistentVolume (filesystem mode) | Default mode — the volume is formatted and mounted as a directory |
| LVM (Logical Volume Manager) | StorageClass + CSI driver | CSI drivers manage provisioning, snapshots, and resizing dynamically |
| NFS mount | PersistentVolume (NFS) / CSI NFS driver | `nfs` volume type or a CSI driver provides shared ReadWriteMany access |
| tmpfs | `emptyDir` with `medium: Memory` | An ephemeral in-memory volume that disappears when the Pod terminates |
| bind mount | `hostPath` volume | Mounts a directory from the host into the Pod — avoid in production |
| LVM snapshot | VolumeSnapshot | CSI snapshot API creates point-in-time copies of PersistentVolumes |
| disk quota | ResourceQuota (storage) / `resources.limits.ephemeral-storage` | Limit total PVC storage per namespace or ephemeral disk per container |

## Configuration

| Linux | Kubernetes | Notes |
|-------|-----------|-------|
| /etc/ config files | ConfigMap | Key-value pairs or whole files mounted into Pods as volumes or env vars |
| environment variables | `env` / `envFrom` in Pod spec | Set variables directly or inject all keys from a ConfigMap/Secret |
| /etc/profile.d/ (shell init scripts) | Init containers | Run setup logic before the main application container starts |
| config file reload (SIGHUP) | Rolling update / ConfigMap reload | Change the ConfigMap and restart Pods (or use a sidecar reloader) |
| alternatives system (`update-alternatives`) | Label selectors on Services | Route traffic to different Pod versions by updating label selectors |
| symlinks | Projected volumes | Combine multiple ConfigMaps/Secrets/ServiceAccount tokens into one mount |

## Security

| Linux | Kubernetes | Notes |
|-------|-----------|-------|
| users / groups (UID/GID) | `runAsUser` / `runAsGroup` (SecurityContext) | Set the UID/GID under which the container process runs |
| chmod / chown | `fsGroup` in SecurityContext | Kubernetes sets group ownership on mounted volumes automatically |
| /etc/passwd, /etc/shadow | ServiceAccount + RBAC | Identity in Kubernetes is a ServiceAccount; permissions use Role/ClusterRole |
| sudoers | ClusterRoleBinding (cluster-admin) | Grants elevated privileges — equivalent to root across the cluster |
| Linux capabilities (`capsh`) | `securityContext.capabilities.add/drop` | Add or drop specific kernel capabilities (e.g., NET_ADMIN, SYS_TIME) |
| SELinux / AppArmor | Pod Security Admission / securityContext | Enforce profiles via `seLinuxOptions` or `appArmorProfile` in the Pod spec |
| firewall (ufw / iptables) | NetworkPolicy | Restrict Pod-to-Pod traffic at L3/L4 with label-based rules |
| SSH keys (authorized_keys) | kubeconfig certificates / OIDC tokens | Authenticate to the API server via client certificates or identity providers |
| Certificate authority (CA) | Cluster CA / cert-manager | The cluster CA signs kubelet and API server certs; cert-manager automates TLS |

## Monitoring and Logging

| Linux | Kubernetes | Notes |
|-------|-----------|-------|
| top / htop | `kubectl top pods` / `kubectl top nodes` | Requires Metrics Server; shows CPU/memory usage per Pod or node |
| vmstat / iostat / sar | Prometheus + node-exporter | Detailed host-level metrics collected and stored as time-series data |
| journalctl | `kubectl logs` | Stream stdout/stderr from containers; add `-f` for follow and `-p` for previous |
| syslog (/var/log/syslog) | Cluster-level logging (Fluent Bit / Fluentd) | Aggregate logs from all Pods and ship to a central backend |
| dmesg (kernel ring buffer) | Node-level logs / `kubectl debug node/` | Inspect kernel messages by debugging directly on the node |
| /proc filesystem | Metrics Server / cAdvisor | cAdvisor exposes per-container resource metrics from the kernel |
| strace (syscall tracing) | `kubectl debug` with ephemeral container | Attach a debug container with strace/tcpdump to a running Pod |
| lsof (open files/sockets) | `kubectl exec` + lsof inside container | Run diagnostic commands inside a running container |
| df -h (disk usage) | `kubectl exec -- df -h` / PVC metrics | Check filesystem usage inside containers or monitor PVC utilization |

## Package Management and Deployment

| Linux | Kubernetes | Notes |
|-------|-----------|-------|
| apt / yum / dnf | Helm | The de facto package manager for Kubernetes — installs charts (bundles of manifests) |
| .deb / .rpm (package file) | Helm chart / Kustomize overlay | A packaged, versioned set of Kubernetes manifests |
| package repository (apt repo / yum repo) | Helm repository / OCI registry | Stores and serves chart versions (e.g., Artifact Hub, container registries) |
| dpkg-reconfigure | `helm upgrade --reuse-values` | Reconfigure a deployed release with new values without reinstalling |
| Ansible / Puppet (config management) | GitOps (Argo CD / Flux) | Declarative, Git-driven continuous delivery reconciles cluster state |
| git hooks (pre-commit, post-receive) | Admission webhooks (validating / mutating) | Intercept API requests to enforce policies or inject sidecars |

## System Administration

| Linux | Kubernetes | Notes |
|-------|-----------|-------|
| hostname | Node name / Pod name | Each node and Pod has a DNS-resolvable name within the cluster |
| reboot | `kubectl drain` + node restart | Drain evicts Pods gracefully before you reboot or replace a node |
| maintenance mode | `kubectl cordon` | Marks a node as unschedulable — existing Pods keep running |
| backup (rsync / tar) | Velero / etcd snapshot | Velero backs up resources and PV data; etcd snapshots preserve cluster state |
| crontab (user-level scheduling) | CronJob | Schedule recurring workloads with familiar cron expressions |
| logrotate | Log rotation in container runtime | containerd/kubelet handle log file rotation; configure via kubelet flags |
| systemd timers | CronJob (for in-cluster) / external triggers | CronJobs replace systemd timers for scheduled tasks inside the cluster |

---

*Navigation: [← Chapter 15: Next Steps](15-next-steps.md) | [Appendix B: kubectl Cheatsheet →](appendix-b-cheatsheet.md)*
