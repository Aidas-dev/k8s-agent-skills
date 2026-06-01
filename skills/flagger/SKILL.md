---
name: flagger
description: Use when working with Flagger — progressive delivery, canary deployments, A/B testing, blue/green on Kubernetes. Covers Canary CRD, analysis/meshes/metrics/webhooks, Helm values. CRDs: Canary, MetricTemplate, AlertProvider.
---

# Flagger

## Overview

Flagger automates progressive delivery for Kubernetes workloads. It gradually shifts traffic to a new version while measuring metrics and running conformance tests. Supports canary releases (weighted traffic), A/B testing (header/cookie routing), and blue/green deployments (instant switch or mirroring).

**CRDs:** `Canary` (flagger.app/v1beta1), `MetricTemplate`, `AlertProvider`.

**Latest:** chart 1.43.0, app v1.43.0 (Apr 2026).

## Architecture

```
User creates/updates Canary resource
  → Flagger creates:
    - <name>-primary Deployment (stable version)
    - <name>-canary Deployment (new version)
    - <name> ClusterIP service (routes to primary)
    - <name>-primary ClusterIP service (stable)
    - <name>-canary ClusterIP service (new)
    - Mesh/Ingress routing objects (if mesh provider set)
  → Analysis loop:
    1. Increment traffic to canary (stepWeight)
    2. Run webhooks (pre-rollout, rollout, post-rollout)
    3. Check metrics (success rate, duration, custom)
    4. If all pass → promote canary to primary
    5. If threshold exceeded → rollback
```

## CRD: Canary

`apiVersion: flagger.app/v1beta1`, `kind: Canary`

### Minimal Example (Kubernetes CNI — no mesh)

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
  namespace: prod
spec:
  provider: kubernetes    # No service mesh (uses ClusterIP + pod labels)
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  service:
    port: 9898
    portDiscovery: true
  analysis:
    interval: 1m
    threshold: 5
    iterations: 10        # Used for blue/green with no mesh provider
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99
        interval: 1m
    webhooks:
      - name: load-test
        type: rollout
        url: http://flagger-loadtester.test/
        metadata:
          cmd: "hey -z 1m -q 10 http://myapp-canary.prod:9898/"
```

### CanarySpec Fields

| Field | Required | Description |
|-------|----------|-------------|
| `provider` | Yes | Traffic provider: `kubernetes`, `istio`, `linkerd`, `nginx`, `contour`, `gloo`, `traefik`, `gatewayapi:v1`, `apisix`, `kuma`, `knative`, `skipper`, `osm`, `smi:v1alpha2`, `appmesh:v1beta2` |
| `targetRef` | Yes | Target Deployment reference |
| `autoscalerRef` | No | HPA reference (copied to canary) |
| `service` | Yes | Service spec (port, portName, targetPort, hosts, gatewayRefs, match, rewrite, timeout, headers, etc.) |
| `suspend` | No | Suspend all canary runs |
| `progressDeadlineSeconds` | No | Max time for canary progress before rollback (default 600) |
| `skipAnalysis` | No | Promote without analysis (default false) |

### AnalysisSpec Fields

| Field | Required | Description |
|-------|----------|-------------|
| `interval` | Yes | Schedule interval (e.g. `1m`, `30s`) |
| `threshold` | Yes | Max failed checks before rollback |
| `maxWeight` | Canary | Max traffic % to canary (0-100). Used with `stepWeight` |
| `stepWeight` | Canary | Traffic increment per interval (0-100). Used with `maxWeight` |
| `stepWeights` | Canary | Explicit array of traffic weights. Replaces stepWeight |
| `stepWeightPromotion` | No | Traffic increment during promotion phase |
| `iterations` | A/B, Blue/Green | Number of iterations (replaces stepWeight/maxWeight) |
| `match` | A/B | HTTP header/cookie match conditions for A/B testing |
| `mirror` | Blue/Green | Mirror traffic to canary (default false) |
| `mirrorWeight` | No | % of traffic to mirror (0-100) |
| `primaryReadyThreshold` | No | % of pods that must be available before starting (% , default 100) |
| `canaryReadyThreshold` | No | % of canary pods that must be available (%, default 100) |
| `metrics` | No | List of metric checks |
| `webhooks` | No | List of webhooks (pre-rollout, rollout, confirm-promotion, etc.) |
| `alerts` | No | List of alert configs |
| `sessionAffinity` | No | Session affinity settings for canary |

### Analysis Strategies

| Strategy | Fields | Traffic shaping | Use case |
|----------|--------|-----------------|----------|
| Canary (weighted) | `stepWeight` + `maxWeight` | Gradual traffic shift | Gradual rollout with metrics |
| Canary (custom steps) | `stepWeights: [5, 10, 25, 50, 75]` | Custom traffic steps | Non-linear rollout |
| A/B Testing | `iterations` + `match` | Header/cookie routing | Test specific user segments |
| Blue/Green | `iterations` | Instant switch | Quick rollback or pre-production validation |
| Blue/Green Mirror | `iterations` + `mirror: true` | Traffic mirroring | Shadow traffic without impact |

### Metrics

```yaml
metrics:
  - name: request-success-rate
    thresholdRange:
      min: 99
    interval: 1m
  - name: request-duration
    thresholdRange:
      max: 500
    interval: 30s
  - name: custom-metric
    templateRef:
      name: my-metric-template
      namespace: flagger
    thresholdRange:
      min: 2
      max: 100
    interval: 1m
