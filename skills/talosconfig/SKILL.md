---
name: talosconfig
description: Talos Linux machine configuration — multi-document YAML format, config documents, generation flags, patching, apply modes, and v1.13+ config features.
---

# talosconfig — Machine Configuration

For CLI operations see `talosctl`. For Terraform provider see `talos-terraform`.

## Format

Multi-document YAML — each document is a config type. All config is declarative, applied via `talosctl apply-config` or `talosctl patch`. Legacy single-document v1alpha1 format still supported.

## Cluster Config Documents

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

## Key Config Fields

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

## Config Generation

| Flag | Purpose |
|------|---------|
| `--with-secrets secrets.yaml` | Use saved secrets (required for reproducibility) |
| `--config-patch @patch.yaml` | Patch all node types |
| `--config-patch-control-plane @cp.yaml` | Patch control planes only |
| `--config-patch-worker @w.yaml` | Patch workers only |
| `--additional-sans <dns>` | Extra SANs for API cert |
| `--dns-domain <domain>` | Cluster DNS domain (default: cluster.local) |
| `--kubernetes-version v1.36.0` | Override K8s version |

**Always use `--with-secrets`** — without it, each run produces different certs/keys and nodes can't join across runs. Store the secrets file encrypted (Vault/SOPS).

## Applying Config

**⚠️ Every change MUST be dry-run first, and the user MUST confirm before applying.**

**⚠️ `apply-config` replaces the ENTIRE machine config.** It needs the full config file, not a partial patch. Targeted changes use `talosctl patch machineconfig`.

```bash
# Targeted patch (dry-run first!)
talosctl patch machineconfig --nodes cp1 --patch @patch.yaml --dry-run
# after user confirms:
talosctl patch machineconfig --nodes cp1 --patch @patch.yaml

# Full config replacement (dry-run first!)
talosctl apply-config --nodes cp1 --file controlplane.yaml --dry-run
# after user confirms:
talosctl apply-config --nodes cp1 --file controlplane.yaml
```

### Apply Modes

| Mode | Behavior |
|------|----------|
| `auto` (default) | Immediate apply, reboot if needed |
| `no-reboot` | Apply without reboot |
| `reboot` | Always reboot to apply |
| `staged` | Apply on next reboot |
| `try` | Apply, rollback after timeout if not confirmed |

## v1.13+ Config Features

- **LifecycleService** — Unified upgrade API. Replaces legacy upgrade lifecycle.
- **EnvironmentConfig document** — Replaces `.machine.env` in machine config. Separate config document.
- **ImageVerificationConfig** — Verify container image signatures before pulling.
- **ExternalVolumeConfig** — Virtiofs volumes for VMs.
- **Slice args** — `extraArgs` now accept YAML slices (`[val1, val2]`) in addition to string maps.
- **KubeSpanConfig document** — WireGuard mesh config as separate document.
- **Advanced network docs** — RoutingRuleConfig, VRFConfig, ProbeConfig, ResolverConfig as separate documents.

## Common Mistakes

- **Losing secrets** — Always `--with-secrets secrets.yaml` during `gen config`. Without secrets, you cannot add nodes, rotate certs, or recover.
- **Forgot `--with-secrets` during gen** — Without it, each run produces different certs/keys. Nodes from one gen can't join configs from another.
- **`--stage`/`--preserve` deprecated** — Legacy upgrade flags, removed in Talos 1.18. Use `--no-reboot` instead of `--stage`.
- **Apply config to all nodes at once** — For CP changes, use sequential apply with health checks between nodes (etcd quorum).
- **Hardcoded network assumptions** — Config encodes node-specific IPs/hostnames. Use `--config-patch-*` per node type rather than editing the base config per node.
- **`.machine.env` in v1.13+** — Replaced by `EnvironmentConfig` document. Old field is deprecated.
