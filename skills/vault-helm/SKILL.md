---
name: vault-helm
description: Use when deploying, configuring, or upgrading HashiCorp Vault on Kubernetes via Helm. Covers HelmRelease values, HA+Raft, injector, storage, TLS, telemetry, post-deploy steps.
---

# Vault (Helm)

## Overview

HashiCorp Vault provides secret management, encryption as a service, and identity-based access. This skill covers deploying Vault on Kubernetes with the official Helm chart in HA+Raft mode.

**Latest:** chart 0.33.0, app 2.0.2.

**No CRDs.** Controlled via Helm values and server config.

> **⚠️ Prefer OpenBao for self-hosting** — OpenBao (v2.6.x, MPL-2.0, Linux Foundation) is a drop-in-compatible Vault fork with **namespaces (secure multi-tenancy + namespace sealing) built into the open-source build** — a feature Vault gates behind paid Enterprise (note: replication is NOT yet in OpenBao). Its **killer feature for home/unstable clusters is Static Key Auto-Unseal** (`seal "static"`, built-in, no KMS): a 32-byte AES-256-GCM key injected from your secret store at boot, so nodes unseal themselves after any restart/power loss — Vault OSS auto-unseal only works via cloud KMS or a transit peer, otherwise you're typing Shamir keys after every reboot. The `hashicorp/vault` Helm chart and this skill's values mostly apply to OpenBao deployments too (image swap + config parity; verify engine/auth parity before committing). See the `vault` router for the full comparison.

## Architecture (HA+Raft)

```
3 Vault pods (StatefulSet)
  → Raft integrated storage (consensus, no external DB)
    → Active node handles requests
      → Standby nodes forward to active
        → Service registration via K8s API
```

Vault OSS supports:
- Raft HA consensus (3+ nodes)
- Kubernetes auth method
- KV v2 secrets engine
- ACL policies
- Auto-unseal (KMS, Transit)
- Prometheus metrics

Vault OSS does **not** support:
- Namespaces (enterprise)
- Disaster Recovery replication (enterprise)
- Sentinel policies (enterprise)
- SAML auth (enterprise)
- OTLP traces (enterprise)

## Deployment (Flux HelmRelease)

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: vault
  namespace: vault
spec:
  chart:
    spec:
      chart: vault
      sourceRef:
        kind: HelmRepository
        name: hashicorp
      version: 0.33.0
  values:
    global:
      tlsDisable: true

    injector:
      enabled: false

    server:
      ha:
        enabled: true
        raft:
          enabled: true
          config: |
            ui = true

            listener "tcp" {
              address     = "0.0.0.0:8200"
              tls_disable = true
            }

            storage "raft" {
              path = "/vault/data"
            }

            service_registration "kubernetes" {}

            telemetry {
              prometheus_retention_time = "12h"
              disable_hostname         = true
            }

      service:
        enabled: true

      auditStorage:
        enabled: false

    ui:
      enabled: true
      serviceType: ClusterIP

    dataStorage:
      size: 10Gi
      storageClass: ceph-block
```

### Key Helm Values

| Value | Default | Description |
|-------|---------|-------------|
| `global.tlsDisable` | `true` | Disable TLS within cluster (Gateway terminates external TLS) |
| `global.externalVaultAddr` | — | Address for external Vault (injector-only) |
| `injector.enabled` | `true` | Enable sidecar injector (disable if using ESO) |
| `injector.replicas` | `1` | Injector replicas |
| `server.ha.enabled` | `false` | Enable HA mode |
| `server.ha.raft.enabled` | `false` | Use Raft integrated storage |
| `server.ha.raft.config` | — | Raw HCL config (listener, storage, telemetry, etc.) |
| `server.ha.raft.replicas` | `3` | Number of Raft nodes |
| `server.service.enabled` | `true` | Create K8s Service |
| `server.auditStorage.enabled` | `false` | Enable audit log storage |
| `server.dataStorage.enabled` | `true` | Persistent storage for Vault data |
| `server.dataStorage.size` | `10Gi` | PVC size |
| `server.dataStorage.storageClass` | — | Storage class |
| `ui.enabled` | `false` | Enable Vault UI |
| `ui.serviceType` | `ClusterIP` | UI service type |

### Telemetry / Monitoring

```yaml
telemetry:
  prometheus_retention_time = "12h"
  disable_hostname         = true
