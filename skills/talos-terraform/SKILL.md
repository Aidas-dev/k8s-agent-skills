---
name: talos-terraform
description: Talos Linux Terraform provider (siderolabs/talos) — machine secrets, machine configuration, config apply, bootstrap, and the sidero-blessed production pattern (live secrets + patch-based config, upgrades owned elsewhere).
---

# talos-terraform — Terraform Provider

For CLI operations see `talosctl`. For machine config YAML see `talosconfig`.

**Provider:** `siderolabs/talos`
**Latest stable:** v0.11.0 (2026-04-27) — SDK targets Talos v1.13 config line
**Next (alpha):** v0.12.0-alpha.x (2026-06-25) — adds `talos_machine`/`talos_cluster`, based on Talos v1.14. NOT stable.

## Provider Setup

```hcl
terraform {
  required_providers {
    talos = {
      source  = "siderolabs/talos"
      version = "~> 0.11.0"   # stable
    }
  }
}
```

## Sidero-Blessed Production Pattern

Production-proven pattern for managing an EXISTING Talos cluster with Terraform. Core idea: **generate config from live cluster secrets + patch files, apply with the stable `talos_machine_configuration_apply` resource, and let a separate system own OS/K8s upgrades.**

### Why NOT talos_machine (alpha)

`talos_machine` exists only in v0.12.0-alpha. It has a **Create/Update bug with `image = null`** — Optional+Computed fields plan as Unknown, and the provider attempts an upgrade with an empty image ref. If OS upgrades are owned elsewhere (e.g. tuppr `TalosUpgrade` CRs), avoid `talos_machine` entirely.

### Architecture

```
locals (node map + live secrets + talosconfig)
  → data talos_machine_configuration (per node: live secrets + patch file)
    → resource talos_machine_configuration_apply (apply config to node)
      → apply_mode = staged_if_needing_reboot
      → lifecycle prevent_destroy
```

### The Pattern

```hcl
# 1. Cluster secrets from the LIVE cluster — NEVER regenerate.
#    Regenerating rotates all certs. Extracted once, gitignored.
locals {
  cluster_secrets = jsondecode(file("${path.module}/local_cluster_secrets.json"))
}

# 2. Client config from the cluster's talosconfig (os:admin).
#    certs.os = machine CA (for config gen) — NOT a valid apid client.
#    apid RBAC requires subject O=os:admin.
locals {
  talos_cfg = yamldecode(file("${path.module}/talosconfig.yaml"))
  talos_client_config = {
    ca_certificate     = local.talos_cfg.contexts[local.talos_cfg.context].ca
    client_certificate = local.talos_cfg.contexts[local.talos_cfg.context].crt
    client_key         = local.talos_cfg.contexts[local.talos_cfg.context].key
  }
}

# 3. Generate machine config per node: live secrets + version contract +
#    the applied config as patch (byte-exact reproduction of node state).
data "talos_machine_configuration" "node" {
  for_each = local.nodes

  cluster_name       = var.cluster_name
  cluster_endpoint   = each.value.endpoint
  machine_type       = each.value.role          # controlplane | worker
  machine_secrets    = local.cluster_secrets
  kubernetes_version = each.value.kubernetes_version
  talos_version      = each.value.talos_version  # RECOMMENDED: pin it

  config_patches = [file("${path.module}/patches/${each.key}-apply.yaml")]
}

# 4. Apply config — config-only, no OS version sync.
resource "talos_machine_configuration_apply" "node" {
  for_each = local.nodes

  node                        = each.value.ip
  client_configuration        = local.talos_client_config
  machine_configuration_input = data.talos_machine_configuration.node[each.key].machine_configuration
  apply_mode                  = "staged_if_needing_reboot"

  lifecycle {
    prevent_destroy = true   # live nodes: reconcile config only, never recreate
  }
}
```

### Key Decisions (from production)

