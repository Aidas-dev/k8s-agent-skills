---
name: dragonfly
description: Use when working with DragonflyDB operator on Kubernetes — creating or troubleshooting Dragonfly resources, configuring replication, snapshots, TLS, authentication, or affinity/scheduling for Dragonfly instances.
---

# DragonflyDB Operator

## Overview

Dragonfly operator manages Dragonfly (Redis-compatible) in-memory data store instances on Kubernetes. API: `dragonflydb.io/v1alpha1`. Single CRD: `Dragonfly` (plural: `dragonflies`). Deployed via Helm chart `oci://ghcr.io/dragonflydb/dragonfly-operator/helm/dragonfly-operator`. Latest operator: **v1.5.0** (Mar 2026). Deployed chart: v1.5.0.

## CRD Fields

| Field | Type | Since | Description |
|-------|------|-------|-------------|
| `replicas` | int | — | Total instances (1 = standalone primary, 2 = 1 primary + 1 replica) |
| `image` | string | — | Dragonfly image (default: `docker.dragonflydb.io/dragonflydb/dragonfly:v1.21.2`) |
| `args` | []string | — | Dragonfly server args (e.g. `--maxmemory=2gb`) |
| `resources` | ResourceRequirements | — | Container CPU/memory |
| `affinity` | Affinity | — | Pod affinity (nodeAffinity, podAntiAffinity, etc.) |
| `nodeSelector` | map | v1.1.1 | Node selector for pod scheduling |
| `tolerations` | []Toleration | — | Pod tolerations |
| `topologySpreadConstraints` | []TopologySpreadConstraint | v1.1.1 | Spread pods across topology domains |
| `annotations` | object | — | Annotations on Dragonfly pods |
| `labels` | object | — | Labels on Dragonfly pods |
| `env` | []EnvVar | — | Environment variables |
| `authentication.passwordFromSecret` | SecretKeySelector | — | Password from Secret key |
| `authentication.clientCaCertSecret` | SecretReference | — | Client CA certificate Secret |
| `tlsSecretRef` | SecretReference | — | TLS cert Secret for server TLS |
| `snapshot.cron` | string | — | Cron schedule for snapshots (e.g. `"\*/5 * * * *"`) |
| `snapshot.persistentVolumeClaimSpec` | PVC Spec | — | PVC for snapshot storage |
| `aclFromSecret` | SecretKeySelector | v1.1.1 | ACL file from Secret |
| `serviceAccountName` | string | — | Pod service account |
| `serviceSpec.type` | string | — | Service type (ClusterIP, LoadBalancer, etc.) |
| `serviceSpec.name` | string | v1.1.3 | Custom service name |
| `serviceSpec.annotations` | object | — | Service annotations |
| `priorityClassName` | string | v1.1.1 | Pod priority class |
| `skipFSGroup` | bool | v1.1.2 | Skip FSGroup assignment (OpenShift) |
| `memcachedPort` | int | v1.1.2 | Memcached port (alternative to `--memcached_port` arg) |
| `additionalContainers` | []Container | — | Sidecar containers |
| `additionalVolumes` | []Volume | — | Extra volumes |

## Dragonfly Spec Patterns

```yaml
apiVersion: dragonflydb.io/v1alpha1
kind: Dragonfly
metadata:
  name: my-cache
spec:
  replicas: 1                    # 1 = standalone primary
  args:
    - --maxmemory=2gb
    - --logtostderr
    - --cluster_mode=emulated    # Enable cluster-compatible mode
    - --lock_on_hashtags         # Hashtag-based locking
    - --default_lua_flags=allow-undeclared-keys

  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: "1"
      memory: 2Gi

  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                  - worker-proxmox

  authentication:
    passwordFromSecret:
      name: dragonfly-password
      key: password

  snapshot:
    cron: "*/5 * * * *"
    persistentVolumeClaimSpec:
      accessModes:
        - ReadWriteOnce
      storageClassName: ceph-block
      resources:
        requests:
          storage: 2Gi

  serviceSpec:
    type: ClusterIP
    annotations:
      external-dns.alpha.kubernetes.io/hostname: dragonfly.example.com
```

