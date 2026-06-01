---
name: talos
description: Use when working with Talos Linux — deploying clusters, managing machine configs, troubleshooting nodes, upgrading Talos or Kubernetes, or working with talosctl CLI.
---

# Talos Linux (v1.13.x)

## Overview

Talos is an immutable Kubernetes OS — no SSH, no shell, API-driven. All management via `talosctl`. Current stable: **v1.13.x** (Linux 6.18, etcd 3.6.9, K8s 1.36). API version: `v1alpha1` (multi-document config format).

## Machine Configuration

Multi-document YAML — each document is a config type. All config is declarative, applied via `talosctl apply-config` or `talosctl patch`.

### Cluster Config Documents

| Document type | Kind | Purpose |
|---------------|------|---------|
| Machine config | `Config` | Core: machine type, install, network, disks |
| Environment config | `EnvironmentConfig` | Container env overrides (v1.13+, replaces `.machine.env`) |
| Install config | `InstallConfig` | Separate disk/install settings |
| Disk encryption config | `DiskEncryptionConfig` | LUKS2 encryption settings per partition |
| Network config | `NetworkConfig` | Interface config, DNS, NTP |
| KubeSpan config | `KubeSpanConfig` | WireGuard mesh settings |
| Image verification config | `ImageVerificationConfig` | Container signature verification (v1.13+) |
| External volume config | `ExternalVolumeConfig` | Virtiofs volumes (v1.13+) |
| Probe config | `ProbeConfig` | Node/API health probes (v1.13+) |
| Resolver config | `ResolverConfig` | DNS resolver config (v1.13+) |
| Routing rule config | `RoutingRuleConfig` | Advanced routing rules (v1.13+) |
| VRF config | `VRFConfig` | VRF interface config (v1.13+) |

Legacy single-document v1alpha1 format still supported.

### Key Config Fields

```yaml
# Cluster document example
kind: Config
version: v1alpha1
machine:
  type: controlplane          # controlplane, worker
  install:
    disk: /dev/sda            # Install target
    image: ghcr.io/siderolabs/installer:v1.13.0
    wipe: false               # Preserve on re-apply
  network:
    hostname: node-1
    interfaces:
      - interface: eno1
        dhcp: false
        addresses: [10.0.1.10/24]
        routes:
          - network: 0.0.0.0/0
            gateway: 10.0.1.1
    nameservers: [1.1.1.1]
  systemDiskEncryption:
    state:                    # Encrypts etcd partition
      provider: luks2
      keys:
        - slot: 0
          tpm: {}             # TPM 2.0 sealed key
    ephemeral:                # Encrypts ephemeral partition
      provider: luks2
      keys:
        - slot: 0
          tpm: {}
cluster:
  network:
    cni:
      name: none              # Install CNI separately
    podSubnets: [10.244.0.0/16]
    serviceSubnets: [10.96.0.0/12]
  apiServer:
    extraArgs:
      feature-gates: [ServerSideApply=true]  # Slice format in v1.13+
  etcd:
    extraArgs:
      quota-backend-bytes: "8589934592"      # 8GB
      auto-compaction-retention: "1000"
```

## CLI Quick Reference

### Cluster Lifecycle

| Command | Purpose |
|---------|---------|
| `talosctl gen config <name> <endpoint> --with-secrets secrets.yaml` | Generate cluster configs |
| `talosctl gen secrets --output-file secrets.yaml` | Generate secrets file (v1.13+) |
| `talosctl apply-config --nodes <ip> --file config.yaml --insecure` | Apply config to new node |
| `talosctl bootstrap --nodes <ip>` | Bootstrap etcd (once, first CP only) |
| `talosctl kubeconfig --nodes <ip> --force` | Get admin kubeconfig |
| `talosctl health --nodes <cps> --wait-timeout=10m` | Check cluster health |
| `talosctl version` | Show client + server versions |

### Maintenance

| Command | Purpose |
|---------|---------|
| `talosctl upgrade --image ghcr.io/siderolabs/installer:v1.13.1 --reboot-mode=default` | Upgrade Talos OS |
| `talosctl upgrade --stage` | Stage upgrade for next reboot |
| `talosctl upgrade-k8s --to v1.36.1 --dry-run` | Upgrade Kubernetes (dry-run first) |
| `talosctl upgrade-k8s --to v1.36.1` | Upgrade Kubernetes |
| `talosctl rollback --nodes <ip>` | Rollback Talos to previous install |
| `talosctl reboot --mode=default` | Reboot node |
| `talosctl reset --graceful --reboot` | Reset node (removes from cluster) |
| `talosctl apply-config --nodes <ip> --file patch.yaml` | Apply config change |
| `talosctl patch machineconfig --nodes <ip> --patch @patch.yaml` | Patch specific fields |

### Diagnostics

| Command | Purpose |
|---------|---------|
| `talosctl service` | List all services and status |
| `talosctl logs <service> --tail 100` | View logs (kubelet, etcd, containerd, etc.) |
| `talosctl dmesg --tail` | Kernel logs |
| `talosctl get machineconfig -o yaml` | Get current machine config |
| `talosctl get disks` | Disk information (`disks.block.talos.dev`) |
| `talosctl get encryptionsalts` | Encryption salt status |
| `talosctl get rd` | List all resource types |
| `talosctl etcd members` | List etcd cluster members |
| `talosctl etcd status` | etcd cluster status |
| `talosctl etcd alarm list` | Check etcd alarms |
| `talosctl etcd snapshot snapshot.db` | Backup etcd |
| `talosctl interfaces` | Network interfaces |
| `talosctl routes` | Routing table |
| `talosctl dashboard` | Real-time cluster dashboard |
| `talosctl support` | Dump debug archive |
| `talosctl netstat --tcp --listening` | Listening connections |

