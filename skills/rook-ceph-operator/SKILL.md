---
name: rook-ceph-operator
description: Use when deploying or managing Rook-Ceph CRDs — CephCluster, CephBlockPool, CephFilesystem, CephObjectStore, CephNFS, CephRBDMirror, and supporting resources. Covers operator chart config, cluster spec, CSI settings, storage classes.
---

# Rook-Ceph Operator CRDs

Charts: `rook-ceph` + `rook-ceph-cluster` from `https://charts.rook.io/release`  
Latest: **v1.19.5** (charts) | Ceph: **v20.2.1** (Tentacle) | API: `ceph.rook.io/v1`  
**Always upgrade in order:** `rook-ceph` → `ceph-csi-driver` (v1.20+) → `rook-ceph-cluster`

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

## Operator CSI Settings

```yaml
# In rook-ceph helm chart values
csi:
  enableRbdDriver: true
  enableCephfsDriver: true
  enableRBDSnapshotter: true
  provisionerReplicas: 2
  enableCSIHostNetwork: true  # needed if SDN blocks external clusters
  topology:
    enabled: false
  rbdFSGroupPolicy: "File"
  cephFSFSGroupPolicy: "File"
  csiRBDPluginResource: |
    - name: driver-registrar
      resource:
        requests:
          memory: 32Mi
          cpu: 5m
    - name: csi-rbdplugin
      resource:
        requests:
          memory: 128Mi
          cpu: 20m
```

## Upgrade Notes (v1.19 → v1.20)

- v1.20 requires `ceph-csi-driver` subchart between rook-ceph and rook-ceph-cluster
- Ceph v20.2.0 has a read-affinity corruption bug — use v20.2.1+
- Rook v1.20 changes Ceph image config format (separate repo + tag fields)
- Always upgrade operator before cluster chart

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
