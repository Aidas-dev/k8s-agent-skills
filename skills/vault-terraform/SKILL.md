---
name: vault-terraform
description: Use when managing HashiCorp Vault resources as Terraform code. Covers provider config, auth backends, roles, policies, KV secrets, identity entities, ephemeral resources for write-once secrets.
---

# Vault (Terraform)

## Overview

The Vault Terraform provider manages Vault resources declaratively. 188+ resources, 38+ data sources, and 16 ephemeral resources for write-once secret patterns.

**Provider:** `hashicorp/vault` 5.9.0.

**Resources by category:**

| Category | Resources | Examples |
|----------|-----------|----------|
| Auth Backends | 10+ | vault_auth_backend, vault_kubernetes_auth_backend_config, vault_approle_auth_backend_role |
| Secrets Engines | 20+ | vault_mount, vault_kv_secret_v2, vault_transit_secret_backend_key |
| Policies | 4 | vault_policy, vault_egp_policy (enterprise) |
| Identity | 8+ | vault_identity_entity, vault_identity_entity_alias, vault_identity_group, vault_identity_group_member_entity_ids |
| PKI | 20+ | vault_pki_secret_backend_root_cert, vault_pki_secret_backend_role |
| Database | 15+ | vault_database_secret_backend_connection, vault_database_secret_backend_role |
| Token | 5 | vault_token, vault_token_auth_backend_role |
| Raft | 2 | vault_raft_snapshot_agent_config |

## Provider Configuration

```hcl
provider "vault" {
  address         = "http://vault.vault:8200"
  token           = var.vault_root_token   # or use Vault_TOKEN env var
  skip_child_token = true                  # avoid child token creation per module
}
```

Auth via environment (alternative):

```bash
export VAULT_ADDR=http://vault.vault:8200
export VAULT_TOKEN=hvs...
```

## Common Resource Patterns

### Kubernetes Auth Backend

```hcl
# Enable K8s auth
resource "vault_auth_backend" "kubernetes" {
  type = "kubernetes"
  path = "kubernetes"
}

# Configure K8s auth (in-cluster)
resource "vault_kubernetes_auth_backend_config" "config" {
  backend            = vault_auth_backend.kubernetes.path
  kubernetes_host    = "https://${var.kubernetes_service_host}:443"
  kubernetes_ca_cert = filebase64(var.k8s_ca_cert_path)
  issuer             = "https://kubernetes.default.svc.cluster.local"
}

# Create role for ESO
resource "vault_kubernetes_auth_backend_role" "eso" {
  backend                          = vault_auth_backend.kubernetes.path
  role_name                        = "eso-reader"
  bound_service_account_names      = ["external-secrets"]
  bound_service_account_namespaces = ["external-secrets"]
  token_policies                   = ["eso-reader"]
  token_ttl                        = 3600
}
```

### KV v2 Secrets Engine

```hcl
# Enable KV v2 at path "secret"
resource "vault_mount" "kv_v2" {
  path = "secret"
  type = "kv-v2"
  options = {
    version = "2"
  }
}

# Write a secret via ephemeral resource (no state)
ephemeral "vault_kv_secret_v2" "app_password" {
  mount = vault_mount.kv_v2.path
  path  = "${vault_mount.kv_v2.path}/data/myapp"
  data_json = jsonencode({
    password = var.app_password
  })
}

# Or persistent (state-tracked)
resource "vault_kv_secret_v2" "config" {
  mount = vault_mount.kv_v2.path
  path  = "${vault_mount.kv_v2.path}/data/myapp"
  data_json = jsonencode({
    api_url = "https://api.example.com"
  })
}
```

### Policies

```hcl
# Read-only policy for app
resource "vault_policy" "reader" {
  name = "eso-reader"
  policy = <<EOT
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}
path "secret/metadata/myapp/*" {
  capabilities = ["list"]
}
EOT
}
```

### Identity Entities

```hcl
# Create entity for a K8s service account
resource "vault_identity_entity" "eso" {
  name     = "external-secrets"
  policies = ["eso-reader"]
  metadata = {
    source = "kubernetes"
  }
}

# Alias K8s SA UID to entity
resource "vault_identity_entity_alias" "eso" {
  name           = "system:serviceaccount:external-secrets:external-secrets"
  mount_accessor = vault_auth_backend.kubernetes.accessor
  canonical_id   = vault_identity_entity.eso.id
}
```

## Ephemeral Resources (Write-Once Secrets)

Vault Terraform provider 5.x+ supports ephemeral resources — secrets written to Vault during apply but never stored in Terraform state:

```hcl
ephemeral "vault_kv_secret_v2" "bootstrap" {
  mount = vault_mount.kv_v2.path
  path  = "${vault_mount.kv_v2.path}/data/bootstrap"
  data_json = jsonencode({
    root_token  = var.vault_root_token
    unseal_keys = var.vault_unseal_keys
  })
}
```

Benefits:
- Secret values never enter `.tfstate`
- Written to Vault during apply, read from Vault during destroy
- Ideal for bootstrapping secrets that should only exist in Vault

## Data Sources

```hcl
# Read existing secret
data "vault_kv_secret_v2" "myapp" {
  mount = vault_mount.kv_v2.path
  path  = "${vault_mount.kv_v2.path}/data/myapp"
}

# List auth backends
data "vault_auth_backends" "all" {}

# Lookup token info
data "vault_token_lookup" "current" {}
```

## Common Mistakes

- **`skip_child_token = true` for module isolation.** Without this, each Terraform module creates a child token, which can exhaust Vault token TTL and create orphaned tokens. Set at provider level.
- **KV v2 path includes `/data/` segment.** `vault_kv_secret_v2` with `path = "secret/data/myapp"` — the provider expects the full path including `/data/`.
- **Ephemeral resources are not destroy-safe alone.** On `terraform destroy`, ephemeral resources are not destroyed — they skip the lifecycle. Use a `terraform apply` with updated config to overwrite, or delete via API.
- **Policy path capabilities mismatch.** `read` allows reading secret data. `list` allows listing metadata keys. `metadata/list` requires `list` on the metadata path, not on the data path.
- **K8s auth backend config requires correct issuer.** The `issuer` field must match the cluster's `serviceaccount.issuer` (usually `https://kubernetes.default.svc.cluster.local`). Wrong issuer = K8s auth login fails.
- **Provider version 5.x has breaking changes from 4.x.** The `vault_kv_secret` resource was replaced by `vault_kv_secret_v2` and `vault_generic_secret` was deprecated. Pin provider version in `required_providers`.
- **Identity alias name format.** For K8s auth, the alias name must be `system:serviceaccount:<namespace>:<name>`. Wrong format creates an alias that never resolves.
