# Apêndice A: Glossário Linux-para-Kubernetes

*Uma referência rápida mapeando conceitos familiares do Linux para seus equivalentes no Kubernetes.*

---

## Gerenciamento de Processos

| Linux | Kubernetes | Notas |
|-------|-----------|-------|
| process | Pod | Um Pod é a menor unidade implantável; ele encapsula um ou mais containers (análogo a um grupo de processos) |
| PID 1 / init / systemd | kubelet | O kubelet é o agente no nível do nó que inicia e supervisiona os Pods |
| fork/exec | Criação de container | O container runtime (containerd) cria containers de forma similar a como o kernel faz fork de processos |
| process group | Pod (multi-container) | Containers no mesmo Pod compartilham namespaces de rede e IPC, como processos em um grupo |
| daemon | DaemonSet | Garante que exatamente um Pod rode em cada (ou em nós selecionados) nó do cluster |
| cron | CronJob | Agenda Pods para execução em um padrão baseado em tempo (sintaxe cron) |
| at (tarefa agendada única) | Job | Executa um Pod até a conclusão e rastreia sucesso/falha |
| SIGTERM | Terminação do Pod (preStop + SIGTERM) | O Kubernetes envia SIGTERM, depois aguarda `terminationGracePeriodSeconds` antes do SIGKILL |
| SIGKILL | Exclusão forçada (`--grace-period=0 --force`) | Terminação imediata do Pod sem shutdown gracioso |
| nice / renice | Resource requests e limits | `resources.requests` define prioridade de agendamento; `resources.limits` limita o uso |
| ulimit | LimitRange / ResourceQuota | LimitRange define padrões por Pod; ResourceQuota limita totais por namespace |
| systemd unit file | Pod spec / Deployment manifest | YAML declarativo descrevendo o estado desejado de uma carga de trabalho |
| `Restart=always` | `restartPolicy: Always` | Política padrão de reinício do Pod — kubelet reinicia containers com falha automaticamente |
| `systemctl start/stop/restart` | `kubectl rollout restart`, `kubectl scale` | Reinícios progressivos ou escalar réplicas para zero e voltar |

## Rede

| Linux | Kubernetes | Notas |
|-------|-----------|-------|
| Endereço IP (por host) | Pod IP (por Pod) | Todo Pod recebe um endereço IP único e roteável do CIDR do cluster |
| iptables / nftables | kube-proxy (modo iptables/IPVS) | kube-proxy programa regras de roteamento de pacotes para ClusterIPs dos Services |
| NAT (SNAT/DNAT) | Service (ClusterIP / NodePort) | Services fornecem IPs virtuais estáveis que fazem DNAT do tráfego para Pods backend |
| port binding (`bind()`) | `containerPort` / Service `port` | Containers declaram portas; Services mapeiam portas externas para portas do container |
| /etc/hosts | CoreDNS (DNS interno do cluster) | Kubernetes registra automaticamente entradas `<svc>.<namespace>.svc.cluster.local` |
| DNS resolver (/etc/resolv.conf) | Pod DNS policy / CoreDNS | O resolv.conf de cada Pod aponta para o serviço DNS do cluster |
| socket (`ss -tlnp`) | `kubectl get endpoints` | Endpoints listam os pares IP:porta que sustentam cada Service |
| reverse proxy (nginx / haproxy) | Ingress / Gateway API | Balanceamento de carga L7 com roteamento por caminho e terminação TLS |
| network namespace | Pod network namespace | Todos os containers em um Pod compartilham um namespace de rede (um IP, um loopback) |
| bridge (`brctl`) | Plugin CNI bridge (ex.: Calico, Cilium) | O plugin CNI configura bridges/rotas virtuais para rede dos Pods |
| VLAN | NetworkPolicy + isolamento de namespace | Segmentação lógica de tráfego via seletores de labels e regras de política |
| traceroute | `kubectl debug` + traceroute dentro do Pod | Execute Pods de diagnóstico para rastrear caminhos de rede dentro do cluster |
| regras de firewall (ufw / iptables) | NetworkPolicy | Declare tráfego de ingress/egress permitido por Pod usando seletores de labels |

