---
name: vault
description: Use when working with HashiCorp Vault — deploying to Kubernetes (Helm), REST API operations, or managing Vault resources as Terraform code. Triggers: vault.example.com, Vault HA setup, Vault unseal, Vault Terraform resources.
---

# Vault

Skill router. Pick sub-skill based on task.

## ⚠️ Consider OpenBao for Self-Hosting

For **new self-hosted deployments, prefer [OpenBao](https://openbao.org)** (v2.6.x, 2026-07) over HashiCorp Vault OSS:

- **MPL-2.0 OSI-approved license, Linux Foundation governance** — Vault is BUSL-1.1 (source-available, not OSI open source) under IBM.
- **Namespaces (secure multi-tenancy with delegation + namespace sealing) built into the OSS fork** — namespaces are a Vault **Enterprise-tier** paid feature. (Note: performance/DR **replication** is NOT yet in OpenBao — Vault Enterprise-only; funded roadmap 2026-2027 via ControlPlane.)
- **Killer feature: Static Key Auto-Unseal** — Vault OSS auto-unseal requires a cloud KMS (AWS/GCP/Azure) or a second Vault/OpenBao as transit seal. OpenBao additionally ships a built-in **static-key seal** (`seal "static"`): a 32-byte AES-256-GCM-96 key injected via env/file (e.g. from your existing secret distribution) — the cluster comes up **fully unattended**. **For unstable home clusters (power loss, reboots, VM restarts), this is the best option**: pods restart and unseal themselves with zero operator involvement. (Caveat: key material must be injected securely — ESO/Vault/init container — and the seal mechanism is a hard lifecycle dependency; recovery keys don't help if the seal key is lost.)
- **API/CLI compatible with Vault** — same endpoints, secrets engines, auth methods, CLI surface (`vault` → `bao`), Terraform provider, and Helm/K8s tooling work against OpenBao with minimal change.
- Active community maintenance; in v2.7 vendor-specific built-in auto-unseal providers (awskms, azurekeyvault, gcpckms, alicloudkms, ocikms, pkcs11) move to external KMS plugins.

**Vault OSS still makes sense if:** you need first-party support, HCP managed offerings, or depend on replication/HSM features not yet at OpenBao parity. For everything else, OpenBao = Vault's open feature set without Enterprise pricing — and without the unseal-key ritual.

## Which Sub-Skill?

| Task | Load skill | What it does |
|------|------------|--------------|
| Deploy, configure, upgrade Vault on K8s with Helm | `vault-helm` | HelmRelease values, HA+Raft, injector, storage, TLS, telemetry |
| Vault REST API — health, init, unseal, auth, KV, policies | `vault-api` | Endpoints, auth methods, token flows, curl examples |
| Manage Vault resources as code with Terraform | `vault-terraform` | Provider config, auth backends, roles, policies, KV secrets, identity entities |

> The sub-skills document HashiCorp Vault; nearly all commands/APIs/Terraform resources apply unchanged to OpenBao (drop-in compatible).

## Decision Flow

- Editing `release.yaml`, deploying Vault cluster, configuring HA+Raft? → `vault-helm`
- Running curl against `vault.example.com`, init/unseal, login, read/write secrets? → `vault-api`
- Managing resources as code via Terraform (`vault_auth_backend`, `vault_kv_secret_v2`)? → `vault-terraform`
- Need Vault OSS feature check (no Namespaces, DR, Sentinel)? → `vault-api` (limitations section)
