# Vector Operator (kaasops)

**Repository:** `github.com/kaasops/vector-operator`  
**Latest release:** v0.4.1 (Jan 12, 2026)  
**API version:** `observability.kaasops.io/v1alpha1`  
**Stars:** 174 | **License:** Apache-2.0

## CRDs (5)

| CRD | Scope | Deploys |
|-----|-------|---------|
| **Vector** | Namespace | DaemonSet + Secret + Service + RBAC |
| **VectorPipeline** | Namespace | Config fragment (auto-routed by source type) |
| **ClusterVectorPipeline** | Cluster | Cluster-scoped pipeline config |
| **VectorAggregator** | Namespace | Deployment + Secret + Service + RBAC |
| **ClusterVectorAggregator** | Cluster | Cluster-scoped aggregator |

## Architecture

```
VectorPipeline (namespaced sources)
  │
  ▼ (auto-routed)
Vector (DaemonSet — agent) ──► VectorAggregator (Deployment — aggregator)
                                      │
                                      ▼
                                Sinks (console, elasticsearch, kafka, etc.)
```

Pipelines with `kubernetes_logs` or file-based sources → auto-routed to **agent** (DaemonSet).  
Pipelines with network sources (vector, kafka, socket, http_server) → auto-routed to **aggregator** (Deployment).

## Installation

### Helm

```bash
helm repo add kaasops https://kaasops.github.io/vector-operator/
helm install vector-operator kaasops/vector-operator \
  --namespace vector-system --create-namespace \
  --version 0.4.1
```

### CRD Management

`createCRD: true` (default) — operator manages CRD lifecycle. Warning: removing chart deletes CRDs.

```
- --createCRD=false  # Manage CRDs externally
```

## Vector CR

### Agent (DaemonSet)

```yaml
apiVersion: observability.kaasops.io/v1alpha1
kind: Vector
metadata:
  name: vector-agent
  namespace: vector
spec:
  agent:
    image: timberio/vector:0.56.0-distroless-libc
    dataDir: /vector-data-dir
    expireMetricsSecs: 300
    api:
      enabled: true
      address: "[::]:8686"
      healthcheck: true
    internalMetrics: true
    service: false
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi
    tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
    env:
      - name: ENVIRONMENT
        value: production
    annotations:
      prometheus.io/scrape: "true"
```

#### Agent Spec Fields

| Field | Default | Description |
|-------|---------|-------------|
| `image` | `timberio/vector:0.48.0-distroless-libc` | Vector image |
| `dataDir` | `/vector-data-dir` | Data directory |
| `expireMetricsSecs` | `300` | Metrics expiration |
| `api.enabled` | `false` | Enable GraphQL API |
| `api.address` | `[::]:8686` | API bind address |
| `api.healthcheck` | `false` | Enable /health probes |
| `api.playground` | `false` | GraphQL playground |
| `internalMetrics` | `false` | Enable internal metrics + PodMonitor |
| `service` | `false` | Create Service for DaemonSet |
| `resources` | — | Container resources |
| `affinity` | — | Pod affinity |
| `tolerations` | — | Node tolerations |
| `securityContext` | — | Pod security context |
| `containerSecurityContext` | — | Container security context |
| `schedulerName` | — | Custom scheduler |
| `runtimeClassName` | — | Runtime class |
| `hostAliases` | — | Pod host aliases |
| `readinessProbe` | — | Readiness probe |
| `livenessProbe` | — | Liveness probe |
| `volumes` | — | Additional volumes |
| `volumeMounts` | — | Additional mounts |
| `priorityClassName` | — | Priority class |
| `hostNetwork` | — | Host network |
| `env` | — | Environment variables |
| `envFrom` | — | Env from Secrets/ConfigMaps |
| `annotations` | — | Pod/Service annotations |
| `labels` | — | Additional labels |
| `imagePullSecrets` | — | Registry pull secrets |

## VectorAggregator CR

### Aggregator (Deployment)