## Armazenamento

| Linux | Kubernetes | Notas |
|-------|-----------|-------|
| /etc/fstab | PersistentVolume (PV) + StorageClass | PVs definem armazenamento disponível; StorageClass permite provisionamento dinâmico |
| mount | volumeMounts (no Pod spec) | Anexa um volume em um caminho específico dentro do sistema de arquivos do container |
| block device (/dev/sda) | PersistentVolume (modo block) | `volumeMode: Block` expõe dispositivos de bloco brutos para os Pods |
| filesystem (ext4/xfs) | PersistentVolume (modo filesystem) | Modo padrão — o volume é formatado e montado como um diretório |
| LVM (Logical Volume Manager) | StorageClass + driver CSI | Drivers CSI gerenciam provisionamento, snapshots e redimensionamento dinamicamente |
| NFS mount | PersistentVolume (NFS) / driver CSI NFS | Tipo de volume `nfs` ou um driver CSI fornece acesso compartilhado ReadWriteMany |
| tmpfs | `emptyDir` com `medium: Memory` | Um volume efêmero em memória que desaparece quando o Pod termina |
| bind mount | Volume `hostPath` | Monta um diretório do host dentro do Pod — evite em produção |
| LVM snapshot | VolumeSnapshot | A API de snapshot CSI cria cópias point-in-time de PersistentVolumes |
| disk quota | ResourceQuota (storage) / `resources.limits.ephemeral-storage` | Limita armazenamento total de PVC por namespace ou disco efêmero por container |

## Configuração

| Linux | Kubernetes | Notas |
|-------|-----------|-------|
| Arquivos de config em /etc/ | ConfigMap | Pares chave-valor ou arquivos inteiros montados em Pods como volumes ou variáveis de ambiente |
| Variáveis de ambiente | `env` / `envFrom` no Pod spec | Define variáveis diretamente ou injeta todas as chaves de um ConfigMap/Secret |
| /etc/profile.d/ (scripts de inicialização do shell) | Init containers | Executa lógica de setup antes do container principal da aplicação iniciar |
| Recarga de arquivo de config (SIGHUP) | Rolling update / recarga de ConfigMap | Altere o ConfigMap e reinicie os Pods (ou use um sidecar reloader) |
| Sistema alternatives (`update-alternatives`) | Label selectors em Services | Roteia tráfego para diferentes versões de Pods atualizando seletores de labels |
| symlinks | Projected volumes | Combina múltiplos ConfigMaps/Secrets/tokens de ServiceAccount em uma única montagem |

## Segurança

| Linux | Kubernetes | Notas |
|-------|-----------|-------|
| users / groups (UID/GID) | `runAsUser` / `runAsGroup` (SecurityContext) | Define o UID/GID sob o qual o processo do container executa |
| chmod / chown | `fsGroup` no SecurityContext | Kubernetes define a propriedade de grupo em volumes montados automaticamente |
| /etc/passwd, /etc/shadow | ServiceAccount + RBAC | Identidade no Kubernetes é um ServiceAccount; permissões usam Role/ClusterRole |
| sudoers | ClusterRoleBinding (cluster-admin) | Concede privilégios elevados — equivalente a root em todo o cluster |
| Linux capabilities (`capsh`) | `securityContext.capabilities.add/drop` | Adiciona ou remove capabilities específicas do kernel (ex.: NET_ADMIN, SYS_TIME) |
| SELinux / AppArmor | Pod Security Admission / securityContext | Aplica perfis via `seLinuxOptions` ou `appArmorProfile` no Pod spec |
| firewall (ufw / iptables) | NetworkPolicy | Restringe tráfego Pod-a-Pod em L3/L4 com regras baseadas em labels |
| SSH keys (authorized_keys) | Certificados kubeconfig / tokens OIDC | Autentica no API server via certificados de cliente ou provedores de identidade |
| Certificate authority (CA) | Cluster CA / cert-manager | A CA do cluster assina certificados do kubelet e API server; cert-manager automatiza TLS |

