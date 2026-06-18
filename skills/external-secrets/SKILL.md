---
name: external-secrets
description: Use when working with External Secrets Operator (ESO) — syncing K8s Secrets from external providers (HashiCorp Vault, AWS, GCP, Azure). Covers SecretStore, ExternalSecret, PushSecret, template engine, Helm values, Vault provider auth.
---

# External Secrets Operator

## Overview

External Secrets Operator (ESO) synchronizes Kubernetes Secrets from external secret management systems. It watches custom resources and reconciles K8s Secrets from the provider's values.

**CRDs:** `SecretStore`, `ClusterSecretStore`, `ExternalSecret`, `ClusterExternalSecret`, `PushSecret`, `ClusterPushSecret`.

**Latest:** chart 2.6.0, app v2.6.0.

## Architecture

```
K8s API (ExternalSecret CR)
  → ESO controller watches for changes
    → Queries external provider (Vault, AWS, GCP, Azure...)
      → Creates/updates K8s Secret
        → Template engine processes data (v2)
```

Three components:

| Component | Role |
|-----------|------|
| **Core Controller** | Watches ExternalSecret/SecretStore CRs, reconciles Secrets, calls provider APIs |
| **Cert Controller** | Manages webhook certificate lifecycle (can use cert-manager) |
| **Webhook** | Validating/mutating webhook for CRD validation |

## CRDs

### SecretStore / ClusterSecretStore

Defines how ESO authenticates to a provider. `SecretStore` is namespaced, `ClusterSecretStore` is cluster-scoped.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://vault.vault:8200"
      path: "secret"
      version: v2
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "eso-reader"
          serviceAccountRef:
            name: "external-secrets"
```

Key fields:

| Field | Description |
|-------|-------------|
| `spec.provider` | Provider type (vault, aws, gcp, azure, etc.) |
| `spec.retrySettings` | Max retries + retry interval on provider failure |
| `spec.conditions` | Service reference for namespace validation |

**Provider auth methods (Vault):**

| Auth | Config | Use case |
|------|--------|----------|
| `kubernetes` | SA bound to role via `vault policy write` + K8s auth role | In-cluster, SA-based |
| `token` | Static token from SecretKeyRef | External or bootstrapping |
| `approle` | roleId + secretId from SecretKeyRef | Machine-to-machine |
| `cert` | Client cert from SecretKeyRef | mTLS |
| `jwt` | JWT from SA or SecretKeyRef | OIDC-integrated |

### ExternalSecret

Defines what secret to fetch and how to store it as a K8s Secret.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
spec:
  refreshInterval: "1h"
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: my-app-secret          # K8s Secret name
    creationPolicy: Owner
    deletionPolicy: Retain
    template:
      type: Opaque
      metadata:
        labels:
          app: my-app
  data:
    - secretKey: password
      remoteRef:
        key: secret/data/myapp
        property: password
  dataFrom:
    - extract:
        key: secret/data/myapp
```

Key fields:

| Field | Description |
|-------|-------------|
| `spec.refreshInterval` | How often to re-fetch from provider (`1h`, `5m`, `CreatedOnce`) |
| `spec.secretStoreRef` | Reference to SecretStore or ClusterSecretStore |
| `spec.target.name` | Name of the K8s Secret to create/update |
| `spec.target.creationPolicy` | `Owner` (ESO manages), `Orphan`, `Merge`, `None` |
| `spec.target.deletionPolicy` | `Retain` (keep Secret on ES delete), `Delete` |
| `spec.target.template` | Template engine to transform data before writing to Secret |
| `spec.data` | Individual key mapping with `remoteRef` |
| `spec.dataFrom` | Bulk fetch: `extract` (all keys), `find` (by regex/name) |

### PushSecret

Pushes a K8s Secret value to an external provider (reverse sync).

```yaml
apiVersion: external-secrets.io/v1beta1
kind: PushSecret
metadata:
  name: push-myapp
spec:
  refreshInterval: "1h"
  secretStoreRefs:
    - name: vault-backend
      kind: ClusterSecretStore
  data:
    - match:
        secretKey: password
        remoteRef:
          remoteKey: secret/data/pushed/myapp
```

### ClusterExternalSecret

Cluster-scoped ExternalSecret that projects into multiple namespaces:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterExternalSecret
metadata:
  name: cluster-app-secrets
spec:
  externalSecretName: app-secrets
  namespaceSelector:
    matchLabels:
      eso-inject: "true"
  refreshTime: "1h"
```

## Template Engine (v2)

ESO's template engine uses Go `text/template`. Applied via `spec.target.template`:

```yaml
spec:
  target:
    template:
      type: kubernetes.io/tls
      data:
        tls.crt: "{{ .crt | base64decode }}"
        tls.key: |
          {{ .key | base64decode | strings.TrimSuffix "\n" }}
