# Technical correction record

Review date: October 15, 2026. Source language: `en-us`.

## AI Council decision

The review identified 42 substantive findings, not a need to rewrite the book.
The council considered keeping the historical 1.32 baseline, moving to the newest
release, and selecting a maintained, explicitly pinned teaching baseline.

- **Contrarian:** replacing every component with its latest release increases the
  compatibility surface. Keeping an unsupported baseline also carries a cost.
  Version pinning must cover clients and add-ons, not just the API server.
- **First principles:** the goal is to teach observable mechanisms reproducibly.
  A lab snapshot and a production recommendation serve different purposes.
  Procedures must state their prerequisites and success conditions.
- **Expansionist:** one shared version contract reduces future translation drift.
  Replacing retired dependencies improves the useful lifetime of exercises.
  Whole-book checks should complement the individual corrections.
- **Outsider:** an operational manual is a maintenance procedure, not just syntax.
  Accepted YAML does not establish recovery, isolation, or availability.
  Those guarantees need explicit checks and limitations.
- **Executor:** use a maintained Kubernetes minor with an official Kind image.
  Give parallel editors non-overlapping file ownership and a shared matrix.
  Record actual validation separately from tests still requiring infrastructure.

**Chairman's decision:** use Kubernetes 1.35.0 as a pinned teaching snapshot with
Kind 0.31.0 and a matching kubectl, replace retired lab dependencies, and retain
the author's practical structure. Production guidance instead requires a current
supported patch and a provider/add-on compatibility check.

**Biggest risk:** presenting static checks as evidence that a lab works in a
running cluster.

**First step:** verify the official Kind image, establish the shared version
contract, then make file-scoped corrections in parallel.