```yaml
apiVersion: observability.kaasops.io/v1alpha1
kind: VectorAggregator
metadata:
  name: vector-aggregator
  namespace: vector
spec:
  image: timberio/vector:0.56.0-distroless-libc
  dataDir: /vector-data-dir
  replicas: 3
  api:
    enabled: true
    address: "[::]:8686"
  internalMetrics: true
  service:
    type: ClusterIP
  resources:
    requests:
      cpu: 200m
      memory: 512Mi
    limits:
      cpu: 2
      memory: 2Gi
  # Same fields as agent.affinity, agent.tolerations, agent.env, etc.
```

### Aggregator Spec Fields

| Field | Default | Description |
|-------|---------|-------------|
| `image` | `timberio/vector:0.48.0-distroless-libc` | Vector image |
| `dataDir` | `/vector-data-dir` | Data directory |
| `replicas` | `1` | Pod count |
| `api` | — | Same as agent API spec |
| `internalMetrics` | `false` | Metrics + PodMonitor |
| `service` | — | Service configuration |
| `resources` | — | Container resources |
| `affinity` | — | Pod affinity/anti-affinity |
| `tolerations` | — | Node tolerations |
| `securityContext` | — | Pod security |
| `nodeSelector` | — | Node labels |
| `env` | — | Environment vars |
| `volumes` | — | Additional volumes |
| `volumeMounts` | — | Additional mounts |
| `readinessProbe` | — | Readiness |
| `livenessProbe` | — | Liveness |
| `annotations` | — | Pod annotations |
| `labels` | — | Additional labels |

## VectorPipeline CR

Pipeline config fragments are auto-merged into the running Vector config. Supports full Vector sources/transforms/sinks.

```yaml
apiVersion: observability.kaasops.io/v1alpha1
kind: VectorPipeline
metadata:
  name: app-logs
  namespace: vector
spec:
  sources:
    kubernetes_logs:
      type: kubernetes_logs
      extra_label_selector: "app!=testdeployment"
    syslog:
      type: syslog
      address: 0.0.0.0:514
      mode: udp

  transforms:
    remap:
      type: remap
      inputs: [kubernetes_logs]
      source: |
        .@timestamp = del(.timestamp)
        .app = .kubernetes.labels.app
        .namespace = .kubernetes.namespace
    filter_errors:
      type: filter
      inputs: [remap]
      condition:
        type: vrl
        source: .status != "error"

  sinks:
    elasticsearch:
      type: elasticsearch
      inputs: [filter_errors]
      endpoint: http://elasticsearch:9200
      mode: bulk
      encoding:
        codec: json
    console:
      type: console
      inputs: [syslog]
      encoding:
        codec: json
```

### ShortNames

- `vp` — VectorPipeline
- `va` — VectorAggregator (may not be registered by default; use full CRD name if not)

### PipelineSpec Fields

| Field | Type | Description |
|-------|------|-------------|
| `sources` | `map[string]Source` | Vector sources (any type) |
| `transforms` | `map[string]Transform` | Vector transforms (any type) |
| `sinks` | `map[string]Sink` | Vector sinks (any type) |