```

Available functions:

| Function | Description |
|----------|-------------|
| `base64decode` / `base64encode` | Base64 encode/decode |
| `upper` / `lower` | String casing |
| `trim` / `trimSuffix` / `trimPrefix` | String trimming |
| `replaceAll` | String replacement |
| `toJSON` / `fromJSON` | JSON marshal/unmarshal |
| `keys` / `values` | Map key/value iteration |
| `toString` / `toBytes` | Type conversion |

**mergePolicy:** Controls how template data merges with secret data:

| Policy | Behavior |
|--------|----------|
| `Replace` | Template output entirely replaces fetched data (default) |
| `Merge` | Template keys override fetched keys, others pass through |

## Refresh Policies

| Policy | Behavior |
|--------|----------|
| `Periodic` | Re-fetch at `refreshInterval` (default `1h`) |
| `CreatedOnce` | Fetch once at creation, never refresh |
| `OnChange` | Watch the provider for changes (provider-dependent) |

## Helm Values

| Value | Default | Description |
|-------|---------|-------------|
| `installCRDs` | `true` | Install CRDs (must be true for Helm-managed) |
| `replicaCount` | `1` | Controller replicas |
| `serviceMonitor.enabled` | `false` | Prometheus ServiceMonitor |
| `serviceMonitor.interval` | `30s` | Scrape interval |
| `webhook.port` | `9443` | Webhook server port |
| `webhook.certManager.enabled` | `false` | Use cert-manager for webhook cert |
| `webhook.certManager.issuerRef` | — | Issuer reference (ClusterIssuer or Issuer) |
| `certController.enabled` | `true` | Enable cert controller (disable if cert-manager used) |
| `certController.replicaCount` | `1` | Cert controller replicas |
| `concurrent` | `1` | Max concurrent reconcilers |
| `leaderElect` | `true` | Leader election for HA |
| `vault.enableTokenCache` | `true` | Cache Vault tokens (disable for short-lived token testing) |
| `scopedRBAC` | `false` | Restrict RBAC per SecretStore namespace |
| `controllerClass` | — | Controller class filter (multi-controller setups) |
| `crds.concurrency` | `10` | CRD patch concurrency |
| `podDisruptionBudget.enabled` | `false` | PDB for HA |
| `global.imagePullSecrets` | `[]` | Image pull secrets |

## Metrics

ESO exposes Prometheus metrics on port `8080` (default):

| Metric | Type | Description |
|--------|------|-------------|
| `externalsecret_sync_calls_total` | Counter | Total sync calls by status (success/error) |
| `externalsecret_reconcile_duration_seconds` | Histogram | Reconcile duration |
| `externalsecret_ready_count` | Gauge | Ready ExternalSecrets count |
| `provider_api_calls_count` | Counter | Provider API calls by provider type |
| `secretstore_duration_seconds` | Histogram | SecretStore validation duration |

## Common Mistakes

- **KV v1 vs v2 path mismatch.** Vault KV v2 paths use `data/` prefix internally. ESO handles this when `version: v2` is set. Set `version: v1` for KV v1 engines. Wrong version = `404` errors.
- **K8s auth role without policy.** The role `eso-reader` must be created in Vault first with a policy attached. ESO errors with `permission denied` if the role exists but has no policy.
- **ServiceAccount in wrong namespace.** `serviceAccountRef.name` must exist in the ESO controller namespace (not the ExternalSecret namespace). Cross-namespace SA references require `serviceAccountRef.namespace`.
- **`refreshInterval: "CreatedOnce"` doesn't update.** If the provider value changes, ESO won't re-fetch. Use `Periodic` for dynamic secrets.
- **Template errors are silent.** A bad template renders the secret with empty values. Use `kubectl describe externalsecret` to check status conditions.
- **`creationPolicy: Orphan` on existing Secret.** ESO won't take ownership but also won't update it. The Secret becomes read-only from ESO's perspective.
- **ClusterSecretStore can reference any namespace SA.** Make sure the SA referenced has appropriate permissions. ClusterSecretStore is NOT namespace-scoped.
- **Metrics port conflicts.** If another service uses port 8080, set `metrics.service.port` or disable metrics.

## Deployment (Flux HelmRelease)

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: external-secrets
  namespace: external-secrets
spec:
  chart:
    spec:
      chart: external-secrets
      sourceRef:
        kind: HelmRepository
        name: external-secrets
      version: 2.6.0
  values:
    installCRDs: true
    serviceMonitor:
      enabled: true
    webhook:
      certManager:
        enabled: true
        issuerRef:
          kind: ClusterIssuer
          name: letsencrypt-production
    certController:
      enabled: false
    vault:
      enableTokenCache: false
    concurrent: 1
    leaderElect: true
```
