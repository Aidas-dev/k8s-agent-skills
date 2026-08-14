---
name: tekton-operator
description: Tekton operator (tektoncd/operator) — install, upgrade, and manage Tekton components (Pipelines, Triggers, Dashboard, Chains, Results, PaC) via TektonConfig and component CRs. Covers profiles, pipeline config, tektonpruner, external results DB.
---

# tekton-operator — Install & Manage Tekton

For authoring pipelines see `tekton-pipelines`. For Pipelines-as-Code see `tekton-pac`.

**Repo:** github.com/tektoncd/operator  
**Latest:** v0.80.0 (2026-06-09). LTS: v0.78.x (pipeline v1.6.x LTS, EOL 2026-12-08), v0.77.x (EOL 2026-08-21). Min K8s 1.28.  
**API group:** `operator.tekton.dev/v1alpha1`

## CR Model

| CR | Manages |
|----|---------|
| `TektonConfig` | **Top-level** — installs/configures all components via profile |
| `TektonPipeline` | Pipeline component |
| `TektonTrigger` | Trigger component |
| `TektonDashboard` | Dashboard |
| `TektonChain` | Supply-chain security (SSDF/SLSA provenance) |
| `TektonResult` | Results API (pipeline run records) |
| `TektonPruner` | Event-based cleanup of old PipelineRuns/TaskRuns |
| `OpenShiftPipelinesAsCode` | Pipelines-as-Code (K8s since v0.80.0 via `spec.platforms.kubernetes.pipelinesAsCode`) |
| `TektonAddon` | Addons (OpenShift: templates, resolver tasks) |
| `TektonScheduler` | Multi-cluster scheduling (requires Kueue + cert-manager) |

**Install TektonConfig, get everything.** The user only needs to create `TektonConfig` with the desired profile — the operator creates the component CRs and manages their lifecycle. `TektonResult` and `TektonChain` are optional components NOT installed through TektonConfig currently.

## Install

```bash
# 1. Operator itself (also installs pipelines/triggers/chains/dashboard via profile CRs below)
kubectl apply -f https://infra.tekton.dev/tekton-releases/operator/latest/release.yaml

# 2. Choose profile — installs the components
kubectl apply -f https://raw.githubusercontent.com/tektoncd/operator/main/config/crs/kubernetes/config/all/operator_v1alpha1_config_cr.yaml
```

### Profiles

| Profile | Installs |
|---------|----------|
| `lite` | Pipeline |
| `basic` | Pipeline, Trigger, Chains |
| `all` (K8s) | Pipeline, Trigger, Chains, Dashboard |
| `all` (OpenShift) | Pipeline, Trigger, Chains, PaC, Addons |

### Helm alternative

The operator repo ships `charts/tekton-operator` — install via Flux GitRepository pinned to a release tag (e.g. `tekton-operator-0.79.0`) + HelmRelease with `installCRDs: true`. The `TektonConfig` CR is created separately after install (post-install), and the operator reconciles it.

## TektonConfig Example

```yaml
apiVersion: operator.tekton.dev/v1alpha1
kind: TektonConfig
metadata:
  name: config
spec:
  targetNamespace: tekton-pipelines   # openshift-pipelines on OpenShift
  targetNamespaceMetadata:
    labels: {}
    annotations: {}
  profile: all
  config:
    nodeSelector: <>
    tolerations: []
    priorityClassName: system-cluster-critical
  chain:
    disabled: false
  pipeline:
    coschedule: disabled           # workspaces | disabled
    enable-api-fields: beta
    enable-bundles-resolver: true
    enable-cel-in-whenexpression: false
    enable-cluster-resolver: true
    enable-custom-tasks: true
    enable-git-resolver: true
    enable-hub-resolver: true
    enable-provenance-in-status: true
    enable-step-actions: false
    max-result-size: 4096
    results-from: termination-message
    running-in-environment-with-injected-sidecars: true
    performance:
      disable-ha: false
      buckets: 1
      replicas: 1
      threads-per-controller: 2
      kube-api-qps: 5.0
      kube-api-burst: 10
  trigger:
    disabled: false
  dashboard:
    readonly: true
  chain:
    disabled: false
  pruner:
    disabled: true               # job-based pruner (cron); must be OFF for tektonpruner
    keep: 100
    resources: [pipelinerun]
    schedule: "0 8 * * *"
  tektonpruner:
    disabled: false
    global-config:
      enforcedConfigLevel: global
      ttlSecondsAfterFinished: 1800
      successfulHistoryLimit: 5
      failedHistoryLimit: 3
  result:
    is_external_db: true
    db_host: tekton-results-db-rw.tekton-pipelines.svc.cluster.local
    db_port: 5432
    db_name: tekton-results
    db_sslmode: disable
    db_secret_name: tekton-results-db-app
    db_secret_user_key: user
    db_secret_password_key: password
```