```

Vault exposes Prometheus metrics on port 8200 at `/v1/sys/metrics`. Scrape via VMAgent/Alloy using the `server.serviceTelemetry.prometheusOperator` option:

```yaml
server:
  serviceTelemetry:
    prometheusOperator: true
```

## Post-Deploy Steps

After the HelmRelease syncs, Vault must be initialized and unsealed:

### 1. Initialize

```bash
kubectl exec -n vault vault-0 -- vault operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -format=json > vault-keys.json
```

Save `vault-keys.json` securely (unseal keys + root token).

### 2. Unseal All Pods

```bash
# Unseal vault-0 (repeat for each key share, 3 of 5)
for i in $(seq 3); do
  KEY=$(cat vault-keys.json | jq -r ".unseal_keys_b64[$((i-1))]")
  kubectl exec -n vault vault-0 -- vault operator unseal "$KEY"
done

# Unseal vault-1 and vault-2 (same 3 keys)
for pod in vault-1 vault-2; do
  for i in $(seq 3); do
    KEY=$(cat vault-keys.json | jq -r ".unseal_keys_b64[$((i-1))]")
    kubectl exec -n vault "$pod" -- vault operator unseal "$KEY"
  done
done
```

### 3. Raft Join (vault-1 and vault-2)

```bash
ROOT_TOKEN=$(cat vault-keys.json | jq -r '.root_token')

for pod in vault-1 vault-2; do
  kubectl exec -n vault "$pod" -- sh -c "
    export VAULT_TOKEN=$ROOT_TOKEN
    vault operator raft join http://vault-0.vault-internal:8200
  "
done
```

### 4. Verify Cluster

```bash
kubectl exec -n vault vault-0 -- vault operator raft list-peers
kubectl exec -n vault vault-0 -- vault status
```

### 5. Login

```bash
kubectl exec -n vault vault-0 -- vault login "$ROOT_TOKEN"
```

## HA Config Reference

The `server.ha.raft.config` value accepts raw HCL. Key stanzas:

### Listener

```hcl
listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = true           # TLS terminated at Gateway; false for direct TLS
  # tls_cert_file = "/vault/userconfig/vault-tls/tls.crt"
  # tls_key_file  = "/vault/userconfig/vault-tls/tls.key"
}
```

### Storage (Raft)

```hcl
storage "raft" {
  path = "/vault/data"
  # node_id = "node1"           # auto-assigned in HA mode
  # performance_multiplier = 1  # tune Raft performance
}
```

### Service Registration

```hcl
service_registration "kubernetes" {
  # namespace = "vault"         # auto-detected from pod namespace
}
```

## Common Mistakes

- **Not all pods unsealed.** Vault requires a quorum of unsealed nodes (N/2 + 1). With 3 nodes, at least 2 must be unsealed. The active node handles requests; standbys forward.
- **Raft join skipped for non-initial pods.** Only `vault-0` initializes the cluster. `vault-1` and `vault-2` must `raft join` after unseal or they stay `standby` without cluster membership.
- **Unseal keys stored only in cluster.** If the keys are lost and all pods restart, Vault data is unrecoverable. Store unseal keys securely offline.
- **`tls_disable = true` exposes plaintext within cluster.** Only use when Gateway or service mesh terminates TLS externally. Never set `tls_disable = true` without network-level encryption between pods.
- **Data loss on PVC delete.** Raft storage uses the PVC. Deleting the StatefulSet PVCs destroys all Vault data. Backup via `vault operator raft snapshot save`.
- **Injector conflicts with ESO.** If using External Secrets Operator, disable the injector (`injector.enabled: false`). Both can conflict on pod mutation.
- **Chart version has breaking config changes.** The `server.ha.raft.config` raw HCL format changed across chart versions. Pin chart version and test upgrades.
- **UI returns 404 if not explicitly enabled.** `ui.enabled: false` by default. Set to `true` and check the service is accessible (ClusterIP + Gateway HTTPRoute).
