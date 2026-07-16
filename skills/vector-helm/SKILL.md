---
name: vector-helm
description: Deploy Vector on Kubernetes via Helm — Agent/Aggregator/Stateless roles, customConfig, sources, transforms, sinks, and production patterns.
---

# Vector — Helm Chart

**Repo:** `https://helm.vector.dev`  
**OCI:** `oci://ghcr.io/vectordotdev/helm-charts/vector`  
**Latest chart:** v0.56.0 (app v0.56.0-distroless-libc)  
**K8s min:** `>=1.28.0-0`

## Quick Install

```bash
helm repo add vector https://helm.vector.dev
helm repo update

# Agent (DaemonSet — collect logs from all nodes)
helm install vector-agent vector/vector \
  --namespace vector --create-namespace \
  --set role=Agent \
  --set customConfig.sources.kubernetes_logs.type=kubernetes_logs \
  --set customConfig.sinks.stdout.type=console \
  --set 'customConfig.sinks.stdout.inputs[0]=kubernetes_logs' \
  --set customConfig.sinks.stdout.encoding.codec=json
```

## Roles

| Role | Workload | Use Case |
|------|----------|----------|
| `Agent` | DaemonSet | Collect logs/metrics from every node |
| `Aggregator` | StatefulSet | Centralized processing with persistent buffer |
| `Stateless-Aggregator` | Deployment | Centralized processing without persistence |

```bash
# Aggregator (StatefulSet — centralized processing)
helm install vector-aggregator vector/vector \
  --namespace vector \
  --set role=Aggregator \
  --set replicas=3 \
  --set customConfig.sources.vector.type=vector \
  --set customConfig.sources.vector.address=0.0.0.0:6000 \
  --set customConfig.sinks.stdout.type=console \
  --set 'customConfig.sinks.stdout.inputs[0]=vector' \
  --set customConfig.sinks.stdout.encoding.codec=json
```

## Configuration

### Pipeline via customConfig

```yaml
customConfig:
  data_dir: /vector-data-dir
  api:
    enabled: true
    address: 127.0.0.1:8686
  sources:
    kubernetes_logs:
      type: kubernetes_logs
      extra_label_selector: "app!=vector"
    internal_metrics:
      type: internal_metrics
    vector:
      address: 0.0.0.0:6000
      type: vector
      version: "2"

  transforms:
    remap:
      type: remap
      inputs: [kubernetes_logs]
      source: |
        .@timestamp = del(.timestamp)
        .service = .kubernetes.container_name

  sinks:
    console:
      type: console
      inputs: [remap]
      encoding:
        codec: json
    prometheus:
      type: prometheus_exporter
      inputs: [internal_metrics]
      address: 0.0.0.0:9090
```

### HAProxy Load Balancer (Aggregator)

```yaml
haproxy:
  enabled: true
  replicas: 2
  service:
    type: ClusterIP
```

### PodMonitor (Prometheus)

```yaml
podMonitor:
  enabled: true
  interval: 30s
  path: /metrics
  port: prom-exporter
```

## Key Parameters

| Parameter | Default | Role | Description |
|-----------|---------|------|-------------|
| `role` | `Aggregator` | all | Workload type |
| `image.repository` | `docker.io/timberio/vector` | all | Image repo |
| `image.tag` | derived from appVersion | all | Image tag |
| `image.base` | `distroless-libc` | all | Base distro |
| `replicas` | `1` | Agg/Stateless | Pod count |
| `customConfig` | `{}` | all | Pipeline definition |
| `existingConfigMaps` | `[]` | all | Use external ConfigMap for config |
| `dataDir` | `""` | all | Data dir (required with existingConfigMaps) |
| `secrets.generic` | `{}` | all | Key-value secrets |
| `persistence.enabled` | `false` | Aggregator | PVC for buffer |
| `persistence.size` | `10Gi` | Aggregator | PVC size |
| `service.enabled` | `true` | all | Create Service |
| `service.type` | `ClusterIP` | all | Service type |
| `rbac.create` | `true` | Agent | RBAC for pod/node access |
| `psp.create` | `false` | Agent | PodSecurityPolicy |
| `autoscaling.enabled` | `false` | Agg/Stateless | HPA |
| `podMonitor.enabled` | `false` | all | Prometheus PodMonitor |
| `haproxy.enabled` | `false` | Aggregator | HAProxy sidecar |
| `podLabels` | `{vector.dev/exclude: "true"}` | all | Labels on pods |
| `env` | `[]` | all | Environment variables |
| `envFrom` | `[]` | all | Env from Secrets/ConfigMaps |
| `extraVolumes` | `[]` | all | Additional volumes |
| `extraVolumeMounts` | `[]` | all | Additional mounts |
| `initContainers` | `[]` | all | Init containers |
| `extraContainers` | `[]` | all | Sidecar containers |
| `resources` | `{}` | all | CPU/memory requests/limits |
| `nodeSelector` | `{}` | all | Node selection |
| `tolerations` | `[]` | all | Taints |
| `affinity` | `{}` | all | Scheduling affinity |
| `updateStrategy` | `{}` | all | Rolling update strategy |
| `terminationGracePeriodSeconds` | `60` | all | Graceful shutdown |
| `livenessProbe` | `{}` | all | Liveness |
| `readinessProbe` | `{}` | all | Readiness |
| `startupProbe` | `{}` | all | Startup |
| `extraObjects` | `[]` | all | Extra K8s manifests |

## Secrets

```yaml
secrets:
  generic:
    datadog_api_key: "your-api-key"
    aws_access_key_id: "AKIA..."
    aws_secret_access_key: "..."
```

Referenced via env:

```yaml
env:
  - name: DATADOG_API_KEY
    valueFrom:
      secretKeyRef:
        name: vector
        key: datadog_api_key
```

## Upgrading

```bash
helm repo update
helm upgrade vector-agent vector/vector \
  --namespace vector \
  --values values.yaml \
  --version 0.56.0
```

## Common Mistakes

- **Role not set** — Default is `Aggregator`. Set `--set role=Agent` for DaemonSet.
- **customConfig replaces all defaults** — When using `customConfig`, you must define the full pipeline (sources, transforms, sinks). No defaults are merged.
- **Agent requires RBAC** — `rbac.create=true` (default) for reading pod logs. Disable only if using pre-created RBAC.
- **API health endpoint** — `customConfig.api.enabled=true` required for liveness/readiness probes. Without it, probes fail.
- **gRPC probes** — Vector 0.55+ supports gRPC health probes via `grpc.health.v1.Health/Check` on port `8686`.
- **Aggregator without persistence** — `persistence.enabled=false` means buffer data is lost on pod restart.
- **HostPath for Agent** — Agent uses `/var/log/`, `/var/lib/`, `/proc`, `/sys` mounts by default. Don't override unless you know the impact.
- **Base distribution** — `distroless-libc` is minimal. Use `debian` if you need shell access or debugging tools.
