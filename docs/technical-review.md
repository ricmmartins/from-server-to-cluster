# Registro de correções técnicas

Data da revisão: 15 de outubro de 2026. Fonte: branch `en-us`, merge
`ba7afe394b8c213ef3a98989ab2d44ae90a9054b`, contendo a correção
`ee88d2c8faaee236b1d85afccdcf549b2e6ff30b` do [PR inglês #1](https://github.com/ricmmartins/from-server-to-cluster/pull/1).
Este porte compara a fonte final com `14956eed831f1d42d2027b34783918cc16c388a2`.

## Decisão do AI Council

A revisão identificou 42 achados substantivos, não a necessidade de reescrever
o livro. O conselho da revisão original considerou manter a base histórica 1.32,
adotar a versão mais nova ou escolher uma base de ensino mantida e explicitamente fixada.
Esta tradução preserva a decisão já aprovada; não houve um novo conselho.

- **Contraponto:** trocar todos os componentes pela versão mais recente amplia
  a superfície de compatibilidade. Manter uma base sem suporte também tem custo.
  O versionamento precisa incluir clientes e add-ons, não só o API server.
- **Primeiros princípios:** o objetivo é ensinar mecanismos observáveis de forma
  reproduzível. Um recorte de laboratório não é uma recomendação de produção.
  Procedimentos precisam declarar pré-requisitos e condições de sucesso.
- **Expansão:** um contrato comum de versões reduz divergências futuras entre
  traduções. Substituir dependências descontinuadas prolonga a utilidade dos
  exercícios. Verificações do livro inteiro complementam correções individuais.
- **Olhar externo:** um manual operacional descreve procedimentos de manutenção,
  não apenas sintaxe. YAML aceito não comprova recuperação, isolamento ou
  disponibilidade; essas garantias exigem verificações e limites explícitos.
- **Execução:** usar uma versão minor mantida do Kubernetes com imagem oficial
  do Kind, editores paralelos com arquivos exclusivos e uma matriz compartilhada.
  Registrar validação executada separadamente de testes que exigem infraestrutura.

**Decisão da presidência:** usar Kubernetes 1.35.0 como recorte fixo para ensino,
Kind 0.31.0 e kubectl correspondente, substituir dependências descontinuadas e
manter a estrutura prática do autor. Para produção, usar um patch suportado atual
e verificar a compatibilidade do provedor e dos add-ons.

**Maior risco:** apresentar verificações estáticas como prova de que um
laboratório funciona em um cluster em execução.

**Primeiro passo da revisão original:** conferir a imagem oficial do Kind,
definir o contrato comum de versões e corrigir arquivos em paralelo.

A [configuração padrão oficial do Kind 0.31.0](https://raw.githubusercontent.com/kubernetes-sigs/kind/v0.31.0/pkg/apis/config/defaults/image.go)
fornece a imagem fixada. Consulte também a
[política de diferença de versões](https://kubernetes.io/releases/version-skew-policy/)
e o [calendário de suporte](https://kubernetes.io/releases/patch-releases/).

## Escolhas de implementação

Fazer correções pontuais, não reescrever capítulos. Explicar honestamente
replicação de banco de dados, armazenamento local ao node e ciclo de vida dos
dados nos exemplos com estado. Separar backups de objetos da API da recuperação
física do etcd. Delimitar a restauração de um único membro, sem apresentá-la como
procedimento de alta disponibilidade. Não sugerir que uma camada gratuita de
gerenciamento inclui workers ou SLA pago. Quando a criação cloud depender de
conta, região ou quota, consultar a disponibilidade e deixar claro que a
documentação, por si só, não comprova a criação.

| Escolha | Decisão e contrapartida |
|---|---|
| Laboratório de Ingress | Manter o objetivo de aprendizado de Ingress com um manifesto explícito do Traefik `v3.7.15` e topologia Kind dedicada, sem exigir Helm antes do capítulo 10. Migrar todo o exercício para Gateway API mudaria seu objetivo; Gateway API continua como alternativa voltada ao futuro. |
| Laboratório de Helm | Usar o chart autocontido `web-demo` do livro, sem depender do catálogo alterado da Bitnami. Templates e imagens ficam visíveis; isso não afirma que toda imagem Bitnami falha ao ser baixada. |
| Imagens das aplicações | Fixar imagens por exercício, em vez de trocar indiscriminadamente todas as tags nginx. O novo laboratório de entrega tem imagem documentada; fixar uma versão não comprova seu estado atual de vulnerabilidades. |
| Exemplos cloud | Consultar versões suportadas antes da criação, sem presumir que o patch `.0` local exista em todo serviço gerenciado. Não se afirma sucesso de criação regional nem cotação específica de uma conta. |

### Componentes fixados para os laboratórios

| Componente | Versão | Escopo |
|---|---|---|
| Kind | 0.31.0 | Criação de clusters locais |
| Imagem do node Kubernetes | 1.35.0, com digest SHA256 no README | Clusters locais descartáveis |
| kubectl | 1.35.0 | Cliente e Kustomize 5.7.1 incluído |
| Helm | 3.19.0 | Exercícios de empacotamento e monitoramento |
| Traefik | 3.7.15 | Capítulo 7; apêndice B referencia sua classe |
| Calico | 3.31.4 | NetworkPolicy no capítulo 11; matriz upstream inclui Kubernetes 1.35 |
| Metrics Server | 0.8.1 | Capítulo 12; matriz upstream suporta Kubernetes 1.31+ |
| Chart kube-prometheus-stack | 92.9.0 | Laboratório opcional de monitoramento efêmero no capítulo 12 |
| Chart local web-demo | 1.0.0 | Capítulo 10; imagem nginx:1.30.6-alpine |

São entradas explícitas, não uma certificação de runtime de ponta a ponta.
Alterá-las exige repetir a revisão de compatibilidade e os testes dos laboratórios.

### Evidências da revisão original e incertezas

- **Confirmado na documentação oficial pela revisão inglesa:** AKS 1.32 saiu do
  suporte comunitário em março de 2026; a adesão ao LTS é explícita, não automática.
  O [calendário](https://learn.microsoft.com/en-us/azure/aks/supported-kubernetes-versions)
  e os [requisitos de LTS](https://learn.microsoft.com/en-us/azure/aks/long-term-support)
  estabelecem ciclo de vida e adesão, não a capacidade de uma assinatura criar essa
  versão hoje. Disponibilidade regional, quotas, permissões de identidade e sucesso
  de criação continuam **não verificados**; o apêndice C exige consulta.
- **Ciclo de vida e condições de preço de tabela registrados na fonte:** EKS 1.32
  entrou em suporte estendido em 23 de março de 2026; a modalidade muda a tarifa
  por hora de cluster. Veja o [calendário EKS](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html)
  e os [preços](https://aws.amazon.com/eks/pricing/). A ilustração de 730 horas não
  é fatura mensal fixa nem cotação de todos os recursos.
- **Falha possível, não reproduzida:** o fornecedor documentou a migração do
  catálogo Bitnami, mas a revisão não demonstrou falha de pull de uma imagem
  específica. Os exercícios deixam de depender do catálogo; não afirmam que todas
  as imagens Bitnami estejam indisponíveis.
- **Não estabelecido:** estado de vulnerabilidades das imagens, implantação cloud
  autenticada, custos específicos de conta, benchmarks, recuperação sob falha e
  autenticidade das histórias de produção.

## Índice dos achados

A numeração corresponde à revisão consolidada original. A validação abaixo
distingue evidência estática de testes operacionais não executados.

| Achado | Correção | Local |
|---|---|---|
| 1 | Fixar componentes compatíveis; distinguir minor suportada de patch para ensino | README; capítulos 1, 11, 12; comandos de cluster |
| 2 | Exclusão forçada remove o objeto da API, não comprova morte do processo | Apêndice A |
| 3 | Delimitar restauração etcd de um membro e explicar requisitos de HA e revisão | Capítulo 14 |
| 4 | Completar pré-requisitos e sequência de upgrade kubeadm por node | Capítulo 14 |
| 5 | Drain concluído não garante substitutos prontos; avaliar dados locais | Capítulo 14 |
| 6 | Backups de API do Velero não são snapshots físicos do etcd | Capítulo 14 |
| 7 | Regravar Secrets existentes após ativar criptografia; proteger chaves de recuperação | Capítulo 11 |
| 8 | User namespaces exigem ativação explícita no Docker rootful | Capítulo 2 |
| 9 | Informar pré-requisitos de inicialização PostgreSQL; replicação é separada | Capítulo 6 |
| 10 | Substituir dependência descontinuada do ingress-nginx | Capítulo 7; apêndice B |
| 11 | Alinhar portas Kind, posicionamento do node e controller Ingress | Capítulo 7 |
| 12 | Restringir persistência hostPath ao node que contém os dados | Capítulo 8 |
| 13 | Separar AKS Free de camadas com SLA pago | Apêndice C |
| 14 | Consultar versões regionais do AKS; LTS é explícito e pago | Apêndice C |
| 15 | Separar preços de suporte padrão e estendido no EKS | Apêndice C |
| 16 | Contar nodes regionais do GKE por zona, não por cluster | Apêndice C |
| 17 | Remover App Mesh das recomendações atuais | Apêndice C |
| 18 | Camada gravável Docker sobrevive a stop/start do mesmo container | Capítulo 2 |
| 19 | PID Docker pertence ao host do daemon, não necessariamente ao cliente | Capítulo 2 |
| 20 | Acesso à API não é o único canal de comunicação | Capítulo 3 |
| 21 | Provisionamento de volumes cloud pertence à integração CSI | Capítulo 3 |
| 22 | Considerar detecção do node, tolerations, eviction e capacidade | Capítulo 4 |
| 23 | Separar requests, pesos em runtime, limits e prioridade de Pod | Capítulo 4; apêndice A |
| 24 | Não presumir que a imagem nginx inclua ps | Capítulo 5 |
| 25 | Sidecars nativos chegaram a GA na versão 1.33 | Capítulo 6 |
| 26 | PDBs não controlam rollouts de controllers de workload nem exclusões diretas | Capítulo 6 |
| 27 | Incluir nftables e qualificar modos e plataformas do kube-proxy | Capítulo 7 |
| 28 | Não sobrescrever o marcador usado para demonstrar persistência | Capítulo 8 |
| 29 | Usar tokens projetados de ServiceAccount, sem presumir Secrets automáticos | Capítulo 9 |
| 30 | Distinguir autorização do objeto Secret de autorização por chave | Capítulo 9 |
| 31 | Autorrecuperação e pruning GitOps dependem das políticas habilitadas | Capítulo 10 |
| 32 | Remover dependência do catálogo alterado de imagens Bitnami | Capítulo 10 |
| 33 | Fornecer aplicação compatível com o security context reforçado | Capítulo 11 |
| 34 | Identificar monitoramento efêmero como demo; documentar armazenamento e retenção em produção | Capítulo 12 |
| 35 | Explicar a ordem real de eviction por pressão no node | Capítulo 12 |
| 36 | Working set não é o contador usado pelo limite OOM do cgroup | Capítulo 12 |
| 37 | Endpoints vazios exigem verificar readiness e selectors | Capítulo 13 |
| 38 | preStop consome o período de encerramento gracioso | Apêndice A |
| 39 | Aceitação de NetworkPolicy não comprova aplicação | Apêndice B |
| 40 | Separar crédito de gerenciamento GKE, custos de workloads e crédito promocional | Capítulo 15 |
| 41 | Usar nomes GKE válidos em minúsculas de forma consistente | Apêndice C |
| 42 | Não presumir que gp3 exista ou seja a StorageClass padrão | Apêndice C |

## Validação do porte e limitações

O porte usa quatro editores paralelos com grupos exclusivos de capítulos e
consolidação de README e registro. As verificações abaixo foram executadas
nesta worktree, com auxiliares adaptados em artifacts próprios; nenhum resultado
da revisão inglesa é apresentado como execução própria.

| Verificação executada neste porte | Resultado |
|---|---|
| Fonte e base | `fetch origin` e ancestralidade confirmaram o merge `ba7afe3` e a correção `ee88d2c` em `origin/en-us`; a branch isolada parte de `pt-br` |
| Paridade EN/PT | 465 blocos executáveis idênticos, inclusive fences e comentários: 379 Bash, um sh, 82 YAML e um de cada tipo Dockerfile, INI e nginx; comparação em LF canônico do Git |
| Diagramas e saídas | Conferência semântica dos blocos não executáveis pelos editores de cada grupo; preservadas as traduções permitidas, sem alterar fluxos, nomes técnicos ou resultados |
| Extração Markdown/YAML | 82 fences YAML e 54 heredocs com recursos examinados; templates Helm não renderizados excluídos da análise de YAML bruto e renderizados separadamente |
| kubeconform 0.8.0 estrito, Kubernetes 1.35.0 | 100 ocorrências de recursos nativos válidas; dois CRDs sem schema publicado nessa ferramenta foram cobertos pelo OpenAPI oficial |
| OpenAPI oficial Kubernetes 1.35.0 e 1.37.0 | As 102 ocorrências nativas, inclusive os dois CRDs, passaram em ambas as versões, com verificação de campos desconhecidos e tratamento de `IntOrString` |
| Exemplos de recursos customizados | Quatro instâncias passaram nos schemas dos dois CRDs ensinados no livro |
| kubectl 1.35.0 offline | Nove geradores literais de Deployment passaram com `--dry-run=client --validate=false`, endpoint loopback sem serviço e sem executar redirecionamentos |
| Bash e ShellCheck 0.11.0 | Os 380 blocos Bash/sh passaram em `bash -n` e ShellCheck com severidade **error**, substituindo placeholders da documentação para analisar sintaxe; não é uma alegação de ausência de warnings |
| Helm 3.19.0 | Lint estrito do chart local passou com valores padrão e de upgrade; ambos os manifests foram renderizados a partir dos heredocs portugueses |
| Kustomize 5.7.1 incluído no kubectl | Base, dev, prod e Helm com Kustomize renderizados; conferidos réplicas, selectors, labels e annotations |
| YAML local renderizado | As 12 ocorrências dos seis renderings Helm/Kustomize passaram no kubeconform estrito e OpenAPI oficial 1.35.0 |
| Add-ons upstream fixados | Metrics Server 0.8.1 e Calico 3.31.4 baixados das URLs presentes no PT; chart kube-prometheus-stack 92.9.0 renderizado para Kubernetes 1.35.0 com os values documentados no PT e CRDs incluídos |
| Schemas dos add-ons | OpenAPI oficial e schemas embarcados cobriram nove recursos Metrics Server, 39 Calico e 136 Prometheus (86 nativos e 50 customizados), sem erros nem tipos descobertos sem cobertura; kubeconform estrito aprovou os schemas disponíveis |
| Consistência localizada | Destinos e anchors Markdown locais resolvidos nos 20 arquivos; todos os comandos de criação Kind fixados, nenhuma URL móvel de instalação encontrada; menções históricas a 1.32 e dependências aposentadas mantidas apenas com contexto |
| Whitespace Git | `git diff --check` aprovado |

Na validação suplementar dos add-ons, campos nativos opcionais serializados como
`null` foram tratados como ausentes (cinco no chart e seis no Calico).
`IntOrString` foi normalizado conforme seu significado no Kubernetes. Os
manifests originais também passaram no kubeconform onde havia schemas
disponíveis: nove recursos Metrics Server, 15 Calico e 76 Prometheus; os demais
tipos foram cobertos pelo OpenAPI e pelos CRDs, não ignorados no resultado
consolidado. Essas verificações não executam CEL, admission webhooks,
reconciliação de controllers nem defaulting do API server.

Os testes com kubectl foram de geradores offline, não de manifests no servidor.
Não se reivindica dry-run bem-sucedido de arquivos: a descoberta de API exige
um servidor mesmo com dry-run de cliente e validação desativada. Na validação
dos scripts, o stdin foi enviado em UTF-8/LF para evitar que a conversão de
newline do Windows gerasse falsos erros de carriage return no ShellCheck.
O Git armazena os blocos com LF; `core.autocrlf` pode apresentá-los com CRLF na worktree.

O Helm emitiu apenas a recomendação opcional de ícone do chart; não houve
alteração cosmética para eliminá-la. O auxiliar Python emitiu aviso de
depreciação de `jsonschema.RefResolver`, não de uma API Kubernetes.

**Não executado:** laboratórios de ponta a ponta, pulls de imagens,
agendamento/conectividade, aplicação de NetworkPolicy, HPA/carga, upgrade ou
restore de produção, injeção de falhas e consulta/criação cloud autenticada.
O cliente Docker está presente, mas o daemon Linux está indisponível.
Nenhum cluster de produção é usado. Sem configurações Terraform no repositório,
`terraform validate` não se aplica. Aprovação de schema não é certificação de runtime.

## Política de tradução

O PR inglês foi integrado primeiro. Este porte separado, com destino a `pt-br`,
mantém a prosa em português brasileiro natural e os blocos cercados executáveis,
inclusive comentários, idênticos à fonte inglesa integrada. Diagramas e saídas
traduzíveis preservam a semântica. A tradução não incorpora capítulos inteiros
em inglês nem transfere alegações de validação sem repeti-las.
