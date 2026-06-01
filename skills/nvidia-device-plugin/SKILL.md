---
name: nvidia-device-plugin
description: Use when working with NVIDIA GPU support on Kubernetes — device plugin, GPU Feature Discovery, Node Feature Discovery, CDI, MIG, time-slicing. Covers Helm chart, values, configuration. No CRDs (ConfigMap-based).
---

# NVIDIA Device Plugin

## Overview

NVIDIA device plugin exposes GPUs to Kubernetes workloads. It runs as a DaemonSet on GPU nodes, discovers NVIDIA GPUs via NVML, and advertises them as allocatable resources (`nvidia.com/gpu`). Includes GPU Feature Discovery (GFD) for node labelling and optional Node Feature Discovery (NFD) for hardware detection.

**No CRDs.** ConfigMap-based configuration.

**Latest:** chart 0.19.2, app v0.19.2.

## Architecture

```
node-feature-discovery (NFD) worker
  → detects PCI devices (class 02/03 = GPU)
    → labels node: nvidia.com/gpu.present=true

gpu-feature-discovery (GFD) daemon
  → queries NVML for GPU details
    → labels node: nvidia.com/gpu.product, nvidia.com/gpu.memory, etc.
    → labels node: nvidia.com/gpu.present=true

k8s-device-plugin daemon
  → watches for pods requesting nvidia.com/gpu
    → mounts GPU devices + drivers into containers
    → generates CDI specs (optional)
```

## Quick Start

Add annotation to any pod that needs GPU:

```yaml
spec:
  runtimeClassName: nvidia
  containers:
    - resources:
        limits:
          nvidia.com/gpu: 1
```

Runtime class `nvidia` must be configured on the node (Talos: `.spec.runtime.runtimes.nvidia` in machine config).

## Key Features

| Feature | Enabled | Config |
|---------|---------|--------|
| GPU device discovery | Always | NVML |
| GPU Feature Discovery (GFD) | ✅ (`gfd.enabled: true`) | Labels nodes with GPU properties |
| Node Feature Discovery (NFD) | ✅ (`nfd.enabled: true`) | Detects PCI GPUs, labels nodes |
| CDI support | ✅ | `cdi.nvidiaHookPath`, `cdi.featureFlags` |
| Config-manager | ✅ | Dynamic config via node labels |
| MIG (Multi-Instance GPU) | Optional | `migStrategy` flag |
| Time-slicing | Optional | Config file `config.name` |
| Health checks | Always | Xid detection, device health |

## Helm Values

### Core Settings

| Value | Default | Description |
|-------|---------|-------------|
| `image.repository` | `nvcr.io/nvidia/k8s-device-plugin` | Image repo |
| `image.tag` | latest | Image tag |
| `runtimeClassName` | `nvidia` | Runtime class for GPU pods |
| `priorityClassName` | — | Pod priority |
| `failOnInitError` | `true` | Fail if no GPU found at startup |
| `deviceListStrategy` | `envvar` | How device IDs are passed to container |
| `deviceIDStrategy` | `uuid` | How devices are identified |
| `nvidiaDriverRoot` | `/` | Host driver root path |

### GFD (GPU Feature Discovery)

```yaml
gfd:
  enabled: true
  nameOverride: gpu-feature-discovery
  noTimestamp: true
  sleepInterval: 60s
  securityContext:
    privileged: true  # Required for NVML access
```

GFD labels the node with GPU properties (product name, memory, CUDA version, driver version). These labels are used by NFD to set `nvidia.com/gpu.present`.

### NFD (Node Feature Discovery)

```yaml
nfd:
  enabled: true
  enableNodeFeatureApi: false  # Use legacy label-based NFD
  master:
    config:
      extraLabelNs: ["nvidia.com"]
  worker:
    config:
      sources:
        pci:
          deviceClassWhitelist: ["02", "03"]  # Display controllers + 3D controllers
          deviceLabelFields: ["vendor"]
```

NFD worker detects PCI devices with class 02 (display) or 03 (3D) from NVIDIA vendor and labels the node. GFD then adds detailed GPU labels.

### Node Selection & Tolerations

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: nvidia.com/gpu.present
              operator: In
              values: ["true"]

tolerations:
  - key: CriticalAddonsOnly
    operator: Exists
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
```

### CDI (Container Device Interface)

```yaml
cdi:
  nvidiaHookPath: null     # Path to nvidia-cdi-hook/nvidia-ctk on host
  featureFlags: null       # Enable/disable specific CDI features
```

CDI generates device specifications so containers can use GPUs without privileged mode. When enabled, the device plugin adds `cdi.k8s.io/*` annotations to pods.

### MIG (Multi-Instance GPU)

Configured via CLI flags (not in values.yaml — use `config.map` or CLI args).

MIG strategies:
- `none` — no MIG (default)
- `single` — expose all MIG devices as single resources
- `mixed` — expose MIG devices individually

### Time-Slicing

GPU sharing by time-slicing. Configured via ConfigMap:

```yaml
config:
  name: ""     # ConfigMap name
  map: {}      # Inline config
  default: ""  # Default config name
  fallbackStrategies: ["named", "single"]
```

### Update Strategy

```yaml
updateStrategy:
  type: RollingUpdate
```

## Deployment (Flux HelmRelease)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: nvidia
  namespace: gpu-operator
spec:
  interval: 24h
  url: https://nvidia.github.io/k8s-device-plugin
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: nvidia-device-plugin
  namespace: gpu-operator
spec:
  chart:
    spec:
      chart: nvidia-device-plugin
      sourceRef:
        kind: HelmRepository
        name: nvidia
      version: 0.19.1
```

## Common Mistakes

- **NFD labels the node, not the device plugin.** NFD sets `nvidia.com/gpu.present=true` based on PCI device detection. If NFD is not deployed or fails, the device plugin won't schedule on the node. Check NFD pod logs.
- **GFD requires privileged access.** Without `securityContext.privileged: true`, GFD can't query NVML and won't label the node. The device plugin then won't detect GPUs.
- **`nvidia.com/gpu.present` label not set.** This label is set by NFD (not by GFD directly). GFD sets detailed labels (`nvidia.com/gpu.product`, etc.), but the presence label comes from NFD's PCI detection.
- **`runtimeClassName: nvidia` not set in workload.** Without the runtime class, the container won't have GPU devices mounted even if it requests `nvidia.com/gpu`.
- **`deviceListStrategy: envvar` is deprecated for CDI.** When using CDI, set `deviceListStrategy: cdi-annotations` to use CDI annotations instead of env vars.
- **MIG and time-slicing don't stack.** Time-slicing can share MIG devices but the interaction is complex. Test thoroughly.
- **NodeFeature API vs legacy mode.** With `enableNodeFeatureApi: false` (deployed), NFD uses labels directly. With the API enabled, NFD creates NodeFeature CRs. The device plugin doesn't care either way, but GFD must match.
- **Chart version doesn't match app version.** Chart 0.19.1 deploys device plugin v0.18.2. Always check `image.tag` separately.