## Monitoramento e Logging

| Linux | Kubernetes | Notas |
|-------|-----------|-------|
| top / htop | `kubectl top pods` / `kubectl top nodes` | Requer Metrics Server; mostra uso de CPU/memória por Pod ou nó |
| vmstat / iostat / sar | Prometheus + node-exporter | Métricas detalhadas no nível do host coletadas e armazenadas como dados de séries temporais |
| journalctl | `kubectl logs` | Exibe stdout/stderr dos containers; adicione `-f` para seguir e `-p` para anterior |
| syslog (/var/log/syslog) | Logging em nível de cluster (Fluent Bit / Fluentd) | Agrega logs de todos os Pods e envia para um backend central |
| dmesg (buffer de anel do kernel) | Logs no nível do nó / `kubectl debug node/` | Inspeciona mensagens do kernel fazendo debug diretamente no nó |
| /proc filesystem | Metrics Server / cAdvisor | cAdvisor expõe métricas de recursos por container do kernel |
| strace (rastreamento de syscalls) | `kubectl debug` com container efêmero | Anexa um container de debug com strace/tcpdump a um Pod em execução |
| lsof (arquivos/sockets abertos) | `kubectl exec` + lsof dentro do container | Executa comandos de diagnóstico dentro de um container em execução |
| df -h (uso de disco) | `kubectl exec -- df -h` / métricas de PVC | Verifica uso do sistema de arquivos dentro de containers ou monitora utilização de PVC |

## Gerenciamento de Pacotes e Implantação

| Linux | Kubernetes | Notas |
|-------|-----------|-------|
| apt / yum / dnf | Helm | O gerenciador de pacotes de fato para Kubernetes — instala charts (conjuntos de manifests) |
| .deb / .rpm (arquivo de pacote) | Helm chart / Kustomize overlay | Um conjunto empacotado e versionado de manifests Kubernetes |
| Repositório de pacotes (apt repo / yum repo) | Helm repository / registro OCI | Armazena e serve versões de charts (ex.: Artifact Hub, registros de container) |
| dpkg-reconfigure | `helm upgrade --reuse-values` | Reconfigura um release implantado com novos valores sem reinstalar |
| Ansible / Puppet (gerenciamento de config) | GitOps (Argo CD / Flux) | Entrega contínua declarativa orientada por Git que reconcilia o estado do cluster |
| git hooks (pre-commit, post-receive) | Admission webhooks (validating / mutating) | Intercepta requisições da API para aplicar políticas ou injetar sidecars |

## Administração de Sistema

| Linux | Kubernetes | Notas |
|-------|-----------|-------|
| hostname | Nome do Node / nome do Pod | Cada nó e Pod tem um nome resolvível por DNS dentro do cluster |
| reboot | `kubectl drain` + reinício do nó | Drain remove Pods graciosamente antes de reiniciar ou substituir um nó |
| modo de manutenção | `kubectl cordon` | Marca um nó como não-agendável — Pods existentes continuam executando |
| backup (rsync / tar) | Velero / snapshot do etcd | Velero faz backup de recursos e dados de PV; snapshots do etcd preservam o estado do cluster |
| crontab (agendamento em nível de usuário) | CronJob | Agenda cargas de trabalho recorrentes com expressões cron familiares |
| logrotate | Rotação de logs no container runtime | containerd/kubelet gerenciam rotação de arquivos de log; configure via flags do kubelet |
| systemd timers | CronJob (para dentro do cluster) / triggers externos | CronJobs substituem systemd timers para tarefas agendadas dentro do cluster |

---

*Navegação: [← Capítulo 15: Próximos Passos](15-next-steps.md) | [Apêndice B: Referência Rápida do kubectl →](appendix-b-cheatsheet.md)*
