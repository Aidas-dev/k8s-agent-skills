---
name: zitadel-terraform
description: Manage ZITADEL identity platform resources as code using the Terraform provider. Covers organizations, projects, applications (OIDC/API/SAML), users (human/machine), authentication providers, policies, system features, and customizations.
---

# ZITADEL Terraform Provider

**Provider:** `zitadel/zitadel`
**Latest:** v3.0.0
**Source:** `github.com/zitadel/terraform-provider-zitadel`
**Registry:** [terraform.io/providers/zitadel/zitadel](https://registry.terraform.io/providers/zitadel/zitadel/latest)

## Provider Config

### Auth Methods

| Method | Config Field | Description |
|--------|-------------|-------------|
| PAT | `access_token` | Personal Access Token - simplest for automation |
| JWT Profile File | `jwt_profile_file` | Path to JSON key file (service account) |
| JWT Profile JSON | `jwt_profile_json` | Inline JSON key credentials |
| JWT File | `jwt_file` | Path to pre-signed JWT |
| System API | `system_api` | PEM-encoded key for system-level operations |

```hcl
terraform {
  required_providers {
    zitadel = {
      source  = "zitadel/zitadel"
      version = "~> 3.0"
    }
  }
}

# Personal Access Token
provider "zitadel" {
  domain       = "auth.kubexa.tech"
  access_token = var.zitadel_pat
}

# JWT Profile (service account)
provider "zitadel" {
  domain           = "auth.kubexa.tech"
  jwt_profile_file = "zitadel-service-account.json"
}

# System API (instance-level management)
provider "zitadel" {
  domain = "auth.kubexa.tech"
  system_api {
    user   = var.system_user_id
    key    = var.system_private_key_pem
  }
}
```

### Provider Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `domain` | ✅ | ZITADEL instance domain |
| `access_token` | ⚠️ | PAT (one of access_token/jwt_file/jwt_profile_file/jwt_profile_json/system_api required) |
| `jwt_profile_file` | ⚠️ | Path to service account JSON key |
| `jwt_profile_json` | ⚠️ | Inline JSON key |
| `jwt_file` | ⚠️ | Pre-signed JWT file path |
| `system_api` | ⚠️ | Block for system API auth |
| `port` | ❌ | Non-default port (optional) |
| `insecure` | ❌ | Use HTTP (default: false) |
| `insecure_skip_verify_tls` | ❌ | Skip TLS verify (dev only) |
| `transport_headers` | ❌ | Custom headers for proxy auth |

## Resources

### Organizations

| Resource | Description | ZITADEL Version |
|----------|-------------|-----------------|
| `zitadel_organization` | Org using org/v2 API | 4.x+ |
| `zitadel_org` | Backward-compatible org (v3/v4 auto-fallback) | 3.x+ |
| `zitadel_org_member` | User membership with role on org | 3.x+ |
| `zitadel_organization_domain` | Domain on org (org/v2 API) | 4.x+ |
| `zitadel_domain` | Domain (deprecated, use org domain) | 3.x |
| `zitadel_organization_metadata` | Key-value metadata on org (v2 API) | 4.x+ |
| `zitadel_org_metadata` | Org metadata (deprecated) | 3.x |

```hcl
# Organization (ZITADEL 4.x)
resource "zitadel_organization" "myapp" {
  name = "myapp-org"
}

# Organization with custom ID and admin
resource "zitadel_organization" "platform" {
  name   = "platform-org"
  org_id = "platform-org-001"
  admins = [
    {
      user_id = data.zitadel_human_user.admin.id
      roles   = ["ORG_OWNER"]
    }
  ]
}

# Organization (backward-compatible 3.x/4.x)
resource "zitadel_org" "legacy" {
  name = "legacy-org"
}

# Org member
resource "zitadel_org_member" "dev" {
  org_id = zitadel_organization.myapp.id
  user_id = data.zitadel_human_user.dev.id
  roles   = ["ORG_USER_MANAGER"]
}

# Org domain (4.x+)
resource "zitadel_organization_domain" "main" {
  org_id = zitadel_organization.myapp.id
  domain = "myapp.kubexa.tech"
}
```

### Projects & Grants

| Resource | Description |
|----------|-------------|
| `zitadel_project` | Project with role assertion, private labeling |
| `zitadel_project_role` | Role definition in a project |
| `zitadel_project_member` | User membership on a project |
| `zitadel_project_grant` | Grant project to another org with roles |
| `zitadel_project_grant_member` | User membership on a granted project |
| `zitadel_user_grant` | Direct authorization of user with project roles |

```hcl
# Project
resource "zitadel_project" "myapp" {
  org_id                   = zitadel_organization.myapp.id
  name                     = "myapp"
  project_role_assertion   = true
  project_role_check       = true
  has_project_check        = false
  private_labeling_setting = "PRIVATE_LABELING_SETTING_ENFORCE_PROJECT_RESOURCE_OWNER_POLICY"
}

# Project roles
resource "zitadel_project_role" "admin" {
  org_id      = zitadel_organization.myapp.id
  project_id  = zitadel_project.myapp.id
  role_name   = "admin"
  display_name = "Administrator"
  group       = "admin-group"
}

resource "zitadel_project_role" "viewer" {
  org_id      = zitadel_organization.myapp.id
  project_id  = zitadel_project.myapp.id
  role_name   = "viewer"
  display_name = "Read-only Viewer"
}

# Project member
resource "zitadel_project_member" "lead" {
  org_id     = zitadel_organization.myapp.id
  project_id = zitadel_project.myapp.id
  user_id    = data.zitadel_human_user.lead.id
  roles      = ["admin"]
}

# Grant project to another org
resource "zitadel_project_grant" "partner" {
  org_id           = zitadel_organization.myapp.id
  granted_org_id   = data.zitadel_org.partner.id
  project_id       = zitadel_project.myapp.id
  role_keys        = ["viewer"]
}

# Direct user grant
resource "zitadel_user_grant" "external" {
  org_id     = zitadel_organization.myapp.id
  project_id = zitadel_project.myapp.id
  user_id    = data.zitadel_human_user.external.id
  role_keys  = ["viewer"]
}
```

### Applications

| Resource | Description |
|----------|-------------|
| `zitadel_application_oidc` | OIDC app (Web, User Agent, Native) |
| `zitadel_application_api` | API app (Basic or Private Key JWT auth) |
| `zitadel_application_saml` | SAML app (XML metadata or URL) |
| `zitadel_application_key` | App key for API/SAML apps |

```hcl
# OIDC Web App (authorization code flow)
resource "zitadel_application_oidc" "web" {
  org_id       = zitadel_organization.myapp.id
  project_id   = zitadel_project.myapp.id
  name         = "myapp-web"

  redirect_uris             = ["https://app.myapp.com/callback"]
  post_logout_redirect_uris = ["https://app.myapp.com"]
  response_types            = ["OIDC_RESPONSE_TYPE_CODE"]
  grant_types               = ["OIDC_GRANT_TYPE_AUTHORIZATION_CODE"]
  app_type                  = "OIDC_APP_TYPE_WEB"
  auth_method_type          = "OIDC_AUTH_METHOD_TYPE_BASIC"
  version                   = "OIDC_VERSION_1_0"
  dev_mode                  = false
  access_token_type         = "OIDC_TOKEN_TYPE_JWT"
  access_token_role_assertion  = true
  id_token_role_assertion      = true
  id_token_userinfo_assertion  = true
  clock_skew                = "0s"
}

# OIDC SPA (user agent / PKCE)
resource "zitadel_application_oidc" "spa" {
  org_id       = zitadel_organization.myapp.id
  project_id   = zitadel_project.myapp.id
  name         = "myapp-spa"

  redirect_uris = ["https://app.myapp.com/callback"]
  response_types = ["OIDC_RESPONSE_TYPE_CODE"]
  grant_types   = ["OIDC_GRANT_TYPE_AUTHORIZATION_CODE"]
  app_type      = "OIDC_APP_TYPE_USER_AGENT"
  auth_method_type = "OIDC_AUTH_METHOD_TYPE_NONE"
  version       = "OIDC_VERSION_1_0"
}

# API App (machine-to-machine)
resource "zitadel_application_api" "api" {
  org_id          = zitadel_organization.myapp.id
  project_id      = zitadel_project.myapp.id
  name            = "myapp-api"
  auth_method_type = "API_AUTH_METHOD_TYPE_PRIVATE_KEY_JWT"
}

# SAML App
resource "zitadel_application_saml" "saml" {
  org_id       = zitadel_organization.myapp.id
  project_id   = zitadel_project.myapp.id
  name         = "myapp-saml"
  metadata_url = "https://sp.example.com/metadata"
}
```

### Users

| Resource | Description | ZITADEL Version |
|----------|-------------|-----------------|
| `zitadel_human_user` | Human user (user/v2 API) | 4.x+ |
| `zitadel_machine_user` | Machine user/service account | 3.x+ |
| `zitadel_machine_key` | Key for machine user | 3.x+ |
| `zitadel_personal_access_token` | PAT for any user | 3.x+ |
| `zitadel_user_metadata` | Key-value metadata on user | 3.x+ |

```hcl
# Human user
resource "zitadel_human_user" "dev" {
  org_id         = zitadel_organization.myapp.id
  user_name      = "dev@myapp.com"
  first_name     = "Alice"
  last_name      = "Dev"
  display_name   = "Alice Dev"
  preferred_language = "en"
  email          = "alice@example.com"
  is_email_verified = true
  initial_password = var.user_password
  initial_skip_password_change = false
}

# Machine user (service account)
resource "zitadel_machine_user" "ci" {
  org_id            = zitadel_organization.myapp.id
  user_name         = "ci-bot@myapp"
  name              = "CI/CD Service Account"
  description       = "Used by GitHub Actions for deployments"
  with_secret       = true
  access_token_type = "ACCESS_TOKEN_TYPE_JWT"
}

# PAT for machine user
resource "zitadel_personal_access_token" "ci_token" {
  org_id          = zitadel_organization.myapp.id
  user_id         = zitadel_machine_user.ci.id
  expiration_date = "2027-01-01T00:00:00Z"
}

# Machine key
resource "zitadel_machine_key" "ci_key" {
  org_id    = zitadel_organization.myapp.id
  user_id   = zitadel_machine_user.ci.id
  type      = "KEY_TYPE_JSON"
  date = "2027-01-01T00:00:00Z"
}

# User metadata
resource "zitadel_user_metadata" "role" {
  org_id  = zitadel_organization.myapp.id
  user_id = zitadel_human_user.dev.id
  key     = "department"
  value   = "engineering"
}
```

### Instance-Level IDP (Identity Providers)

Each IDP type has a corresponding resource on both instance and org level.

| Resource | Provider Type |
|----------|--------------|
| `zitadel_idp_apple` | Apple |
| `zitadel_idp_azure_ad` | Azure AD / Microsoft |
| `zitadel_idp_github` | GitHub |
| `zitadel_idp_github_es` | GitHub Enterprise Server |
| `zitadel_idp_gitlab` | GitLab |
| `zitadel_idp_gitlab_self_hosted` | GitLab Self-Hosted |
| `zitadel_idp_google` | Google |
| `zitadel_idp_ldap` | LDAP |
| `zitadel_idp_oauth` | Generic OAuth2 |
| `zitadel_idp_oidc` | Generic OIDC |
| `zitadel_idp_saml` | SAML |

```hcl
# Generic OIDC IDP (instance level)
resource "zitadel_idp_oidc" "dex" {
  name                  = "Dex"
  client_id             = var.dex_client_id
  client_secret         = var.dex_client_secret
  issuer                = "https://dex.example.com"
  scopes                = ["openid", "profile", "email", "groups"]
  is_linking_allowed    = true
  is_creation_allowed   = true
  is_auto_creation      = false
  is_auto_update        = true
}

# LDAP IDP
resource "zitadel_idp_ldap" "corp" {
  name            = "Corporate AD"
  host            = "ldap.corp.example.com"
  port            = 636
  base_dn         = "DC=corp,DC=example,DC=com"
  bind_dn         = "CN=zitadel,CN=Users,DC=corp,DC=example,DC=com"
  bind_password   = var.ldap_password
  user_base       = "CN=Users,DC=corp,DC=example,DC=com"
  user_filters    = ["(objectClass=person)"]
  id_attribute    = "objectGUID"
  first_name_attribute = "givenName"
  last_name_attribute  = "sn"
  display_name_attribute = "displayName"
  email_attribute      = "mail"
  phone_attribute      = "telephoneNumber"
  avatar_url_attribute = "thumbnailPhoto"
  is_linking_allowed   = true
  is_creation_allowed  = true
  is_auto_creation     = false
  is_auto_update       = true
}
```

### Org-Level IDP

Same providers, prefixed with `org_`: `zitadel_org_idp_oidc`, `zitadel_org_idp_ldap`, etc.

```hcl
# Org-level OIDC IDP
resource "zitadel_org_idp_oidc" "partner_sso" {
  org_id          = zitadel_organization.myapp.id
  instance_id     = zitadel_idp_oidc.dex.id
  is_linking_allowed  = true
  is_creation_allowed = true
  is_auto_creation    = false
}
```

### Default Policies (Instance-Level)

| Resource | Purpose |
|----------|---------|
| `zitadel_default_domain_policy` | Domain customization (user domain, org domain modes) |
| `zitadel_default_label_policy` | Branding: logo, colors, font, hide login suffix |
| `zitadel_default_lockout_policy` | Lockout after failed attempts |
| `zitadel_default_login_policy` | Login flow: MFA, passwordless, IDP, lifetime |
| `zitadel_default_notification_policy` | Password change notification (verify vs send) |
| `zitadel_default_oidc_settings` | OIDC settings: lifetimes, token clock skew |
| `zitadel_default_password_age_policy` | Password expiry, reuse prevention |
| `zitadel_default_password_complexity_policy` | Password strength: length, chars, symbols |
| `zitadel_default_privacy_policy` | Privacy URL, TOS, help link |
| `zitadel_default_security_settings` | Security: embedding, IFrame origins |

```hcl
# Login policy - MFA + passwordless
resource "zitadel_default_login_policy" "cfg" {
  allow_username_password  = true
  allow_register           = false
  allow_external_idp       = true
  force_mfa                = true
  passwordless_type        = "PASSWORDLESS_TYPE_ALLOWED"
  multi_factor             = "MULTI_FACTOR_TYPE_TOTP"
  second_factor            = "SECOND_FACTOR_TYPE_OTP"
  user_login_mfa           = "USER_LOGIN_MFA_REQUIRED"

  # Session
  lifetime_seconds = 86400
  mfa_init_skip_lifetime_seconds = 7776000
}

# Password complexity
resource "zitadel_default_password_complexity_policy" "cfg" {
  min_length = 16
  has_lowercase = true
  has_uppercase = true
  has_number    = true
  has_symbol    = true
}

# Password age (90 day expiry, 10 history)
resource "zitadel_default_password_age_policy" "cfg" {
  max_age_days     = 90
  expire_warn_days = 14
  not_recently_count = 10
}

# Lockout (5 attempts)
resource "zitadel_default_lockout_policy" "cfg" {
  max_password_attempts = 5
}

# Branding
resource "zitadel_default_label_policy" "cfg" {
  primary_color        = "#5469D4"
  secondary_color      = "#1A1F36"
  warn_color           = "#E53E3E"
  background_color     = "#FFFFFF"
  font_color           = "#1A1F36"
  primary_color_dark   = "#B2BFF5"
  secondary_color_dark = "#D1D5DB"
  warn_color_dark      = "#FC8181"
  background_color_dark = "#1A202C"
  font_color_dark      = "#EDF2F7"
  disable_watermark    = false
  logo_url             = "https://static.kubexa.tech/logo.svg"
  logo_url_dark        = "https://static.kubexa.tech/logo-dark.svg"
}
```

### Org-Level Policies

Override defaults per-org: `zitadel_lockout_policy`, `zitadel_login_policy`, `zitadel_password_complexity_policy`, `zitadel_password_age_policy`, `zitadel_label_policy`, `zitadel_domain_policy`, `zitadel_privacy_policy`, `zitadel_notification_policy`.

```hcl
# Org-specific login policy overrides
resource "zitadel_login_policy" "myapp" {
  org_id = zitadel_organization.myapp.id
  allow_username_password = true
  allow_external_idp      = true
  force_mfa               = false
}
```

### System & Instance

| Resource | Description | Auth Required |
|----------|-------------|---------------|
| `zitadel_system_features` | System-wide feature flags | System API |
| `zitadel_instance_features` | Per-instance features | System API |
| `zitadel_instance_member` | User membership on instance | System API |
| `zitadel_instance_custom_domain` | Custom domain for instance | System API |
| `zitadel_instance_trusted_domain` | Trusted domain | System API |
| `zitadel_instance_restrictions` | Instance restrictions | System API |
| `zitadel_instance_secret_generator` | Secret generator config | System API |
| `zitadel_webkey` | Web key | System API |
| `zitadel_active_webkey` | Active web key | System API |

```hcl
# System features (requires System API auth)
resource "zitadel_system_features" "cfg" {
  login_default_org   = true
  oidc_token_exchange = true
  user_schema         = false
  improved_performance = [
    "IMPROVED_PERFORMANCE_PROJECT_GRANT",
    "IMPROVED_PERFORMANCE_ORG_DOMAIN_VERIFIED"
  ]
  login_v2 = {
    required = true
    base_uri = "https://login.kubexa.tech"
  }
}
```

### Actions (Serverless ZITADEL Functions)

| Resource | Description |
|----------|-------------|
| `zitadel_action` | Custom action (Go code, triggered by events) |
| `zitadel_action_target` | Target for action execution (webhook URL) |
| `zitadel_action_target_public_key` | Public key for payload encryption |
| `zitadel_action_execution_event` | Trigger action on event |
| `zitadel_action_execution_function` | Trigger action on function call |
| `zitadel_action_execution_request` | Trigger action on request |
| `zitadel_action_execution_response` | Trigger action on response |
| `zitadel_trigger_actions` | Map triggers to actions (legacy) |

### Message Templates (Instance Defaults)

| Resource | For |
|----------|-----|
| `zitadel_default_init_message_text` | Account initialization email |
| `zitadel_default_password_reset_message_text` | Password reset notification |
| `zitadel_default_password_change_message_text` | Password changed notification |
| `zitadel_default_verify_email_message_text` | Email verification |
| `zitadel_default_verify_email_otp_message_text` | OTP verification email |
| `zitadel_default_verify_phone_message_text` | Phone verification SMS |
| `zitadel_default_verify_sms_otp_message_text` | OTP SMS verification |
| `zitadel_default_domain_claimed_message_text` | Domain claimed notification |
| `zitadel_default_passwordless_registration_message_text` | Passkey registration |
| `zitadel_default_invite_user_message_text` | User invitation |

### Message Templates (Org Overrides)

Same list without `default_` prefix: `zitadel_init_message_text`, `zitadel_verify_email_message_text`, etc.

### Login Texts

| Resource | Description |
|----------|-------------|
| `zitadel_default_login_texts` | Instance-level login UI (v1) text customization |
| `zitadel_login_texts` | Org-level login UI (v1) text override |

### Providers (Email & SMS)

| Resource | Description |
|----------|-------------|
| `zitadel_email_provider_smtp` | SMTP email provider |
| `zitadel_email_provider_http` | HTTP email provider (API-based) |
| `zitadel_smtp_config` | SMTP config (deprecated - use email_provider_smtp) |
| `zitadel_sms_provider_twilio` | Twilio SMS provider |
| `zitadel_sms_provider_http` | HTTP SMS provider |

```hcl
# SMTP Email Provider
resource "zitadel_email_provider_smtp" "main" {
  from_address   = "noreply@kubexa.tech"
  from_name      = "ZITADEL Auth"
  smtp_host      = "smtp.kubexa.tech"
  smtp_port      = 587
  smtp_user      = "zitadel@smtp.kubexa.tech"
  smtp_password  = var.smtp_password
  tls_type       = "TLS"
  start_tls      = true
}
```

## Data Sources

| Data Source | Purpose |
|-------------|---------|
| `data.zitadel_org` | Look up org by id/name/domain |
| `data.zitadel_orgs` | List orgs |
| `data.zitadel_organization` | Look up org (v2 API) |
| `data.zitadel_organizations` | List organizations |
| `data.zitadel_organization_domain` | Single domain |
| `data.zitadel_organization_domains` | List domains |
| `data.zitadel_organization_metadata` | Single metadata entry |
| `data.zitadel_organization_metadatas` | List metadata |
| `data.zitadel_project` | Look up project |
| `data.zitadel_projects` | List projects |
| `data.zitadel_project_role` | Single role |
| `data.zitadel_project_roles` | All roles in project |
| `data.zitadel_human_user` | Look up human user |
| `data.zitadel_human_users` | List human users |
| `data.zitadel_machine_user` | Look up machine user |
| `data.zitadel_machine_users` | List machine users |
| `data.zitadel_user_metadata` | Single metadata entry |
| `data.zitadel_user_metadatas` | List metadata |
| `data.zitadel_application_oidc` | Single OIDC app |
| `data.zitadel_application_oidcs` | List OIDC apps |
| `data.zitadel_application_api` | Single API app |
| `data.zitadel_application_apis` | List API apps |
| `data.zitadel_application_saml` | Single SAML app |
| `data.zitadel_application_samls` | List SAML apps |
| `data.zitadel_action` | Action |
| `data.zitadel_action_target` | Action target |
| `data.zitadel_action_target_public_key` | Target public key |
| `data.zitadel_action_execution_event` | Event execution |
| `data.zitadel_action_execution_function` | Function execution |
| `data.zitadel_action_execution_request` | Request execution |
| `data.zitadel_action_execution_response` | Response execution |
| `data.zitadel_trigger_actions` | Trigger actions |
| `data.zitadel_instance` | Instance info |
| `data.zitadel_instance_custom_domains` | Custom domains |
| `data.zitadel_instance_features` | Instance features |
| `data.zitadel_instance_restrictions` | Instance restrictions |
| `data.zitadel_instance_secret_generator` | Secret generator |
| `data.zitadel_instance_trusted_domains` | Trusted domains |
| `data.zitadel_system_features` | System features |
| `data.zitadel_idp_*` | Lookup any IDP type |
| `data.zitadel_org_idp_*` | Lookup org IDP |
| `data.zitadel_default_oidc_settings` | Default OIDC settings |
| `data.zitadel_webkey` | Web key |
| `data.zitadel_zitadel` | Provider session token |

```hcl
# Data source examples
data "zitadel_org" "default" {
  name = "myapp-org"
}

data "zitadel_project" "default" {
  org_id     = data.zitadel_org.default.id
  project_id = zitadel_project.myapp.id
}

data "zitadel_human_user" "admin" {
  org_id      = data.zitadel_org.default.id
  user_name   = "admin@myapp.com"
}

data "zitadel_machine_users" "all" {
  org_id = data.zitadel_org.default.id
}

data "zitadel_application_oidcs" "web_apps" {
  org_id     = data.zitadel_org.default.id
  project_id = zitadel_project.myapp.id
}

data "zitadel_project_roles" "all" {
  org_id     = data.zitadel_org.default.id
  project_id = zitadel_project.myapp.id
}
```

## Complete Example

```hcl
# Full bootstrap: org -> project -> OIDC app -> machine user -> PAT
terraform {
  required_providers {
    zitadel = {
      source  = "zitadel/zitadel"
      version = "~> 3.0"
    }
  }
}

provider "zitadel" {
  domain           = "auth.kubexa.tech"
  jwt_profile_file = "zitadel-admin.json"
}

# Org
resource "zitadel_organization" "platform" {
  name = "platform"
}

# Project
resource "zitadel_project" "idp" {
  org_id                 = zitadel_organization.platform.id
  name                   = "Identity Platform"
  project_role_assertion = true
  project_role_check     = true
}

# Role
resource "zitadel_project_role" "admin" {
  org_id       = zitadel_organization.platform.id
  project_id   = zitadel_project.idp.id
  role_name    = "admin"
  display_name = "Administrator"
}

# OIDC Web App
resource "zitadel_application_oidc" "console" {
  org_id       = zitadel_organization.platform.id
  project_id   = zitadel_project.idp.id
  name         = "Console UI"

  redirect_uris       = ["https://console.kubexa.tech/callback"]
  response_types      = ["OIDC_RESPONSE_TYPE_CODE"]
  grant_types         = ["OIDC_GRANT_TYPE_AUTHORIZATION_CODE"]
  app_type            = "OIDC_APP_TYPE_WEB"
  auth_method_type    = "OIDC_AUTH_METHOD_TYPE_BASIC"
  access_token_type   = "OIDC_TOKEN_TYPE_JWT"
  access_token_role_assertion = true
  id_token_role_assertion     = true
}

# Machine user for CI
resource "zitadel_machine_user" "ci" {
  org_id      = zitadel_organization.platform.id
  user_name   = "ci-bot@platform"
  name        = "CI/CD Bot"
  with_secret = true
}

# PAT for CI
resource "zitadel_personal_access_token" "ci_pat" {
  org_id          = zitadel_organization.platform.id
  user_id         = zitadel_machine_user.ci.id
  expiration_date = "2028-01-01T00:00:00Z"
}
```

## Importing

```bash
# Most resources use <id>[:<org_id>][:<additional_fields>] format
terraform import zitadel_organization.imported '123456789012345678'
terraform import zitadel_project.imported '123456789012345678:123456789012345678'
terraform import zitadel_human_user.imported '123456789012345678:123456789012345678:Password1!'
terraform import zitadel_application_oidc.imported '123456789012345678:123456789012345678:123456789012345678:123456789012345678@zitadel:...'
terraform import zitadel_system_features.imported 'system'
```

## Common Mistakes

- **PAT not persistent** - `access_token` in provider config is a Terraform variable, not state. Use `random_password` or vault for token management.
- **JWT profile vs PAT** - JWT profile (service account key) is preferred for long-running automation. PAT is simpler for ad-hoc but expires.
- **Org resource version** - `zitadel_org` (backward-compatible) vs `zitadel_organization` (v2 API, 4.x+). Prefer `zitadel_organization` for new deployments on ZITADEL 4.x.
- **Application secrets not stored** - `client_secret` in OIDC/API apps is only in state. Use `resource.random_password` if you need a deterministic/known secret.
- **System API auth required** - `zitadel_system_features`, `zitadel_instance_*` require system API authentication block in provider config.
- **Message text IDs** - Each message text resource uses a constant ID (like `default_init_message_text`). Check the import format for exact values.
- **Default policies are singleton** - Only one `zitadel_default_*` policy resource per instance. Import before managing with Terraform.
- **Action execution after event/function** - Must create the action first, then wire execution triggers as separate resources.
