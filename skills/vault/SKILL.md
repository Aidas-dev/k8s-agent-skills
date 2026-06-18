---
name: vault
description: Use when working with HashiCorp Vault — deploying to Kubernetes (Helm), REST API operations, or managing Vault resources as Terraform code. Triggers: vault.kubexa.tech, Vault HA setup, Vault unseal, Vault Terraform resources.
---

# Vault

Skill router. Pick sub-skill based on task.

## Which Sub-Skill?

| Task | Load skill | What it does |
|------|------------|--------------|
| Deploy, configure, upgrade Vault on K8s with Helm | `vault-helm` | HelmRelease values, HA+Raft, injector, storage, TLS, telemetry |
| Vault REST API — health, init, unseal, auth, KV, policies | `vault-api` | Endpoints, auth methods, token flows, curl examples |
| Manage Vault resources as code with Terraform | `vault-terraform` | Provider config, auth backends, roles, policies, KV secrets, identity entities |

## Decision Flow

- Editing `release.yaml`, deploying Vault cluster, configuring HA+Raft? → `vault-helm`
- Running curl against `vault.kubexa.tech`, init/unseal, login, read/write secrets? → `vault-api`
- Managing resources as code via Terraform (`vault_auth_backend`, `vault_kv_secret_v2`)? → `vault-terraform`
- Need Vault OSS feature check (no Namespaces, DR, Sentinel)? → `vault-api` (limitations section)
