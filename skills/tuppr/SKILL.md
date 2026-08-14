---
name: tuppr
description: Tuppr — Kubernetes controller for automated, orchestrated Talos and Kubernetes upgrades. Declare a target version in a CR; tuppr drains, upgrades, reboots, and health-checks nodes. Covers TalosUpgrade and KubernetesUpgrade CRDs, coordination, health checks, maintenance windows.
---

# tuppr — Talos & Kubernetes Upgrade Controller

Automated, GitOps-friendly upgrades of **Talos Linux** and **Kubernetes**. Declare a target version in a custom resource; tuppr plans and executes the rollout — draining, upgrading, rebooting, and health-checking each node (sequentially or in parallel batches). Upgrades always run from a healthy node — tuppr **never self-upgrades the node it runs on**.

**Repo:** github.com/home-operations/tuppr  
**API group:** `tuppr.home-operations.com/v1alpha1`  
**Latest chart:** 0.5.0 (OCI `ghcr.io/home-operations/charts/tuppr`)  
**Docs:** tuppr.home-operations.com

## When to Use tuppr

- You want Talos/K8s upgrades driven by the **Kubernetes API** (GitOps) instead of running `talosctl upgrade` by hand
- You manage nodes with Terraform but want **upgrades owned separately** (see `talos-terraform` — tuppr is the "dedicated upgrade system" that pattern assumes)

For manual upgrades → `talosctl`. For machine config → `talosconfig`. This skill is only the controller.

## Install

```bash
# Direct
helm install tuppr oci://ghcr.io/home-operations/charts/tuppr \
  --version 0.5.0 \
  --namespace system-upgrade --create-namespace
```

GitOps (Flux) — the recommended production path:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: OCIRepository
metadata:
  name: tuppr
  namespace: system-upgrade
spec:
  interval: 15m
  url: oci://ghcr.io/home-operations/charts/tuppr
  layerSelector:
    mediaType: application/vnd.cncf.helm.chart.content.v1.tar+gzip
    operation: copy
  ref:
    tag: 0.5.0
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: tuppr
  namespace: system-upgrade
spec:
  chartRef:
    kind: OCIRepository
    name: tuppr
  interval: 1h
  values:
    replicaCount: 2
    monitoring:
      serviceMonitor:
        enabled: true
```

### Prerequisites

- The **Talos API-access CRD** (a real Talos cluster)
- `talosServiceAccount.create: true` (default) — makes Talos generate the `*-talosconfig` Secret the controller mounts. On non-Talos clusters (e2e/kind), set false and provide the Secret yourself
- Leader election recommended (default enabled; only one controller active)

## Resources

| Resource | Upgrades | Reboot | Per cluster |
|----------|----------|--------|-------------|
| `TalosUpgrade` | Talos Linux on nodes | Yes | Many (queued, node-selectable) |
| `KubernetesUpgrade` | Kubernetes version | No | Exactly one (webhook-enforced) |

**Only one upgrade runs at a time cluster-wide** — TalosUpgrade plans queue FCFS, and the two kinds never run concurrently.

## TalosUpgrade

```yaml
apiVersion: tuppr.home-operations.com/v1alpha1
kind: TalosUpgrade
metadata:
  name: cluster
spec:
  talos:
    # renovate: datasource=docker depName=ghcr.io/siderolabs/installer
    version: v1.13.8            # required — target Talos version

  policy:
    debug: false                # verbose logging
    force: false                # skip etcd health checks
    rebootMode: default         # default | powercycle
    placement: soft             # hard | soft
    stage: false                # stage upgrade
    timeout: 30m                # per-node upgrade timeout

  healthChecks:                 # CEL-based, run before each node
    - apiVersion: v1
      kind: Node
      expr: status.conditions.exists(c, c.type == "Ready" && c.status == "True")
    - apiVersion: ceph.rook.io/v1
      kind: CephCluster
      expr: status.ceph.health in ['HEALTH_OK']

  nodeSelector:                 # scope which nodes this plan targets
    matchExpressions:
      - {key: tuppr.home-operations.com/upgrade, operator: In, values: ["enabled"]}
      - {key: node-role.kubernetes.io/control-plane, operator: DoesNotExist}

  parallelism: 1                # 1 = sequential; >1 = batched concurrent nodes

  drain:                        # pod eviction options
    deleteLocalData: true
    ignoreDaemonSets: true
    force: true

  maintenance:                  # only run inside windows
    windows:
      - start: "0 2 * * 0"      # cron — Sunday 02:00
        duration: "4h"
        timezone: "UTC"

  talosctl:
    image:
      repository: ghcr.io/siderolabs/talosctl
      tag: v1.13.8              # auto-detected if omitted
      pullPolicy: IfNotPresent
```

## KubernetesUpgrade

```yaml
apiVersion: tuppr.home-operations.com/v1alpha1
kind: KubernetesUpgrade
metadata:
  name: kubernetes
spec:
  kubernetes:
    # renovate: datasource=docker depName=ghcr.io/siderolabs/kubelet
    version: v1.36.3            # required — target Kubernetes version
    # imageRepository: registry.example.com/k8s   # optional private mirror

  healthChecks:
    - apiVersion: v1
      kind: Node
      expr: status.conditions.exists(c, c.type == "Ready" && c.status == "True")
      timeout: 10m
```

**One per cluster** — admission webhook rejects additional resources. To upgrade again, edit `spec.kubernetes.version`. History in `.status.history[]` (capped at 10, newest first).

## Operations

```bash
kubectl get talosupgrade -w          # watch upgrade progress
kubectl get kubernetesupgrade -w
kubectl describe talosupgrade cluster      # detailed status + events
kubectl describe kubernetesupgrade kubernetes
```

```bash
# Suspend/resume
kubectl annotate talosupgrade cluster tuppr.home-operations.com/suspend="true"
kubectl annotate kubernetesupgrade kubernetes tuppr.home-operations.com/suspend="true"
```

Per-node overrides via node annotations: unique version or schematic for specific nodes.

## Version History Notes (0.4.x → 0.5.x)

- Talos 1.14 upgrades supported end-to-end
- Pre-pulls the Talos installer image before starting a run
- Lease-based Alertmanager silence during upgrade runs (0.4.1)
- Schematic annotation overrides runtime schematic

## Common Mistakes

- **Two KubernetesUpgrade resources** — Webhook rejects; only one per cluster.
- **Upgrade + config change simultaneously** — tuppr and `talos_machine_configuration_apply` (TF) both touch nodes. Sequence them; tuppr's health checks catch mid-upgrade config drift.
- **No health checks on critical clusters** — Add CEL checks (Ceph health, backup sync status) so tuppr aborts before draining with an unhealthy cluster.
- **Forgetting `--namespace`** — The controller watches its namespace; CRs must land where tuppr watches.
- **`force: true` as habit** — Skips etcd health checks; data-loss risk. Only for broken-cluster recovery.
- **Overwriting the `*-talosconfig` Secret** — tuppr mounts it; regenerating via `talosServiceAccount` rotation invalidates the controller mid-run.
