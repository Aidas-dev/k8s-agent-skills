---
name: victoriametrics-operator
description: Use when creating or managing VictoriaMetrics Operator CRDs — VMAgent, VMSingle, VMCluster, VMAlert, VMAlertmanager, VMAuth, VMUser, VMServiceScrape, VMPodScrape, VMRule, VMProbe, VMNodeScrape, VMScrapeConfig, VMStaticScrape, VMAlertmanagerConfig, VLogs, VLAgent, VLCluster, VMDistributed.
---

# VictoriaMetrics Operator CRDs

Helm chart: `oci://ghcr.io/victoriametrics/helm-charts/victoria-metrics-operator`  
Latest chart: **0.63.1** | API: `v1beta1` (metrics), `v1` (logs)

All CRDs are namespaced. Operator must be installed with `crds.enabled=true keep=true` to retain CRDs on chart uninstall.

## CRD Reference

### Metrics — Storage

| CRD | API | Purpose | Key Fields |
|-----|-----|---------|------------|
| **VMSingle** | `v1beta1` | Single-node metrics storage. Good for small-medium scale. Vertical scaling. | `retentionPeriod`, `storage`, `extraArgs`, `dedup.minScrapeInterval`, `nodeSelector`, `resources`, `securityContext`, `image.tag`, `license` |
| **VMCluster** | `v1beta1` | Multi-node cluster with vminsert/vmselect/vmstorage. For large scale, multi-tenant. | `vmstorage` (replicationFactor, dedup), `vmselect`, `vminsert`, `serviceScrape`, `topologySpreadConstraints` |

### Metrics — Collection

| CRD | API | Purpose | Key Fields |
|-----|-----|---------|------------|
| **VMAgent** | `v1beta1` | Metrics scraper/collector — pulls from scrape targets, pushes to remote storage. Supports Deployment, StatefulSet, or DaemonSet mode. | `remoteWrite[]`, `selectAllByDefault`, `serviceScrapeSelector`, `podScrapeSelector`, `nodeScrapeNamespaceSelector`, `daemonSetMode`, `hostNetwork`, `inlineScrapeConfig`, `scrapeInterval`, `scrapeClasses[]`, `additionalScrapeConfigs` |

### Metrics — Scrape Configuration

| CRD | API | Purpose | Key Fields |
|-----|-----|---------|------------|
| **VMServiceScrape** | `v1beta1` | Scrape targets from Service endpoints. Drop-in replacement for Prometheus ServiceMonitor. | `selector.matchLabels`, `endpoints[]` (port, interval, path, scheme, tlsConfig, basicAuth), `namespaceSelector` (any:true) |
| **VMPodScrape** | `v1beta1` | Scrape targets from Pods directly (no Service needed). | `selector.matchLabels`, `podMetricsEndpoints[]`, `namespaceSelector` |
| **VMNodeScrape** | `v1beta1` | Scrape targets from all nodes in cluster. Useful for node_exporter. | `selector`, `endpoints[]` |
| **VMStaticScrape** | `v1beta1` | Static scrape targets with known addresses. | `targets[]` (static address list) |
| **VMProbe** | `v1beta1` | Probe configuration for blackbox-exporter style checks. | `endpoint` (host, port, scheme), `module`, `targets` (static, ingress, prometheusDiscovery) |
| **VMScrapeConfig** | `v1beta1` | Custom scrape config with full Prometheus service discovery support (consul, nomad, ec2, gce, azure, etc.). | `scrapeConfig` (inline YAML), `sdConfigs` |

### Alerting

| CRD | API | Purpose | Key Fields |
|-----|-----|---------|------------|
| **VMAlert** | `v1beta1` | Alerting/recording rules engine — evaluates rules against datasource, sends alerts to notifier. | `datasource.url`, `remoteWrite.url`, `remoteRead.url`, `notifier.url`, `evaluationInterval`, `selectAllByDefault`, `ruleSelector`, `ruleNamespaceSelector` |
| **VMAlertmanager** | `v1beta1` | Alert notification dispatcher — handles dedup, grouping, silencing, routing. | `replicaCount`, `configRawYaml` (alertmanager YAML config), `configSecret`, `nodeSelector`, `resources` |
| **VMAlertmanagerConfig** | `v1beta1` | Alertmanager routing config CRD. Defines routes, receivers, time intervals, mute time intervals, inhibition rules. | `route`, `receivers[]`, `timeIntervals[]`, `muteTimeIntervals[]`, `inhibitRules[]` |
| **VMRule** | `v1beta1` | Alerting and recording rule definitions. | `groups[].rules[]` (alert, expr, for, labels, annotations) |

### Auth

| CRD | API | Purpose | Key Fields |
|-----|-----|---------|------------|
| **VMAuth** | `v1beta1` | Auth proxy/router — routes requests to backend VM components based on path/user. | `userSelector`, `userNamespaceSelector`, `unauthorizedAccessSpec`, `extraArgs` |
| **VMUser** | `v1beta1` | User definition for VMAuth. Each user gets a generated or static password. | `username`, `password`/`generatePassword`, `targetRefs[]` (crd.kind, crd.name, crd.namespace, paths[]), `bearerToken` |

### Logs (API `v1`)

| CRD | API | Purpose | Key Fields |
|-----|-----|---------|------------|
| **VLSingle** | `v1` | Single-node VictoriaLogs storage. | `retentionPeriod`, `storage`, `nodeSelector`, `resources`, `extraArgs`, `removeAnnotations` |
| **VLAgent** | `v1` | Log collector — scrapes K8s pod logs via k8sCollector, ships to VLSingle. | `remoteWrite[]`, `k8sCollector` (enabled, includePodLabels, excludeFilter), `remoteWriteSettings`, `image.tag` |
| **VLCluster** | `v1` | Multi-node VictoriaLogs cluster (vlinsert, vlselect, vlstorage). | `vlstorage`, `vlselect`, `vlinsert`, `replicationFactor` |