The [official Kind 0.31.0 default](https://raw.githubusercontent.com/kubernetes-sigs/kind/v0.31.0/pkg/apis/config/defaults/image.go)
provides the pinned node image. See also the Kubernetes
[version-skew policy](https://kubernetes.io/releases/version-skew-policy/) and
[release support calendar](https://kubernetes.io/releases/patch-releases/).

## Implementation choices

Use small corrections rather than chapter rewrites. Keep stateful demonstrations
honest about database replication, node-local storage, and data lifecycle.
Separate API-object backups from physical etcd recovery. Scope single-member
recovery examples explicitly instead of implying they are HA procedures.
Do not imply that a free management tier includes worker resources or a paid SLA.
Where cloud creation depends on an account, region, or quota, use discovery and
state that availability has not been established by documentation alone.

The council's minimal-change and reproducibility criteria also determine these
implementation choices:

| Choice | Decision and trade-off |
|---|---|
| Ingress lab | Keep the Ingress learning objective, using an explicit Traefik `v3.7.15` manifest and a dedicated Kind topology. This avoids a Helm prerequisite before chapter 10. A wholesale Gateway API migration would change the exercise's objective; Gateway API remains the forward-looking alternative. |
| Helm lab | Use the book's self-contained `web-demo` chart rather than depend on the changed Bitnami catalog. This makes templates and image choices visible; it is not a claim that every Bitnami image fails to pull. |
| Application images | Pin images per exercise rather than indiscriminately replace every nginx tag. The new delivery lab uses its own documented image; a pin is a teaching snapshot, not proof of current vulnerability status. |
| Cloud examples | Discover supported versions before creation rather than assume that the local `.0` patch exists in every managed service. No regional creation success or account-specific quote is asserted. |

### Pinned lab components

| Component | Version | Scope |
|---|---|---|
| Kind | 0.31.0 | Local cluster creation |
| Kubernetes node image | 1.35.0, with the SHA256 digest in README | Disposable local clusters |
| kubectl | 1.35.0 | Client and bundled Kustomize 5.7.1 |
| Helm | 3.19.0 | Packaging and monitoring exercises |
| Traefik | 3.7.15 | Chapter 7; Appendix B refers to its class |
| Calico | 3.31.4 | Chapter 11 NetworkPolicy lab; upstream matrix includes Kubernetes 1.35 |
| Metrics Server | 0.8.1 | Chapter 12; upstream matrix supports Kubernetes 1.31+ |
| kube-prometheus-stack chart | 92.9.0 | Chapter 12 optional ephemeral monitoring lab |
| Local web-demo chart | 1.0.0 | Chapter 10; image nginx:1.30.6-alpine |

These are explicit inputs, not a claim of end-to-end runtime certification.
Changing them requires repeating the compatibility review and lab tests.

### Evidence and remaining uncertainty

- **Confirmed from official documentation:** AKS 1.32 left community support in
  March 2026; LTS enrollment is explicit, not automatic. The
  [support calendar](https://learn.microsoft.com/en-us/azure/aks/supported-kubernetes-versions)
  and [LTS requirements](https://learn.microsoft.com/en-us/azure/aks/long-term-support)
  establish lifecycle and enrollment, not whether a particular subscription can
  create that version today. Regional availability, quotas, identity permissions,
  and actual creation success remain **unverified**; Appendix C requires discovery.
- **Confirmed lifecycle and list-price conditions:** EKS 1.32 entered extended
  support on March 23, 2026; its support tier changes the cluster-hour fee.
  See the [EKS calendar](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html)
  and [pricing](https://aws.amazon.com/eks/pricing/). A 730-hour illustration is
  not a fixed monthly invoice or a quote for all resources.
- **Potential failure, not reproduced:** the Bitnami catalog migration was
  documented by the vendor, but the review did not demonstrate a particular
  image pull failing. The delivery exercises now avoid that dependency; they do
  not claim that all Bitnami images are unavailable.
- **Not established by this review:** image vulnerability status, authenticated
  cloud deployment, account-specific costs, throughput benchmarks, successful
  recovery under failure, and the authenticity of historical production stories.

## Finding index

Numbers match the consolidated review. This is the correction scope; validation
below distinguishes static evidence from unexecuted operational tests.

| Finding | Correction | Location |
|---|---|---|
| 1 | Pin compatible lab components; distinguish supported minor and teaching patch | README; chapters 1, 11, 12; cluster commands throughout |
| 2 | Force deletion removes an API object, not proof of process death | Appendix A |
| 3 | Delimit single-member etcd restore and explain HA/revision requirements | Chapter 14 |
| 4 | Complete per-node kubeadm upgrade prerequisites and sequence | Chapter 14 |
| 5 | Drain completion is not replacement readiness; assess local data | Chapter 14 |
| 6 | Velero API backups are not physical etcd snapshots | Chapter 14 |
| 7 | Rewrite existing Secrets after enabling encryption; protect recovery keys | Chapter 11 |
| 8 | User namespaces require explicit enablement in rootful Docker | Chapter 2 |
| 9 | Supply PostgreSQL initialization prerequisites; replication is separate | Chapter 6 |
| 10 | Replace retired ingress-nginx dependency | Chapter 7; Appendix B |
| 11 | Align Kind port mapping, node placement, and Ingress controller | Chapter 7 |
| 12 | Scope hostPath persistence to the node that actually holds the data | Chapter 8 |
| 13 | Separate AKS Free from paid SLA tiers | Appendix C |
| 14 | Discover regional AKS versions; LTS is explicit and paid | Appendix C |
| 15 | Separate EKS standard and extended support pricing | Appendix C |
| 16 | Count GKE regional nodes per zone, not per cluster | Appendix C |
| 17 | Remove App Mesh as a current recommendation | Appendix C |
| 18 | Docker writable layers survive stop/start of the same container | Chapter 2 |
| 19 | A Docker PID belongs to the daemon host, not necessarily the client | Chapter 2 |
| 20 | API access is not the only communication channel | Chapter 3 |
| 21 | Cloud volume provisioning belongs to CSI integration | Chapter 3 |
| 22 | Account for node detection, tolerations, eviction, and capacity | Chapter 4 |
| 23 | Separate requests, runtime weighting, limits, and Pod priority | Chapter 4; Appendix A |
| 24 | Avoid assuming the nginx image includes ps | Chapter 5 |
| 25 | Native sidecars became GA in 1.33 | Chapter 6 |
| 26 | PDBs do not govern workload-controller rollouts or direct deletes | Chapter 6 |
| 27 | Include nftables and qualify kube-proxy mode/platform descriptions | Chapter 7 |
| 28 | Do not overwrite the marker used to demonstrate persistence | Chapter 8 |
| 29 | Use projected ServiceAccount tokens rather than assuming auto-created Secrets | Chapter 9 |
| 30 | Distinguish Secret-object authorization from per-key authorization | Chapter 9 |
| 31 | GitOps self-healing and pruning depend on enabled policies | Chapter 10 |
| 32 | Remove dependence on the changed Bitnami image catalog | Chapter 10 |
| 33 | Supply an application compatible with the hardened security context | Chapter 11 |
| 34 | Label ephemeral monitoring as a demo; document production storage/retention | Chapter 12 |
| 35 | Explain actual node-pressure eviction ranking | Chapter 12 |
| 36 | Working set is not the cgroup OOM limit counter | Chapter 12 |
| 37 | Empty endpoints require readiness and selector checks | Chapter 13 |
| 38 | preStop consumes the termination grace period | Appendix A |
| 39 | NetworkPolicy acceptance is not enforcement | Appendix B |
| 40 | Separate GKE management credit, workload charges, and promotional credit | Chapter 15 |
| 41 | Use valid lowercase GKE names consistently | Appendix C |
| 42 | Do not assume gp3 exists or is the default StorageClass | Appendix C |

## Validation and limitations

The correction pass used four parallel editors with exclusive chapter ownership,
followed by whole-book static checks. All 18 chapter/appendix files, README, and
this record were included in the consistency scan.

| Check actually performed | Result |
|---|---|
| Markdown extraction | 82 YAML fences and 54 resource-bearing shell heredocs examined; unrendered Helm templates were excluded from raw-YAML checks and rendered separately |
| kubeconform 0.8.0, strict, Kubernetes 1.35.0 | 100 native resource occurrences valid; two CRDs lacked published kubeconform schemas and were checked using official OpenAPI instead |
| Official Kubernetes OpenAPI 1.35.0 and 1.37.0 | All 102 native resource occurrences, including the two CRDs, passed; `IntOrString` respected and unknown fields checked |
| Custom-resource examples | Four instances passed the schemas of the two CRDs taught in the book |
| kubectl 1.35.0 client dry-run | Nine literal Deployment generator commands passed with `--dry-run=client --validate=false`, explicitly targeting an unused loopback endpoint; output redirections were not executed |
| Bash and ShellCheck 0.11.0 | All 380 Bash/sh blocks passed `bash -n` and ShellCheck at **error severity**, substituting documentation placeholders for syntax analysis; this is not a warning-free lint claim |
| Helm 3.19.0 | Strict lint passed for the local chart with default and upgrade values; install/upgrade manifests rendered |
| Kustomize bundled with kubectl 1.35.0 | Base, dev, prod, and Helm-plus-Kustomize rendered; replicas, selectors, labels, and annotations checked; deprecated `commonLabels` replaced without changing rendered semantics |
| Rendered local Helm/Kustomize YAML | 12 resource occurrences passed strict kubeconform |
| Pinned upstream add-ons | Metrics Server 0.8.1 and Calico 3.31.4 downloaded; kube-prometheus-stack 92.9.0 rendered with the documented values and Kubernetes 1.35.0 |
| Upstream native/custom schemas | Strict kubeconform covered available native schemas; supplementary official OpenAPI/embedded CRD schemas checked 136 Prometheus-stack resources, nine Metrics Server resources, and 39 Calico resources without remaining uncovered kinds |
| Consistency | Local Markdown file destinations and anchors resolved; all Kind creation commands pinned; no moving installation URLs found by the command scan; historical 1.32 references retained only with explicit context |
| Git whitespace | `git diff --check` passed |

For the supplementary upstream OpenAPI check, optional native fields serialized
as `null` were treated as absent (five in the chart output, six in the Calico
manifest), and `IntOrString` was normalized according to its Kubernetes meaning.
The original manifests also passed kubeconform wherever its schemas were
available. These checks do not execute CEL validations, admission webhooks,
controller reconciliation, or API-server defaulting.

The client dry-run of a file-based manifest was **attempted and blocked** because
kubectl requires API discovery even with client dry-run and validation disabled.
It is not counted as a successful manifest dry-run. The nine generator checks
above are offline client checks, not server-side validation.

**Not performed:** end-to-end labs, image pulls, scheduling/connectivity tests,
NetworkPolicy enforcement tests, HPA/load tests, production upgrade/restore,
failure injection, or authenticated cloud discovery/creation. The Docker client
was present but its Linux daemon was unavailable; no existing production cluster
was used. `terraform validate` was **not applicable**: the repository contains no
Terraform configurations. Schema success is not a runtime certification.

Helm emitted only an optional chart-icon recommendation; no icon-only change was
made. The validation helper environment emitted a `jsonschema.RefResolver`
deprecation warning; that helper-library warning is not a Kubernetes API
deprecation or a failed resource check.

## Translation policy

Merge the English correction PR first. Then port the same corrections to
`pt-br` in natural Brazilian Portuguese, preserving executable code exactly and
repeating the checks. Keep executable fenced blocks, including their comments,
identical to the merged English source; translate the surrounding explanations
naturally. Use a separate PR targeting each language branch. Do not merge English
chapter text wholesale into the translated branch.
