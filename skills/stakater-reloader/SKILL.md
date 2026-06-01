---
name: stakater-reloader
description: Use when working with Stakater Reloader — watching ConfigMaps/Secrets and triggering workload rollouts on changes. Covers annotations, Helm values, patterns, and troubleshooting. NOT for CRD-based operators (no CRDs).
---

# Stakater Reloader

## Overview

Reloader watches ConfigMaps and Secrets in real time via the K8s watch API. When data content changes (not metadata), it patches the pod template of matching workloads — triggering a rolling restart that respects the workload's own `RollingUpdate` strategy and PDBs.

**No CRDs.** All configuration is annotation-based.

**Deployed:** chart 2.2.11, app v1.4.16 (`infrastructure/controllers/stakater-reloader.yaml`).
**Latest:** chart 2.2.12, app v1.4.17 (May 2026).

## Supported Workloads

| Workload | Notes |
|----------|-------|
| Deployment | Full support |
| StatefulSet | Full support |
| DaemonSet | Full support |
| Argo Rollout | Requires `reloader.isArgoRollouts: true` |
| CronJob | Supported |
| Job | Supported |
| DeploymentConfig | OpenShift, auto-detected |

## Watched Resources

| Resource | Enable |
|----------|--------|
| Secret | Default (disable: `reloader.ignoreSecrets: true`) |
| ConfigMap | Default (disable: `reloader.ignoreConfigMaps: true`) |
| SecretProviderClass | Requires `reloader.enableCSIIntegration: true` |

## Workload Annotations

Placed on Deployment/StatefulSet/DaemonSet/Rollout/CronJob/Job.

### Pattern 1: Auto

Watch all ConfigMaps and Secrets referenced in the pod spec (env vars, envFrom, volume mounts).

```yaml
metadata:
  annotations:
    reloader.stakater.com/auto: "true"
```

Typed variants (watch only one type):

```yaml
metadata:
  annotations:
    secret.reloader.stakater.com/auto: "true"        # Secrets only
    configmap.reloader.stakater.com/auto: "true"      # ConfigMaps only
    secretproviderclass.reloader.stakater.com/auto: "true"  # CSI only
```

### Pattern 2: Named Resource

Watch specific resources by name, even if not referenced in pod spec.

```yaml
metadata:
  annotations:
    secret.reloader.stakater.com/reload: "db-credentials,api-keys"
    configmap.reloader.stakater.com/reload: "app-config,feature-flags"
    secretproviderclass.reloader.stakater.com/reload: "vault-csi-provider"
```

### Pattern 3: Search + Match

Workload opts into search mode; only Secrets/ConfigMaps with `match: "true"` trigger reloads. Lets resource owners control reload behaviour.

```yaml
# On workload:
metadata:
  annotations:
    reloader.stakater.com/search: "true"

# On Secret/ConfigMap:
metadata:
  annotations:
    reloader.stakater.com/match: "true"
```

### Exclude Annotations

Skip specific resources from triggering reloads (works alongside `auto`):

```yaml
metadata:
  annotations:
    reloader.stakater.com/auto: "true"
    secrets.exclude.reloader.stakater.com/reload: "audit-log-secret"
    configmaps.exclude.reloader.stakater.com/reload: "shared-readonly-config"
    secretproviderclasses.exclude.reloader.stakater.com/reload: "csi-provider"
```

### Argo Rollout Strategy

```yaml
metadata:
  annotations:
    reloader.stakater.com/rollout-strategy: "restart"  # or "rollout" (default)
```

- `rollout`: patches pod template (GitOps may detect drift)
- `restart`: deletes pods directly (no drift, more disruptive)

### Pause Period

Cooldown after a reload to prevent rapid restarts:

```yaml
metadata:
  annotations:
    deployment.reloader.stakater.com/pause-period: "5m"
```

## Resource Annotations

Placed on ConfigMap or Secret.

| Annotation | Value | Effect |
|------------|-------|--------|
| `reloader.stakater.com/match` | `"true"` | Resource opt-in for search+match pattern |
| `reloader.stakater.com/ignore` | `"true"` | Skip this resource entirely, never triggers reload |

```yaml
metadata:
  annotations:
    reloader.stakater.com/ignore: "true"
```

## System Annotations (Read-Only)

Set by Reloader, do not modify:

| Annotation | Strategy | Description |
|------------|----------|-------------|
| `reloader.stakater.com/last-reloaded-from` | `annotations` | Which resource triggered the reload |
| `deployment.reloader.stakater.com/paused-at` | — | Timestamp when pause started |

## Reload Strategy

Controls *how* Reloader triggers the restart:

| Strategy | `reloader.reloadStrategy` | Mechanism | GitOps Compat |
|----------|--------------------------|-----------|---------------|
| Default | `default` (or `env-vars`) | Injects `STAKATER_<TYPE>_<HASH>` env var into pod template | ❌ Argo detects drift |
| Annotations | `annotations` | Adds `reloader.stakater.com/last-reloaded-from` annotation | ✅ No drift |

For Argo CD / GitOps environments, set `reloader.reloadStrategy: annotations`.

## Deployment

### HelmRelease (Flux)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: stakater
  namespace: kube-system
spec:
  interval: 24h
  url: https://stakater.github.io/stakater-charts
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: stakater-reloader
  namespace: kube-system