## Key Decisions

### Pruner vs TektonPruner (mutually exclusive)

| | `pruner` | `tektonpruner` |
|---|---|---|
| Mechanism | Cron job (schedule) | Event-based controller |
| Config | `keep` N resources | TTL + success/failure history limits |
| Recommendation | **Disable when using tektonpruner** | Prefer this for event-driven cleanup |

**Both cannot be enabled simultaneously.** Job-based `pruner` must be `disabled: true` for `tektonpruner` to work. TTL cleanup (`ttlSecondsAfterFinished`) + separate success/failure history limits is the modern pattern.

### Pipeline config highlights

- `enable-api-fields: beta` unlocks alpha→beta features; stable keeps v1 semantics
- Resolvers (bundles/cluster/git/hub) toggle independently
- `coschedule: workspaces` groups taskruns sharing workspaces
- `enable-step-actions` — task-level step actions (beta)
- `performance` block tunes controller replicas/threads (HA: `disable-ha: false` runs 2 replicas)

### Results with external DB (production)

Point `result.is_external_db: true` at your own Postgres (e.g. CNPG). The DB secret holds user/password keys (customizable via `db_secret_user_key`/`db_secret_password_key`). Results API keeps run history beyond PipelineRun object TTL — needed if PaC prunes runs but you still want audit.

## v0.80.0 Changes (breaking)

- **OpenTelemetry metrics migration** — config key `metrics.backend-destination` → `metrics-protocol`; workqueue metrics renamed `tekton_*lifecycle_*` → `kn_*`; several unused metrics removed.
- **Pipelines-as-Code on Kubernetes** — `OpenShiftPipelinesAsCode` CR now installable on plain K8s via `spec.platforms.kubernetes.pipelinesAsCode` (previously OpenShift-only).
- Centralized TLS config (OpenShift) — components inherit cluster APIServer TLS profile when `enableCentralTLSConfig` set.

## Common Mistakes

- **Creating component CRs directly** — Create `TektonConfig` (top-level) and let it own components. Individual CRs conflict with TektonConfig management.
- **Both pruners enabled** — `pruner` (job-based) and `tektonpruner` (event-based) are mutually exclusive. Disable `pruner` to use TTL-based cleanup.
- **Inline config vs ConfigMaps** — TektonConfig pipeline section overrides generated ConfigMaps; editing the ConfigMaps directly gets reverted by the operator on next reconcile.
- **Wrong targetNamespace** — defaults to `tekton-pipelines` (K8s) / `openshift-pipelines` (OpenShift); PaC cannot deploy in the same namespace as the operator (Repository CR forbidden there).
- **Forgetting installCRDs** — Helm install of the operator without `installCRDs: true` leaves CRDs missing; TektonConfig apply fails.
- **PaC deployed standalone + operator PaC** — if you vendored the PaC release (release.k8s.yaml), don't also enable `OpenShiftPipelinesAsCode` — duplicate controllers fight over webhooks.
- **Air-gap** — operator defaults pull images from upstream registries; follow the AirGapImageConfiguration guide for custom registries.