All fields follow the [Vector configuration reference](https://vector.dev/docs/reference/configuration/).

## ClusterVectorPipeline CR

Same as VectorPipeline but cluster-scoped. Sources from any namespace.

```yaml
apiVersion: observability.kaasops.io/v1alpha1
kind: ClusterVectorPipeline
metadata:
  name: cluster-wide-logs
spec:
  sources:
    kubernetes_logs:
      type: kubernetes_logs
  transforms:
    enrich:
      type: remap
      inputs: [kubernetes_logs]
      source: |
        .cluster = "production"
  sinks:
    elasticsearch:
      type: elasticsearch
      inputs: [enrich]
      endpoint: http://elasticsearch:9200
```

## ClusterVectorAggregator CR

Cluster-scoped aggregator. Same spec as VectorAggregator but available across namespaces.

```yaml
apiVersion: observability.kaasops.io/v1alpha1
kind: ClusterVectorAggregator
metadata:
  name: cluster-aggregator
spec:
  image: timberio/vector:0.56.0-distroless-libc
  replicas: 3
  internalMetrics: true
```

## Common Patterns

### Collect logs from specific namespaces

```yaml
apiVersion: observability.kaasops.io/v1alpha1
kind: VectorPipeline
metadata:
  name: production-logs
  namespace: vector
spec:
  sources:
    k8s:
      type: kubernetes_logs
      extra_label_selector: "app!=excluded-app"
      extra_namespace_label_selector: "environment=production"
  sinks:
    elastic:
      type: elasticsearch
      inputs: [k8s]
      endpoint: https://elastic.prod.svc:9200
```

### Secure credentials

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vector-secrets
  namespace: vector
type: Opaque
stringData:
  elastic_password: "secure-pass"
  datadog_api_key: "dap-key"
```

```yaml
# VectorPipeline referencing secrets via env vars
apiVersion: observability.kaasops.io/v1alpha1
kind: Vector
metadata:
  name: vector-agent
  namespace: vector
spec:
  agent:
    image: timberio/vector:0.56.0-distroless-libc
    env:
      - name: ELASTIC_PASSWORD
        valueFrom:
          secretKeyRef:
            name: vector-secrets
            key: elastic_password
```

### Collect journald logs

```yaml
apiVersion: observability.kaasops.io/v1alpha1
kind: VectorPipeline
metadata:
  name: journald-logs
spec:
  sources:
    journald:
      type: journald
      current_boot_only: true
  sinks:
    console:
      type: console
      inputs: [journald]
      encoding:
        codec: json
```

### Collect logs from files

```yaml
apiVersion: observability.kaasops.io/v1alpha1
kind: VectorPipeline
metadata:
  name: file-logs
spec:
  sources:
    app_logs:
      type: file
      include:
        - /var/log/app/*.log
      exclude:
        - /var/log/app/*.gz
  sinks:
    console:
      type: console
      inputs: [app_logs]
      encoding:
        codec: json
```

### Aggregator with HA

```yaml
apiVersion: observability.kaasops.io/v1alpha1
kind: VectorAggregator
metadata:
  name: vector-aggregator
  namespace: vector
spec:
  image: timberio/vector:0.56.0-distroless-libc
  replicas: 3
  api:
    enabled: true
  service:
    type: ClusterIP
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
```

## Helm Values (Operator)

```yaml
image:
  repository: kaasops/vector-operator
  tag: "v0.4.1"
  pullPolicy: IfNotPresent

replicaCount: 1
createCRD: true

resources:
  limits:
    cpu: "1"
    memory: 1Gi
  requests:
    cpu: 100m
    memory: 50Mi

args:
#  - "-watch-namespace=vector"
#  - "-watch-name=vector-operator"
#  - "-enable-reconciliation-invalid-pipelines=true"
#  - "-reconciliation-retry-delay=120s"

vector:
  enable: false
  name: "vector"
  useApiServerCache: false
  # agent:
  #   image: timberio/vector:0.48.0-distroless-libc

openshift:
  enabled: false
```

### Operator CLI Flags

| Flag | Description |
|------|-------------|
| `-watch-namespace` | Namespace filter for watched objects |
| `-watch-name` | Filter by `app.kubernetes.io/managed-by` label |
| `-enable-reconciliation-invalid-pipelines` | Reconcile pipelines with invalid configs |
| `-reconciliation-retry-delay` | Retry delay (default: 120s) |

## Common Mistakes

- **VectorPipeline without matching Vector/VectorAggregator** — Pipeline config is only applied if a Vector or VectorAggregator exists in the same namespace.
- **Source type routing** — `kubernetes_logs` and `file` sources → agent. Network sources (`vector`, `kafka`, `socket`) → aggregator. Mismatched routing = config not applied.
- **Namespace isolation** — VectorPipeline in namespace A won't be applied to Vector in namespace B. Use ClusterVectorPipeline for cross-namespace.
- **API healthcheck for probes** — Set `api.enabled: true` and `api.healthcheck: true` on Vector CR for liveness/readiness probes to work.
- **CRD deletion** — `createCRD: true` + `helm uninstall` = CRDs deleted = ALL Vector CRs disappear.
- **image tag** — Operator default image is `timberio/vector:0.48.0-distroless-libc`. Always set to match your desired Vector version.
- **PodMonitor** — Requires `internalMetrics: true` on Vector CR and the PodMonitor CRD installed (prometheus-operator).
- **Secret rotation** — Changing secrets in `Secrets` resources doesn't auto-reload Vector pod. Restart or use `env` with `envFrom` for automatic reload.