spec:
  interval: 1h
  chart:
    spec:
      chart: reloader
      sourceRef:
        kind: HelmRepository
        name: stakater
      version: 2.2.11
```

### Helm Values

| Value | Default | Description |
|-------|---------|-------------|
| `reloader.watchGlobally` | `true` | Watch all namespaces (`false` = deployment namespace only) |
| `reloader.autoReloadAll` | `false` | Treat all workloads as if `auto: "true"` |
| `reloader.reloadOnCreate` | `false` | Trigger reload on resource creation (not just updates) |
| `reloader.reloadOnDelete` | `false` | Trigger reload on resource deletion |
| `reloader.reloadStrategy` | `"default"` | `default`/`env-vars` or `annotations` |
| `reloader.isArgoRollouts` | `false` | Enable Argo Rollout support |
| `reloader.isOpenshift` | `false` | Enable OpenShift DeploymentConfig support |
| `reloader.enableCSIIntegration` | `false` | Enable Secrets Store CSI Driver support |
| `reloader.ignoreSecrets` | `false` | Ignore all Secrets |
| `reloader.ignoreConfigMaps` | `false` | Ignore all ConfigMaps |
| `reloader.ignoreJobs` | `false` | Ignore Jobs |
| `reloader.ignoreCronJobs` | `false` | Ignore CronJobs |
| `reloader.namespaceSelector` | `""` | Label selector to restrict watched namespaces |
| `reloader.ignoreNamespaces` | `""` | Comma-separated namespaces to exclude |
| `reloader.resourceLabelSelector` | `""` | Label selector to filter watched ConfigMaps/Secrets |
| `reloader.enableHA` | `false` | Leader election for multi-replica |
| `reloader.syncAfterRestart` | `false` | On leadership change, re-scan all workloads (requires `reloadOnCreate: true`) |
| `reloader.logFormat` | `""` | `"json"` for structured logs |
| `reloader.logLevel` | `"info"` | `trace`, `debug`, `info`, `warning`, `error` |
| `reloader.webhookUrl` | `""` | Webhook URL for notifications instead of restart |
| `reloader.podMonitor.enabled` | `false` | Prometheus PodMonitor for metrics |
| `reloader.deployment.replicas` | `1` | Replicas (keep 1 unless HA) |
| `reloader.deployment.resources` | `{}` | CPU/memory requests and limits |
| `reloader.deployment.nodeSelector` | `{}` | Node selector for pod scheduling |
| `reloader.custom_annotations` | `{}` | Override all annotation keys with custom domain |

## Behaviours & Rules

- **Data-only triggers:** Metadata changes (labels, annotations) never trigger reloads. Only data content changes.
- **`auto` overrides `search`:** If both are set, `auto` wins.
- **Exclude works with auto:** `secrets.exclude`/`configmaps.exclude` can selectively exclude resources even with `auto: "true"`.
- **`reloader.stakater.com/ignore` on a resource** overrides everything — any workload referencing it will not reload when it changes.
- **Global `autoReloadAll: true`** makes every workload auto-reload. Set `reloader.stakater.com/auto: "false"` on individual workloads to opt out.
- **Typed annotations are independent:** `secret.reloader.stakater.com/auto` and `configmap.reloader.stakater.com/auto` can be combined. Either being `"true"` enables reloading for that type.
- **Custom annotations replace defaults entirely.** When `custom_annotations` is set, the standard `reloader.stakater.com/*` keys are no longer recognised.

## Tools Integration

Reloader is tool-agnostic — it watches K8s Secrets/ConfigMaps regardless of how they're created:

- **External Secrets Operator** — writes K8s Secret → Reloader detects change → restarts workload
- **Secrets Store CSI Driver** — uses `SecretProviderClass` annotation + `enableCSIIntegration: true`
- **Vault Agent Injector** — creates K8s Secret from Vault → Reloader detects change
- **Flux/Helm/Kustomize** — fully compatible, no chart changes needed
- **Argo CD** — use `annotations` reload strategy to avoid sync drift

## Metrics

Prometheus metric: `reloader_reload_executed_total` (counter, labels: `namespace`, `name`, `resource`).

Enable scraping:

```yaml
reloader:
  podMonitor:
    enabled: true
```

## Common Mistakes

- **Setting `reloader.stakater.com/auto: "true"` but Secret doesn't exist yet.** Reloader only detects changes — it won't trigger on first deployment when resources are created simultaneously. Use `reloadOnCreate: true` for that.
- **Argo CD detects drift with default reload strategy.** Switch to `annotations` strategy if using GitOps.
- **`reloader.stakater.com/ignore: "true"` on the *resource*, not the workload.** This annotation goes on the ConfigMap/Secret, not the Deployment.
- **Exclude annotations on the wrong resource.** `secrets.exclude.*` and `configmaps.exclude.*` go on the *workload*, not the Secret/ConfigMap.
- **`reloader.deployment.replicas > 1` without `enableHA: true`.** The chart forces replicas to 1 if HA is disabled.
- **Custom annotations replace defaults entirely.** Don't set `custom_annotations` unless you want to migrate all workloads to new keys.
- **`syncAfterRestart` requires `reloadOnCreate: true`.** Without it, the leader election sync on startup does nothing.
- **Webhook URL doesn't trigger a restart.** It only sends a notification. Use it for logging/monitoring, not as a reload mechanism.
