---
name: harbor-terraform
description: Use when managing Harbor infrastructure as code with Terraform — projects, robot accounts, registries, replication rules, retention policies, webhooks, users, groups, system config, and the goharbor/terraform-provider-harbor.
---

# Harbor Terraform Provider

Provider: `goharbor/harbor`. Latest: **v3.12.0** (Jun 4, 2026). Registry: `v3.11.6` (Apr 22, 2026). Source: `github.com/goharbor/terraform-provider-harbor`.

Tested against: Harbor 2.13–2.15, Terraform 1.12–1.14.

## Provider Config

```hcl
terraform {
  required_providers {
    harbor = {
      source  = "goharbor/harbor"
      version = "~> 3.11"
    }
  }
}

provider "harbor" {
  url       = "https://harbor.example.com"
  username  = "admin"
  password  = var.harbor_password
  insecure  = false    # Verify TLS (default: true — must explicitly set false)
}
```

### Auth Methods
| Method | Fields | Env Var |
|--------|--------|---------|
| Basic | `username` + `password` | `HARBOR_USERNAME`, `HARBOR_PASSWORD` |
| Bearer Token | `bearer_token` | — |
| OIDC Session | `session_id` | `HARBOR_SESSION_ID` |

`api_version` (default: 2) — use 1 for Harbor pre-2.0. `robot_prefix` — auto-detected via admin API unless explicitly set.

## Resources (20)

### Projects

```hcl
resource "harbor_project" "myapp" {
  name                   = "myapp"
  public                 = false
  vulnerability_scanning = true
  vulnerability_scanner  = "Trivy"             # v3.11.6+: per-project scanner override
  enable_content_trust   = true                # Notary
  enable_content_trust_cosign = false          # Cosign signatures
  auto_sbom_generation   = true                # Harbor 2.11+ SBOM on push
  storage_quota          = 100                 # GB
  deployment_security    = "high"              # Block images >= this severity
  cve_allowlist          = ["CVE-1234"]
  force_destroy          = false               # Allow destroy even with repos
}

# Proxy cache project
resource "harbor_registry" "dockerhub" {
  provider_name = "docker-hub"
  name          = "dockerhub-proxy"
  endpoint_url  = "https://hub.docker.com"
}

resource "harbor_project" "proxy" {
  name                           = "docker-proxy"
  registry_id                    = harbor_registry.dockerhub.registry_id
  proxy_speed_kb                 = -1
  proxy_cache_local_on_not_found = true   # Harbor 2.15.1+
}
```

### Robot Accounts

```hcl
# System-level
resource "harbor_robot_account" "system" {
  name        = "ci-system"
  description = "System-level CI robot"
  level       = "system"
  secret      = random_password.robot.result
  permissions {
    kind      = "project"
    namespace = "*"    # All projects
    access {
      resource = "repository"
      action   = "pull"
    }
  }
}

# Project-level
resource "harbor_robot_account" "deploy" {
  name        = "ci-deploy"
  description = "Deploy robot for myapp"
  level       = "project"
  permissions {
    kind      = "project"
    namespace = harbor_project.myapp.name
    access {
      resource = "repository"
      action   = "pull"
    }
    access {
      resource = "repository"
      action   = "push"
    }
  }
}
```

### Replication

```hcl
resource "harbor_replication" "backup" {
  name        = "replicate-to-dr"
  description = "Replicate myapp to DR site"
  registry_id = harbor_registry.dr.registry_id
  destination = "harbor"
  filters {
    name_filter = "myapp/**"
    resource    = "image"
  }
  trigger {
    type          = "event_based"
    override      = true
    target_events = ["imageUpload", "imageDelete"]
  }
  enabled = true
}
```

### Retention Policies

```hcl
resource "harbor_retention_policy" "main" {
  scope = harbor_project.myapp.id
  rule {
    tag_selectors {
      kind       = "doublestar"
      decoration = "matches"
      pattern    = "**"
      extras     = jsonencode({untagged: true})
    }
    scope_selectors {
      kind       = "project"
      decoration = "repoMatches"
      pattern    = "**"
    }
    action    = "retain"
    template  = "always"
    params    = jsonencode({num_latest_per_artifact: 10})
  }
}
```

### System Configuration

```hcl
resource "harbor_config_system" "cfg" {
  project_creation_restriction = "adminonly"
  robot_token_expiration       = 30
  robot_name_prefix            = "harbor@"
  storage_per_project          = 100
  notification_enable          = true
  banner_notification          = "Production Harbor - no test data"
}

resource "harbor_config_auth" "auth" {
  auth_mode          = "oidc_auth"
  oidc_name          = "dex"
  oidc_endpoint      = "https://dex.example.com"
  oidc_client_id     = "harbor"
  oidc_client_secret = var.oidc_secret
  oidc_scope         = "openid,profile,email,groups"
  oidc_verify_cert   = true
}

resource "harbor_config_security" "sec" {
  cve_allowlist = ["CVE-2024-1234", "CVE-2025-5678"]
  expires_at    = "1893456000"
}
```

### Other Resources

| Resource | Purpose |
|----------|---------|
| `harbor_garbage_collection` | GC schedule, workers, untagged deletion |
| `harbor_group` | User groups (LDAP/internal/OIDC) |
| `harbor_immutable_tag_rule` | Immutable tag rules per repo/project |
| `harbor_interrogation_services` | Default scanner config (Trivy/Clair) |
| `harbor_label` | Labels (global or project-scoped) |
| `harbor_preheat_instance` | P2P preheat instances |
| `harbor_project_member_group` | Group membership with role |
| `harbor_project_member_user` | User membership with role |
| `harbor_project_webhook` | Webhook policies |
| `harbor_purge_audit_log` | Audit log purge schedule |
| `harbor_tasks` | Scan policy schedule |
| `harbor_user` | Internal users |

## Data Sources (8)

| Data Source | Purpose |
|-------------|---------|
| `harbor_groups` | Look up groups by name/LDAP DN |
| `harbor_project` | Look up single project |
| `harbor_projects` | Look up multiple projects |
| `harbor_project_member_groups` | List member groups |
| `harbor_project_member_users` | List member users |
| `harbor_registry` | Look up registry by name |
| `harbor_robot_accounts` | List/filter robot accounts |
| `harbor_users` | Look up users |

## Importing Existing Resources

```hcl
terraform import harbor_project.main /projects/1
terraform import harbor_robot_account.system /robots/123
terraform import harbor_label.main /labels/1
terraform import harbor_user.main /users/42
```

## Common Mistakes

- **Robot secret not stored in state** — `secret` is sensitive. Use `random_password` resource and `secret = resource.random_password.robot.result`.
- **`force_destroy` required for non-empty projects** — Set to `true` to delete projects that still contain repos.
- **OIDC session_id is experimental** — Will be deprecated when Harbor provides a better auth method. Prefer `bearer_token` or basic auth.
- **Robot prefix auto-detection** — Requires admin API access. Without it, set `robot_prefix` explicitly in provider config.
- **`registry_id` vs `id`** — The `harbor_registry` resources expose `.registry_id` (int), not `.id`. Use `.registry_id` when referencing in projects.
- **v3.12.0 not on registry yet** — Latest on Terraform Registry is v3.11.6. Use `source = "goharbor/harbor"` + version constraint if depending on v3.12.0 features (per-project scanner, proxy cache local-on-not-found).