```

Built-in metric checks (when `templateRef` is not set):
- `request-success-rate` — Prometheus query `rate(...)` for non-5xx responses
- `request-duration` — Prometheus query `histogram_quantile(0.99, ...)` for P99 latency

Custom metrics use `MetricTemplate` CRD (see below).

### Webhooks

```yaml
webhooks:
  - name: "load test"
    type: rollout              # Run during canary analysis
    url: http://tester/        # Webhook endpoint
    timeout: 5m
    retries: 3
    disableTLS: false
    metadata:
      cmd: "hey -z 1m http://app:9898/"
```

Webhook types (execution order):

| Type | Phase | Purpose |
|------|-------|---------|
| `pre-rollout` | Before canary starts | Acceptance tests, DB migrations check |
| `confirm-rollout` | Before canary starts (gating) | Manual approval gate |
| `rollout` | During analysis (each step) | Load tests |
| `confirm-promotion` | Before promotion (gating) | Manual approval for promotion |
| `post-rollout` | After promotion | Smoke tests, cleanup |
| `rollback` | After rollback | Cleanup, notifications |
| `event` | Any time | Informational events |
| `confirm-traffic-increase` | Before each step increase (gating) | Per-step manual approval |

### Alerts

```yaml
alerts:
  - name: "Slack"
    severity: error       # info, warn, error
    providerRef:
      name: dev-slack
      namespace: flagger
```

## CRD: MetricTemplate

`apiVersion: flagger.app/v1beta1`, `kind: MetricTemplate`

Defines custom metric queries for canary analysis:

```yaml
apiVersion: flagger.app/v1beta1
kind: MetricTemplate
metadata:
  name: db-connections
  namespace: flagger
spec:
  provider:
    type: prometheus
    address: http://prometheus.monitoring:9090
  query: |
    avg_over_time(
      pg_stat_activity_count{namespace="{{ namespace }}",app="{{ target }}"}[{{ interval }}]
    )
```

Template variables: `{{ namespace }}`, `{{ target }}`, `{{ interval }}`.

## CRD: AlertProvider

`apiVersion: flagger.app/v1beta1`, `kind: AlertProvider`

```yaml
apiVersion: flagger.app/v1beta1
kind: AlertProvider
metadata:
  name: dev-slack
  namespace: flagger
spec:
  type: slack
  channel: flagger-alerts
  username: flager
  address: https://hooks.slack.com/services/TOKEN
```

Supported types: `slack`, `teams`, `discord`, `rocket`.

## Deployment (Flux HelmRelease)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: flagger
  namespace: flagger-system
spec:
  interval: 24h
  url: https://flagger.app
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: flagger
  namespace: flagger-system
spec:
  chart:
    spec:
      chart: flagger
      sourceRef:
        kind: HelmRepository
        name: flagger
      version: "1.39.0"
  values:
    meshProvider: ""                # Kubernetes CNI mode
    metricsServer: "http://prometheus:9090"
    prometheus:
      install: false                # Use existing Prometheus
```

## Helm Values

| Value | Default | Description |
|-------|---------|-------------|
| `meshProvider` | `""` (kubernetes) | Traffic provider: istio, linkerd, nginx, contour, kubernetes, etc. |
| `metricsServer` | `http://prometheus.istio-system:9090` | Prometheus URL |
| `logLevel` | `info` | Log level |
| `crd.create` | `false` | Create CRDs (Helm v3 handles this separately) |
| `prometheus.install` | `false` | Install bundled Prometheus |
| `prometheus.retention` | `2h` | Prometheus data retention |
| `serviceMonitor.enabled` | `false` | Create ServiceMonitor |
| `podMonitor.enabled` | `false` | Create PodMonitor |
| `namespace` | `""` (all) | Watch single namespace (empty = all) |
| `selectorLabels` | `app,name,app.kubernetes.io/name` | Labels for workload selection |

## Provider-Specific Features

| Feature | Istio | Linkerd | Contour | NGINX | Kubernetes | Gateway API |
|---------|-------|---------|---------|-------|-----------|-------------|
| Weighted canary | ✅ | ✅ | ✅ | ✅ | ➖ | ✅ |
| A/B testing | ✅ | ➖ | ✅ | ✅ | ➖ | ✅ |
| Blue/green (switch) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Blue/green (mirror) | ✅ | ➖ | ➖ | ➖ | ➖ | ➖ |
| Request success rate | ✅ | ✅ | ✅ | ➖ | ✅ | ✅ |
| Request duration | ✅ | ✅ | ✅ | ➖ | ✅ | ✅ |

## Common Mistakes

- **`meshProvider: ""` (Kubernetes CNI) has no traffic shaping.** Flagger can only do blue/green (iterations-based) with the `kubernetes` provider. Weighted canary (stepWeight) requires a service mesh or ingress controller.
- **Prometheus must be reachable.** Without `metricsServer`, the analysis loop immediately fails. Verify Prometheus URL and that Flagger can query it.
- **CRDs not installed.** `crd.create: false` means CRDs must be installed separately. If running Flux, the CRDs from the upstream `crds.yaml` must exist before Canary resources are applied.
- **Webhook URL must be reachable from Flagger pod.** Load test webhooks are called during canary analysis. If the webhook times out or returns error, the canary fails. Use cluster-internal URLs.
- **`targetRef` must be a Deployment.** Flagger only supports Deployment as the target. Other workload types (StatefulSet, DaemonSet) won't work.
- **`progressDeadlineSeconds` too low.** If canary takes longer than this (e.g., image pull delay, slow startup), Flagger rolls back. Default 600s. Increase for large images.
- **Missing `service.port`.** Required field. Flagger creates ClusterIP services and needs to know the container port.
- **Metric template `{{ target }}` defaults to the canary name.** In Prometheus queries, `{{ target }}` resolves to the service name. Ensure your metrics service matches the label selectors Flagger sets.
