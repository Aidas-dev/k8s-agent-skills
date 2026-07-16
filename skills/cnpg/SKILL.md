---
name: cnpg
description: Use when working with CloudNativePG (CNPG) for PostgreSQL on Kubernetes — creating or troubleshooting Cluster, Backup, ScheduledBackup, Pooler, or ImageCatalog resources; defining bootstrap, storage, backup, or affinity settings.
---

# CloudNativePG (CNPG)

## Overview

CloudNativePG operator manages HA PostgreSQL clusters on Kubernetes. API: `postgresql.cnpg.io/v1`. Deployed via Helm chart (latest: 0.28.2 \u2192 operator 1.29.1).

## CRD Reference

| CRD | Purpose |
|-----|---------|
| `Cluster` | HA PostgreSQL cluster (core resource) |
| `Backup` | On-demand backup |
| `ScheduledBackup` | Cron-based backup schedule |
| `Pooler` | PgBouncer connection pooler |
| `Database` | Database-level management within cluster |
| `Publication` / `Subscription` | Logical replication |
| `ImageCatalog` / `ClusterImageCatalog` | Extension image management (v1.29+) |
| `FailoverQuorum` | Quorum-based failover settings |

## Cluster Spec (Key Fields)

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
spec:
  instances: 1                    # Replicas (1 = standalone primary)
  imageName: ghcr.io/cloudnative-pg/postgresql:18

  storage:
    size: 10Gi
    storageClass: ceph-block      # Omit or null for cluster default
    resizePVC: true               # Allow storage resize via Helm upgrade

  walStorage:                     # Optional \u2014 separate WAL volume
    size: 5Gi
    storageClass: ceph-block

  bootstrap:
    initdb:
      database: app               # Default DB name
      owner: app                  # Default owner user
      postInitSQL:                # Run after init
        - CREATE EXTENSION vector;

  affinity:
    nodeSelector:
      kubernetes.io/hostname: worker-01
    podAntiAffinity:
      type: preferred             # required or preferred
      topologyKey: kubernetes.io/hostname

  postgresql:
    parameters:
      shared_preload_libraries: vector  # Extensions
    pg_hba:                             # Custom pg_hba (v1.29+ use podSelectorRefs)

  primaryUpdateStrategy: unsupervised   # unsupervised (default) or switchover
  primaryUpdateMethod: restart          # restart or switchover

  monitoring:
    enablePodMonitor: true              # Generates Prometheus PodMonitor

  backup:
    volumeSnapshot:
      snapshotClass: csi-ceph-block     # CSI snapshot class
```

## Bootstrap Methods

| Method | Use case |
|--------|----------|
| `initdb` | Fresh cluster from scratch |
| `pg_basebackup` | Clone from existing CNPG cluster |
| `recovery` | PITR from Barman/volumeSnapshot/plugin backup |
| `pg_basebackup` via `recovery` | Bootstrap replica from another cluster |

## Backup Methods

| Method | Status | Details |
|--------|--------|---------|
| `barmanObjectStore` | **Deprecated** (removed v1.30) | S3/GCS/Azure via Barman Cloud |
| `volumeSnapshot` | Current | CSI snapshots (recommended for Ceph, EBS, etc.) |
| `plugin` | Current | CNPG-I gRPC plugin for backup/WAL/recovery |

## Affinity / Scheduling Patterns

- **nodeSelector** — Pin to specific node (common for stateful workloads)
- **podAntiAffinity** — Spread replicas across nodes (`required` for HA, `preferred` for soft)
- **topologySpreadConstraints** — Distribute across zones
- **tolerations** — For tainted nodes (GPU, infra-only)

Set per-instance overrides via `instances` + `nodeSelector` to pin specific roles.

## ImageCatalog (v1.29+)

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: ClusterImageCatalog
metadata:
  name: postgres-extensions
spec:
  images:
    - image: ghcr.io/my-org/postgres:18-pgvector
      major: 18
```

Then reference in Cluster: `spec.imageCatalogRef: postgres-extensions`

## Common Mistakes

- **BarmanObjectStore as default** \u2014 deprecated, use `volumeSnapshot` or `plugin`
- **No WAL storage** \u2014 important for HA. Add `walStorage` block for performance
- **`resizePVC: false`** \u2014 prevents storage expansion on Helm upgrade
- **`primaryUpdateStrategy: supervised`** \u2014 requires manual approval for switchover; use `unsupervised` for automatic
- **Same storageClass for all workloads** \u2014 ceph-block fine for most, but immich needs local-fast for vector extension performance
- **Missing `INHERITED_ANNOTATIONS` / `INHERITED_LABELS`** \u2014 empty is fine; only set if you want annotations propagated to PVCs

## Version Mapping

| Helm Chart | Operator | Release Notes |
|------------|----------|---------------|
| 0.28.2 | 1.29.1 | Latest (CVE fixes) |
| 0.28.0 | 1.29.0 | ImageCatalog, podSelectorRefs, CNPG-I plugins |

Check `helm list -n <ns>` for deployed chart version; cross-reference with [cloudnative-pg.io/release-notes](https://cloudnative-pg.io/release-notes/).