## Replication Model

- `replicas=1` — standalone primary
- `replicas=2` — 1 primary + 1 replica
- `replicas=3` — 1 primary + 2 replicas
- **Always exactly 1 primary** regardless of replica count
- Operator manages automatic failover if primary fails
- Service `<name>.<ns>.svc.cluster.local` always points to current primary

## Authentication

| Method | Config | Description |
|--------|--------|-------------|
| Password | `authentication.passwordFromSecret` | Basic password auth (maps to `--requirepass`) |
| Client CA | `authentication.clientCaCertSecret` | TLS client cert verification |
| ACL file | `aclFromSecret` | ACL rules file from Secret (v1.1.1+) |

Password can also be set via `args: ["--requirepass=<pw>"]` or `env: [{name: DFLY_requirepass, value: "<pw>"}]`.

## TLS

```yaml
spec:
  tlsSecretRef:
    name: dragonfly-tls    # Secret must have tls.crt, tls.key
```

Secret must exist in same namespace. Optionally combine with `authentication.clientCaCertSecret` for mutual TLS.

## Snapshots

Snapshots store Dragonfly data to PVC for persistence across restarts:

```yaml
spec:
  snapshot:
    cron: "*/5 * * * *"
    persistentVolumeClaimSpec:
      storageClassName: ceph-block
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 2Gi
```

- `cron` is optional — omit for on-demand only
- Snapshots **not auto-pruned** — manage disk or use static `--dbfilename` to overwrite
- PVC spec follows standard Kubernetes `PersistentVolumeClaimSpec`

## Exposing Dragonfly as Cache

Applications connect via the service at `<name>.<ns>.svc.cluster.local:6379`. In apps using Dragonfly as Redis-compatible cache (valkey/Redis clients):

```yaml
# In the app's ConfigMap or env
REDIS_URL: redis://:${REDIS_PASSWORD}@my-cache.namespace:6379
REDIS_PASSWORD: # from Dragonfly auth secret
```

For cluster-mode emulated (`--cluster_mode=emulated`), Redis cluster clients (e.g. `ioredis` cluster mode) can connect as if it's a Redis Cluster — but there's only one actual Dragonfly instance behind the service.

## Monitoring

Dragonfly exposes metrics on the `admin` port. Use a `PodMonitor` to scrape:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: dragonfly-monitor
spec:
  selector:
    matchLabels:
      app: my-cache               # Must match Dragonfly resource name
  podMetricsEndpoints:
    - port: admin
```

The operator creates pods with label `app: <dragonfly-name>` by default.

## Common Mistakes

- **`replicas` confusion** — replicas=2 means 1 primary + 1 replica, NOT 2 primaries
- **No snapshot cron** — without `cron` + PVC, restarts lose all data; always configure for stateful use
- **`cluster_mode=emulated` without `lock_on_hashtags`** — emulated cluster needs hashtag locking for multi-key ops
- **Password collision** — don't set password via both `authentication.passwordFromSecret` AND `--requirepass` arg; use the CRD field
- **Same storageClass for all** — immich with ceph-block fine; if latency-sensitive, use local SSD via `nodeSelector` + local storage
- **No resource limits** — Dragonfly can OOM under load; always set `resources.limits.memory`
- **Helm chart OCI URL** — use `oci://ghcr.io/dragonflydb/dragonfly-operator/helm`, chart name `dragonfly-operator`, version `v1.5.0`

## Version History

| Operator | Helm Chart | Date | Notes |
|----------|-----------|------|-------|
| v1.5.0 | v1.5.0 | Mar 2026 | Latest (deployed) |
| v1.4.0 | v1.4.0 | Jan 2026 | |
| v1.3.1 | v1.3.1 | Nov 2025 | |