### Config Generation

| Flag | Purpose |
|------|---------|
| `--with-secrets secrets.yaml` | Use saved secrets (required for reproducibility) |
| `--config-patch @patch.yaml` | Patch all node types |
| `--config-patch-control-plane @cp.yaml` | Patch control planes only |
| `--config-patch-worker @w.yaml` | Patch workers only |
| `--additional-sans <dns>` | Extra SANs for API cert |
| `--dns-domain <domain>` | Cluster DNS domain (default: cluster.local) |
| `--kubernetes-version v1.36.0` | Override K8s version |

## Apply Modes

| Mode | Behavior |
|------|----------|
| `auto` (default) | Immediate apply, reboot if needed |
| `no-reboot` | Apply without reboot |
| `reboot` | Always reboot to apply |
| `staged` | Apply on next reboot |
| `try` | Apply, rollback after timeout if not confirmed |

## Reboot Modes

| Mode | Behavior |
|------|----------|
| `default` | Normal reboot via kexec |
| `powercycle` | Skip kexec (cold boot, useful for BMC) |
| `force` | Skip graceful teardown (emergency only) |

## Common Patterns

### Bootstrap cluster

```bash
talosctl gen secrets --output-file secrets.yaml
talosctl gen config prod https://10.0.1.100:6443 \
  --with-secrets secrets.yaml \
  --config-patch-control-plane @network.yaml
talosctl apply-config --nodes 10.0.1.10 --file controlplane.yaml --insecure
talosctl bootstrap --nodes 10.0.1.10
talosctl apply-config --nodes 10.0.1.11 --file controlplane.yaml
talosctl kubeconfig --nodes 10.0.1.10
kubectl get nodes
```

### Upgrade (rolling CP)

```bash
# Always stage/test first
talosctl upgrade --nodes <cp1> --image ghcr.io/siderolabs/installer:v1.13.1 \
  --reboot-mode default --stage
# On next reboot, apply staged upgrade
talosctl upgrade --nodes <cp1> --image ghcr.io/siderolabs/installer:v1.13.1

# Sequential CP upgrade with verification
for node in cp1 cp2 cp3; do
  talosctl upgrade --nodes $node --image ghcr.io/siderolabs/installer:v1.13.1
  kubectl wait --for=condition=Ready node/$node --timeout=10m
  talosctl etcd members
  sleep 30
done
```

### Upgrade Kubernetes

```bash
talosctl upgrade-k8s --to v1.36.1 --dry-run
talosctl upgrade-k8s --to v1.36.1
```

### Apply config change

```bash
# Patch specific fields
talosctl patch machineconfig --nodes cp1 --patch @patch.yaml
# Or full apply
talosctl apply-config --nodes cp1 --file controlplane.yaml
```

### etcd maintenance

```bash
talosctl etcd snapshot /backup/etcd-$(date +%Y%m%d).snapshot
talosctl etcd defrag --nodes cp1
talosctl etcd alarm list
talosctl etcd alarm disarm       # Clear no-space alarms
talosctl etcd forfeit-leadership # Move leader for maintenance
```

## Common Mistakes

- **Bootstrap more than once** — Only one `talosctl bootstrap` per cluster lifetime. Running it again creates split-brain.
- **Losing secrets** — Always `--with-secrets secrets.yaml` during `gen config`. Without secrets, you cannot add nodes, rotate certs, or recover. Store encrypted in Vault/SOPS.
- **Upgrade all CP simultaneously** — Sequential only. etcd needs quorum. One node at a time, verify health between each.
- **`talosctl get disks` vs `talosctl disks`** — `get disks` uses the `disks.block.talos.dev` resource. `talosctl disks` is a separate command with different output.
- **No `--preserve` flag** — Removed in v1.13+. Upgrade preserves data by default. Use `--reboot-mode` instead.
- **Forgot `--with-secrets` during gen** — Without it, each run produces different certs/keys. Nodes from one gen can't join configs from another.
- **Apply config to all nodes at once** — For CP changes, use sequential apply with health checks between nodes (etcd quorum).
- **`talosctl reset` without `--graceful=false`** — By default `--graceful=true` which cordons/drains and removes from etcd. Set `--graceful=false --reboot` for hard reset.

## v1.13+ Features

- **LifecycleService** — Unified upgrade API. Replaces legacy upgrade lifecycle.
- **Clang-built kernel** — ThinLTO, BTI (branch target identification) for arm64.
- **CDI enabled by default** — Container Device Interface for GPU/accelerator devices.
- **ImageVerificationConfig** — Verify container image signatures before pulling.
- **EnvironmentConfig document** — Replaces `.machine.env` in machine config. Separate config document.
- **ExternalVolumeConfig** — Virtiofs volumes for VMs.
- **Slice args** — `extraArgs` now accept YAML slices (`[val1, val2]`) in addition to string maps.
- **Flannel + kube-network-policies** — Built-in network policy support without Cilium.
- **KubeSpanConfig document** — WireGuard mesh config as separate document.
- **Advanced network docs** — RoutingRuleConfig, VRFConfig, ProbeConfig, ResolverConfig as separate documents.
