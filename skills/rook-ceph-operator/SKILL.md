---
name: rook-ceph-operator
description: Use when deploying or managing Rook-Ceph CRDs — CephCluster, CephBlockPool, CephFilesystem, CephObjectStore, CephNFS, CephRBDMirror, and supporting resources. Covers operator chart config, cluster spec, CSI settings, storage classes.
---

# Rook-Ceph Operator CRDs

Charts: `rook-ceph` + `rook-ceph-cluster` (from `https://charts.rook.io/release`) + `ceph-csi-drivers` (v1.20+, from `https://charts.ceph.io/ceph-csi-operator`)  
Latest: **v1.20.x** (charts) | Ceph: **v20.2.2** (Tentacle) | API: `ceph.rook.io/v1`  
**Install/upgrade order:** `rook-ceph` → `ceph-csi-drivers` → `rook-ceph-cluster`

> **⚠️ v1.20 breaking change:** CSI drivers are now managed by the separate `ceph-csi-drivers` Helm chart. Rook no longer creates CSI ServiceAccounts, RBAC, or Driver CRs. **Without this chart, all PVC provisioning breaks.** See [Upgrade Notes](#upgrade-notes-v119--v120) below.

## CRD Reference

| CRD | API | Purpose | Key Fields |
|-----|-----|---------|------------|
| **CephCluster** | `v1` | Core cluster CRD — defines monitors, OSDs, manager, network, storage topology. | `spec.mon.count`, `spec.mgr.count`, `spec.storage` (nodes[], useAllNodes, deviceFilter, config.deviceClass), `spec.cephVersion.image`, `spec.dataDirHostPath`, `spec.dashboard`, `spec.network`, `spec.placement` |
| **CephBlockPool** | `v1` | RBD block storage pool — creates Ceph pool + optional StorageClass. | `spec.failureDomain`, `spec.replicated.size`, `spec.deviceClass`, `spec.erasureCoded` (dataChunks, codingChunks), `spec.application` |
| **CephFilesystem** | `v1` | CephFS shared filesystem — metadata + data pools, MDS daemons. | `spec.metadataPool`, `spec.dataPools[]`, `spec.metadataServer.activeCount`, `spec.metadataServer.activeStandby` |
| **CephObjectStore** | `v1` | S3-compatible object store — RGW gateway, metadata + data pools. | `spec.metadataPool`, `spec.dataPool`, `spec.gateway.port`, `spec.gateway.instances`, `spec.gateway.placement` |
| **CephObjectStoreUser** | `v1` | S3 access key/secret for object store users. | `spec.store`, `spec.displayName` |
| **CephNFS** | `v1` | NFS export of CephFS filesystem. | `spec.rados.namespace`, `spec.server.active` |
| **CephRBDMirror** | `v1` | RBD mirroring daemon for cross-cluster replication. | `spec.count` |
| **CephFilesystemMirror** | `v1` | CephFS snapshot mirroring across clusters. | `spec.placement` (placement only) |
| **CephFilesystemSubVolumeGroup** | `v1` | Subvolume group for CephFS quota/UID/GID isolation. | `spec.filesystemName`, `spec.pinning`, `spec.quota` |
| **CephBucketTopic** | `v1` | S3 bucket notification topic (Kafka, AMQP, webhook). | `spec.endpoint`, `spec.persistent`, `spec.opaqueData` |
| **CephBucketNotification** | `v1` | Binds topic to object store events. | `spec.topic`, `spec.events[]`, `spec.filter` (key/metadata/tagFilters) |
| **CephClient** | `v1` | Ceph auth key management for external clients. | `spec.caps` |
| **CephBlockPoolRadosNamespace** | `v1` | Rados namespace within a block pool for tenant isolation. | `spec.blockPoolName`, `spec.mirroring` |
| **CephCOSIDriver** | `v1` | COSI (Container Object Storage Interface) driver. | Standard COSI config |

## Deployed Pattern

```yaml
# CephCluster — via rook-ceph-cluster chart values.cephClusterSpec
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  mon:
    count: 3
    allowMultiplePerNode: false
  mgr:
    count: 1
  dashboard:
    enabled: true
    ssl: false
  storage:
    useAllNodes: false
    useAllDevices: false
    nodes:
      - name: worker-01
        deviceFilter: ^vdb$
      - name: worker-02
        deviceFilter: ^vdb$
        config:
          deviceClass: nvme
          osdsPerDevice: "1"
      - name: worker-03
        deviceFilter: ^vdb$
        config:
          deviceClass: nvme
          osdsPerDevice: "1"
      - name: worker-04
        deviceFilter: ^nvme
        config:
          deviceClass: nvme
          osdsPerDevice: "1"
  skipUpgradeChecks: true
  dataDirHostPath: /var/lib/rook
---
# CephBlockPool — ceph-blockpool from chart
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ceph-blockpool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 2
  deviceClass: nvme
---
# StorageClass created automatically from CephBlockPool
# provisioner: rook-ceph.rbd.csi.ceph.com
# parameters:
#   clusterID: rook-ceph
#   imageFormat: "2"
#   imageFeatures: layering
```

## Quick Reference

| Need | CRD | Notes |
|------|-----|-------|
| Block storage (PVC) | CephBlockPool + StorageClass | `ceph.rook.io/v1` |
| Shared filesystem (RWX) | CephFilesystem + StorageClass | Requires MDS, CephFS CSI |
| S3-compatible object store | CephObjectStore + CephObjectStoreUser | RGW gateway, port 80 |
| NFS export of CephFS | CephNFS | NFS-Ganesha server |
| Cross-cluster RBD mirror | CephRBDMirror | For DR |
| CephFS snapshot mirror | CephFilesystemMirror | For DR |
| S3 bucket notifications | CephBucketTopic + CephBucketNotification | Kafka/AMQP/webhook |
| Tenant isolation in pool | CephBlockPoolRadosNamespace | Per-namespace RADOS namespace |

## ceph-csi-drivers Chart (v1.20+)

**CRITICAL:** Rook v1.20 delegates CSI driver management to the `ceph-csi-operator` sub-chart (bundled with `rook-ceph`) + the separate `ceph-csi-drivers` chart. The drivers chart creates the ServiceAccounts, RBAC, and `Driver` CRs that the operator needs.

**Always websearch** `rook/docs/v1.20/Helm-Charts/csi-drivers-chart/` for the latest recommended `values.yaml` before installing.

### Installation

```bash
helm repo add ceph-csi-operator https://charts.ceph.io/ceph-csi-operator
helm install ceph-csi-drivers ceph-csi-operator/ceph-csi-drivers \
  --namespace rook-ceph \
  -f https://raw.githubusercontent.com/rook/rook/v1.20.2/deploy/charts/ceph-csi-drivers/values.yaml
```

### Critical: Driver Names Must Match StorageClasses

The `ceph-csi-drivers` chart defaults to driver names like `rbd.csi.ceph.com`, but existing StorageClasses reference `rook-ceph.rbd.csi.ceph.com` (prefixed with the operator namespace). **Set the correct names:**

```yaml
drivers:
  rbd:
    enabled: true
    name: rook-ceph.rbd.csi.ceph.com     # Must match StorageClass provisioner
  cephfs:
    enabled: true
    name: rook-ceph.cephfs.csi.ceph.com   # Must match StorageClass provisioner
  nfs:
    enabled: false
  nvmeof:
    enabled: false
```

Wrong names → StorageClass provisioner mismatch, volumes fail to provision.

## Upgrade Notes (v1.19 → v1.20)

### Breaking: CSI drivers moved to separate chart

In v1.19, Rook managed CSI drivers internally. In v1.20, a new `ceph-csi-operator` manages them via `Driver` CRs, and the `ceph-csi-drivers` Helm chart is **required** to create the backing ServiceAccounts + RBAC.

**Installation order:** `rook-ceph` → `ceph-csi-drivers` → `rook-ceph-cluster`

### Known Issues (websearch before deploying)

1. **v1.20.0: Missing SA/RBAC** — Without `ceph-csi-drivers` chart, ctrlplugin Deployments stuck at 0/2 with `serviceaccount not found`. Fixed by installing the drivers chart with correct driver names. (rook#17644)

2. **v1.20.1: Leader election broken** — RoleBindings only included prefixed SA names, but pods use unprefixed names (`rbd-ctrlplugin-sa`). Provisioner can't acquire lease → PVCs hang. Fixed in v1.20.2 or patch RoleBindings to add unprefixed SAs. (rook#17787)

3. **Helm upgrade conflicts** — Server-side apply may conflict with existing `Driver`/`OperatorConfig` CRs managed by Rook. Add `--force-conflicts` or delete old CRs before upgrade.

4. **Ceph v20.2.0 read-affinity bug** — Use v20.2.1+ or v20.2.2.

### Migration Steps

1. Upgrade `rook-ceph` chart to v1.20 (this deploys ceph-csi-operator sub-chart)
2. Retrieve existing CSI settings from your `rook-ceph-operator-config` ConfigMap
3. Install `ceph-csi-drivers` chart with driver names matching your StorageClasses
4. If upgrade fails with apply conflicts, use `helm upgrade --force-conflicts`
5. Verify: `kubectl get deploy -n rook-ceph -l app.kubernetes.io/name=rbd` shows ctrlplugin + nodeplugin running
6. Upgrade `rook-ceph-cluster` chart

### Config Changes

- CSI settings removed from `rook-ceph-operator-config` ConfigMap — now managed via `OperatorConfig` + `Driver` CRs
- CSI custom images remain in `rook-ceph` chart values (`csi.csiOperator.controllerManager.manager.env`)
- `rookUseCsiOperator: true` is the new default in v1.20

## Common Mistakes

- **deviceFilter regex** — `^vdb$` matches exactly `vdb`, `^nvme` matches any nvme device. Wrong regex = no OSDs created.
- **failureDomain: osd vs host** — `osd` allows OSDs on same node, `host` spreads across nodes. Use `host` for production.
- **CephFS `dataPools[].name`** — Required when multiple data pools. First pool is default.
- **CephFS preservePoolsOnDelete: false** — Deleting the CRD deletes ALL data. Set `true` for safety.
- **ObjectStore gateway placement** — RGW must be scheduled on a node with OSDs or have network access to them.
- **CephBlockPool size: 1** — No redundancy. Data lost if OSD fails. Only for test/dev or slow tier with HDD.
- **OSD per device filter** — `osdsPerDevice: "1"` creates one OSD per device. Default may create too many.
- **dashboard ssl: false** — Only for internal clusters. Enable SSL (`ssl: true`) for external access.
- **Insufficient mon memory** — MONS need minimum 512Mi RAM. Starving mons causes cluster instability.