| Decision | Why |
|----------|-----|
| Secrets from **live cluster** (extracted, gitignored) | Regenerating = cert rotation = cluster outage |
| `os:admin` client config from talosconfig | `certs.os` (machine CA) is NOT accepted by apid RBAC |
| `talos_machine_configuration_apply` (stable) | `talos_machine` is alpha + broken with `image = null` |
| `apply_mode = staged_if_needing_reboot` | Config-only change applies immediately; reboot-requiring change stages for next reboot |
| **No `image` set** | OS version NOT managed by TF — a dedicated upgrade system owns upgrades (see `tuppr` skill: TalosUpgrade/KubernetesUpgrade CRs) |
| `config_patches` = the applied config as patch | Byte-exact reproduction of what's on the node |
| `prevent_destroy = true` | Live nodes never destroyed/recreated by TF |

## Core Resources (v0.11.0 stable)

| Resource | Purpose |
|----------|---------|
| `talos_machine_secrets` | Generate cluster secrets (bootstrap only — never on existing cluster!) |
| `talos_machine_configuration` (data source) | Generate machine config for a node type |
| `talos_machine_configuration_apply` | Apply config to a node (stable, production path) |
| `talos_machine_bootstrap` | Bootstrap etcd (first CP only) |
| `talos_cluster_kubeconfig` | Fetch kubeconfig |
| `talos_image_factory_schematic` | Image factory schematic (secureboot) |

**Data sources:** `talos_client_configuration`, `talos_cluster_health`, `talos_machine_disks`, Image Factory (versions/extensions/overlays/urls).

**Ephemeral variants (new in 0.11):** `talos_machine_configuration`, `talos_machine_secrets`, `talos_client_configuration`, `talos_cluster_kubeconfig` — keep secrets OUT of state. Requires Terraform 1.10+. Use `_wo` (write-only) attributes (Terraform 1.11+) with them.

## Bootstrap Workflow (new cluster)

```hcl
# 1. Generate secrets ONCE (fresh cluster only)
resource "talos_machine_secrets" "this" {}

# 2. Generate config per node type
data "talos_machine_configuration" "controlplane" {
  cluster_name     = "example-cluster"
  cluster_endpoint = "https://10.5.0.2:6443"
  machine_type     = "controlplane"
  machine_secrets  = talos_machine_secrets.this.machine_secrets
  talos_version    = "v1.13.8"          # pin it — new provider SDK enables new config features
  config_patches = [file("patches/controlplane.yaml")]
}

# 3. Apply config to first CP (insecure maintenance mode)
resource "talos_machine_configuration_apply" "first_cp" {
  node                        = "10.5.0.2"
  client_configuration        = talos_machine_secrets.this.client_configuration
  machine_configuration_input = data.talos_machine_configuration.controlplane.machine_configuration
}

# 4. Bootstrap etcd
resource "talos_machine_bootstrap" "this" {
  depends_on           = [talos_machine_configuration_apply.first_cp]
  node                 = "10.5.0.2"
  client_configuration = talos_machine_secrets.this.client_configuration
}
```

### Why pin `talos_version`?

Without it, a new provider version with a new Talos SDK major enables new machineconfig features by default → unexpected behavior. Example values: `v1.12`, `v1.13.8`, `1.13`.

## Common Mistakes

- **Regenerating secrets on an existing cluster** — `talos_machine_secrets` is bootstrap-only. On a live cluster, extract secrets from the cluster and store gitignored. Regenerating rotates all certs → outage.
- **Using `certs.os` as apid client** — The machine CA cert is for config generation, not apid auth. apid RBAC requires `O=os:admin`. Use the cluster's talosconfig client cert.
- **`talos_machine` with `image = null`** — Alpha resource, Create/Update bug: Optional+Computed plans Unknown, provider tries empty-image upgrade. Use `talos_machine_configuration_apply` and let a dedicated system own upgrades.
- **No `talos_version` pin** — New provider + new SDK major silently enables new config features. Always set it.
- **No `prevent_destroy`** — TF can destroy/recreate live nodes. Always set `lifecycle { prevent_destroy = true }` for production nodes.
- **apply_mode default** — `auto` reboots immediately when needed. `staged_if_needing_reboot` avoids surprise reboots on config-only changes.
- **`kubernetes_version` mismatch** — In `talos_machine_configuration`, `kubernetes_version` only sets component images at bootstrap; it does NOT trigger upgrades. Sync it with whatever owns upgrades.
- **Client-side vs write-only confusion** — With ephemeral resources, use `_wo` attributes everywhere to keep secrets out of state (requires Terraform 1.11+).
