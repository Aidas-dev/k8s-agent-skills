---
name: talos
description: Use when working with Talos Linux — route to the correct sub-skill based on what you need: talosctl CLI operations, machine configuration, or the Terraform provider.
---

# Talos — Skill Router

**Latest:** v1.13.x (Linux 6.18, etcd 3.6.9, K8s 1.36)  
**API version:** `v1alpha1` (multi-document config format)

## Overview

Talos is an immutable Kubernetes OS — no SSH, no shell, API-driven. All management via `talosctl`, machine config YAML, or the Terraform provider.

## Which Sub-Skill?

**Match the user's request against the trigger keywords below, then call `skill(name="<matched-skill>")` to load the sub-skill's full content.**

### Routing Table

| Trigger keywords | User wants to... | Skill to load |
|---|---|---|
| talosctl, upgrade, upgrade-k8s, bootstrap, kubeconfig, reboot, reset, rollback, etcd, service, logs, dmesg, dashboard, health, diagnostics, troubleshoot, CLI | Run talosctl commands against a cluster | `talosctl` |
| machine config, config, gen config, apply-config, patch, patching, secrets.yaml, network config, install disk, multi-document, v1alpha1, YAML config, LUKS, KubeSpan, disk encryption | Write or modify machine config YAML | `talosconfig` |
| terraform, provider, siderolabs/talos, talos_machine, machine_secrets, machine_configuration, cluster, IaC, infra as code, plan, apply | Manage Talos as infrastructure code | `talos-terraform` |
| tuppr, TalosUpgrade, KubernetesUpgrade, automated upgrade, orchestrated upgrade, GitOps upgrade, upgrade controller, maintenance window | Automate Talos/K8s upgrades via Kubernetes controller | `tuppr` |

### Decision Tree

If routing is ambiguous:

1. **User is running commands** (upgrade, bootstrap, diagnostics) → `talosctl`
2. **User is writing YAML** (config documents, patches) → `talosconfig`
3. **User is writing Terraform** (`.tf` files, `talos_*` resources) → `talos-terraform`
4. **User wants automated/GitOps upgrades** (CRs, controller, `TalosUpgrade`/`KubernetesUpgrade`) → `tuppr`
5. **Combination?** → Load the one matching the primary action; config often flows into CLI usage, so `talosconfig` → `talosctl` is a common pair

### When NOT to load this router

- User question is about a **specific OS-level concern** (kernel, hardware, GPU) → check if a hardware/GPU skill applies first
- User needs **cluster provisioning** on a specific cloud (Hetzner, Proxmox, AWS) → that's the cloud provider's concern; the `talos-terraform` skill covers the Talos-side resources
- User is working on **Tailscale on Talos nodes** → load `tailscale-talos` instead
- User just needs a **quick version/overview** → the Overview above is sufficient