### Advanced

| CRD | API | Purpose |
|-----|-----|---------|
| **VMDistributed** | `v1beta1` | Distributed setup across zones — manages VMAgent, VMCluster, VMAlert, VMAlertmanager, VMAuth per zone. Automates multi-zone deployment. |
| **VMAnomaly** | `v1beta1` | Anomaly detection engine (vmanomaly) — ML-based anomaly detection for metrics. |

## Common Patterns

### VMAgent — DaemonSet + HostNetwork (Talos)

VMAgent runs on every node, uses host networking (no CNI dependency for scraping):

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMAgent
metadata:
  name: vmagent
spec:
  selectAllByDefault: true
  serviceScrapeNamespaceSelector: {}
  podScrapeNamespaceSelector: {}
  daemonSetMode: true
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
  remoteWrite:
    - url: http://vmsingle:8428/api/v1/write
  inlineScrapeConfig: |
    - job_name: default-pods
      kubernetes_sd_configs:
        - role: pod
```

### VMSingle — Storage with Dedup

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMSingle
metadata:
  name: vmetrics
spec:
  retentionPeriod: "30d"
  storage:
    storageClassName: ceph-block
    resources:
      requests:
        storage: 50Gi
  extraArgs:
    dedup.minScrapeInterval: 30s
    memory.allowedPercent: "60"
  nodeSelector:
    kubernetes.io/hostname: worker-proxmox
```

### VMServiceScrape — Auto-Discover Service

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceScrape
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: my-app
  endpoints:
    - port: http
      interval: 30s
```

### VMAlert + VMRule — Alerting Pipeline

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMAlert
metadata:
  name: vmalert
spec:
  datasource:
    url: http://vmsingle:8428
  notifier:
    url: http://vmalertmanager:9093
  evaluationInterval: 20s
  selectAllByDefault: true
---
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMRule
metadata:
  name: alerts
spec:
  groups:
    - name: k8s
      rules:
        - alert: InstanceDown
          expr: up == 0
          for: 1m
          labels:
            severity: critical
```

### VLogs + VLAgent — Log Pipeline

```yaml
apiVersion: operator.victoriametrics.com/v1
kind: VLSingle
metadata:
  name: vlogs
spec:
  retentionPeriod: "30d"
  storage:
    storageClassName: ceph-block
    resources:
      requests:
        storage: 30Gi
---
apiVersion: operator.victoriametrics.com/v1
kind: VLAgent
metadata:
  name: vlagent
spec:
  remoteWrite:
    - url: http://vlsingle:9428/insert/native
  k8sCollector:
    enabled: true
    includePodLabels: true
```

### VMUser + VMAuth — Access Control

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMAuth
metadata:
  name: vmauth
spec:
  userNamespaceSelector: {}
  userSelector: {}
---
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMUser
metadata:
  name: grafana
spec:
  username: grafana
  generatePassword: true
  targetRefs:
    - crd:
        kind: VMSingle
        name: vmetrics
        namespace: monitoring
      paths:
        - "/select/.*"
        - "/vmui.*"
```

## Quick Reference

| Need | Use CRD |
|------|---------|
| Store metrics | VMSingle or VMCluster |
| Scrape metrics from services | VMServiceScrape |
| Scrape metrics from pods directly | VMPodScrape |
| Scrape metrics from nodes | VMNodeScrape |
| Probe endpoints (blackbox) | VMProbe |
| Custom scrape configs | VMScrapeConfig |
| Alert rules | VMRule |
| Evaluate rules & send alerts | VMAlert |
| Route/group/silence alerts | VMAlertmanager + VMAlertmanagerConfig |
| Auth proxy with user mgmt | VMAuth + VMUser |
| Store logs | VLSingle or VLCluster |
| Collect K8s pod logs | VLAgent |
| Multi-zone HA | VMDistributed |
| ML anomaly detection | VMAnomaly |

## Common Mistakes

- **Scrape config not picked up** — VMAgent uses selectors (`serviceScrapeSelector`, `podScrapeSelector`). If `selectAllByDefault: true` is not set, must match labels. Empty `{}` selectors match all.
- **VMSingle dedup not set** — Without `dedup.minScrapeInterval`, HA VMAgent sends duplicate data. Set on VMSingle `extraArgs`.
- **VMServiceScrape role deprecated** — `endpointslices` role is deprecated as of v0.67.0. Use `endpointslice` instead.
- **VMAgent DaemonSet on Talos** — Use `daemonSetMode: true` + `hostNetwork: true`. No CNI needed. Set `dnsPolicy: ClusterFirstWithHostNet`.
- **VMAgent hostNetwork port conflict** — Default VMAgent port is `8429`. Explicitly set `port: "8429"` to avoid conflicts.
- **VMAgent nodes/proxy RBAC** — Scraping K8s API (/api/v1/nodes/...) needs `nodes/proxy` ClusterRole.
- **Alertmanager configRawYaml vs configSecret** — Use `configRawYaml` for inline config (simple). Use `configSecret` for complex/versioned configs.
- **VLAgent image tag** — Must set `image.tag` explicitly; operator may not default to latest. Defaults may lag.
- **VMAuth user path patterns** — Use regex paths (`"/select/.*"`), not globs. Test with target service.
- **Prometheus CRD compatibility** — VM Operator can convert `ServiceMonitor`/`PodMonitor`/`PrometheusRule`/`Probe`/`ScrapeConfig` from prometheus-operator. Set `-controller.disableReconcileFor` to disable unwanted converters.
