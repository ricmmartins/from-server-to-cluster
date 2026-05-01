# Do Servidor ao Cluster

**Kubernetes para Profissionais Linux**

*Por [Ricardo Martins](https://ricardomartins.com.br)*

---

Um livro prático para profissionais Linux em transição para Kubernetes. Se você gerencia servidores, entende processos e pensa em termos de units do systemd e regras de iptables — este livro traduz tudo o que você já sabe para o mundo Kubernetes.

## Para Quem É Este Livro

Você é um sysadmin Linux, engenheiro de cloud ou profissional DevOps que:
- Gerencia servidores Linux diariamente (ou já gerenciou)
- Entende processos, sistemas de arquivos, rede e permissões
- Quer aprender Kubernetes sem começar do zero
- Prefere entender o *porquê* das coisas funcionarem, não apenas o *como*

**Pré-requisito:** Fundamentos sólidos de Linux. Se precisar revisar, comece pelo [Linux Hackathon](https://linuxhackathon.com).

## O Que Torna Este Livro Diferente

A maioria dos livros de Kubernetes assume que você está começando do zero. Este assume que você *já é bom em algo* — Linux — e constrói sobre essa base.

Cada capítulo começa com o que você já sabe, mapeia para o Kubernetes e honestamente indica onde a analogia falha. Você encontrará histórias reais de produção, decisões de arquitetura explicadas e laboratórios de diagnóstico (não apenas tutoriais) que constroem o pensamento sistêmico.

**Este não é um hackathon baseado em desafios** (esse é o [k8shackathon.com](https://k8shackathon.com)). Este livro é a leitura complementar — o *porquê* por trás do *quê*.

## Capítulos

| # | Capítulo | O Que Você Vai Aprender |
|---|----------|-------------------------|
| 01 | [Do Servidor ao Cluster](chapters/01-from-server-to-cluster.md) | Por que o Kubernetes existe, o que profissionais Linux já sabem, o caminho de aprendizado |
| 02 | [Containers Desmistificados](chapters/02-containers-demystified.md) | Linux namespaces, cgroups, imagens, Dockerfiles, runtimes de containers |
| 03 | [Arquitetura do Kubernetes](chapters/03-kubernetes-architecture.md) | Control plane, nodes, etcd, API server — pela perspectiva Linux |
| 04 | [Como o Kubernetes Pensa](chapters/04-how-kubernetes-thinks.md) | Estado desejado, loops de reconciliação, controllers, labels, selectors, scheduling |
| 05 | [Seu Primeiro Cluster](chapters/05-your-first-cluster.md) | Configuração do Kind, kubectl, kubeconfig, explorando componentes do cluster |
| 06 | [Pods e Workloads](chapters/06-pods-and-workloads.md) | Pods, Deployments, ReplicaSets, DaemonSets, StatefulSets, Jobs, Probes |
| 07 | [Rede](chapters/07-networking.md) | Services, Ingress, Gateway API, DNS — do iptables ao kube-proxy |
| 08 | [Armazenamento e Persistência](chapters/08-storage-and-persistence.md) | Volumes, PV, PVC, StorageClass, drivers CSI |
| 09 | [Configuração e Secrets](chapters/09-configuration-and-secrets.md) | ConfigMaps, Secrets, variáveis de ambiente — como aplicações consomem configuração |
| 10 | [Empacotamento e Entrega](chapters/10-packaging-and-delivery.md) | Helm, Kustomize, introdução ao GitOps |
| 11 | [Segurança](chapters/11-security.md) | RBAC, Pod Security Admission, NetworkPolicy, proteção de Secrets, criptografia em repouso |
| 12 | [Escalabilidade e Observabilidade](chapters/12-scaling-and-observability.md) | HPA, VPA, gerenciamento de recursos, classes de QoS, Prometheus, Grafana |
| 13 | [Troubleshooting](chapters/13-troubleshooting.md) | 5 cenários reais: sintomas, diagnóstico, correção — kubectl debug, events, logs |
| 14 | [Operações em Produção](chapters/14-production-operations.md) | Upgrades de cluster, backup/restore do etcd, recuperação de desastres, CRDs e Operators |
| 15 | [Próximos Passos](chapters/15-next-steps.md) | Certificações CKA, CKAD, CKS, KCNA, caminhos de carreira, comunidade |

### Apêndices

| | Apêndice | Conteúdo |
|---|----------|----------|
| A | [Glossário Linux-para-Kubernetes](chapters/appendix-a-glossary.md) | systemctl para kubectl, iptables para NetworkPolicy e mais de 50 mapeamentos |
| B | [Cheatsheet do kubectl](chapters/appendix-b-cheatsheet.md) | Comandos essenciais e padrões YAML |
| C | [Comparação de Plataformas Cloud](chapters/appendix-c-cloud-platforms.md) | AKS vs EKS vs GKE — configuração, preços, funcionalidades |

## O Ecossistema de Aprendizado

Este livro faz parte de um caminho progressivo de aprendizado:

```
linuxhackathon.com          k8shackathon.com           ai4infra.com
Fundamentos Linux     --->  Kubernetes Hackathon  ---> IA para Infraestrutura
(20 desafios)               (20 desafios)              (IA + Cloud)
        \                        |                      /
         \                       |                     /
          `----> Este Livro <---'                    /
           "Do Servidor ao Cluster"                 /
            (O PORQUÊ por trás do QUÊ)  -----------'
```

| Recurso | Formato | Foco |
|---------|---------|------|
| [Linux Hackathon](https://linuxhackathon.com) | Desafios práticos | Fundamentos Linux (20 desafios) |
| [**Este Livro**](https://fromservertocluster.com) | Narrativa + laboratórios | Ponte conceitual: Linux para Kubernetes |
| [K8s Hackathon](https://k8shackathon.com) | Desafios práticos | Prática com Kubernetes (20 desafios) |
| [AI for Infra](https://ai4infra.com) | Ebook | IA aplicada à infraestrutura |

## Como Usar Este Livro

1. **Leia sequencialmente** na primeira vez — os capítulos se constroem uns sobre os outros
2. **Execute os laboratórios** — cada capítulo tem um laboratório de diagnóstico usando [Kind](https://kind.sigs.k8s.io/)
3. **Consulte as tabelas de comparação** — referência rápida para mapeamento de conceitos Linux-para-K8s
4. **Leia os quadros "Onde a Analogia Falha"** — é aqui que o entendimento real acontece
5. **Depois faça o hackathon** — aplique o que aprendeu em [k8shackathon.com](https://k8shackathon.com)

## Requisitos Técnicos

- Uma máquina Linux (física, VM ou WSL2)
- [Docker](https://docs.docker.com/get-docker/) instalado
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) instalado
- [kubectl](https://kubernetes.io/docs/tasks/tools/) instalado
- [Helm](https://helm.sh/docs/intro/install/) (a partir do Capítulo 10)
- 8 GB de RAM mínimo (16 GB recomendado)

Todos os laboratórios foram testados no **Kubernetes v1.32** com Kind. Exemplos específicos de cloud usam AKS, EKS e GKE como variantes.

## Conteúdo em Inglês

Procurando a versão em inglês? Visite a [branch principal](https://github.com/ricmmartins/from-server-to-cluster).

## Sobre o Autor

**Ricardo Martins** é Principal Solutions Engineer na Microsoft com profunda expertise em Linux, infraestrutura cloud e Kubernetes. É autor do [AI for Infrastructure Professionals](https://ai4infra.com), criador do [Linux Hackathon](https://linuxhackathon.com) e do [Kubernetes Hackathon](https://k8shackathon.com), e vem ajudando profissionais de infraestrutura a evoluir suas carreiras há mais de uma década.

- Blog (EN): [rmmartins.com](https://rmmartins.com)
- Blog (PT-BR): [ricardomartins.com.br](https://ricardomartins.com.br)
- GitHub: [github.com/ricmmartins](https://github.com/ricmmartins)
- LinkedIn: [linkedin.com/in/yourprofile](https://www.linkedin.com/in/yourprofile)

## Licença

Esta obra está licenciada sob [Creative Commons Atribuição-NãoComercial-CompartilhaIgual 4.0 Internacional (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/).

Você é livre para compartilhar e adaptar este material para fins não comerciais, desde que dê os devidos créditos e distribua suas contribuições sob a mesma licença.

---

*"Você não precisa esquecer tudo o que sabe sobre Linux para aprender Kubernetes. Você precisa ver como ele evolui."*
