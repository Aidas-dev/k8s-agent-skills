---
name: netbird-terraform
description: NetBird Terraform provider (netbirdio/netbird) — groups, peers, routes, networks, network resources/routers, policies, setup keys, DNS zones/records, posture checks, users, tokens, SCIM, identity providers, reverse proxy services. Declarative NetBird management via public API.
---

# netbird-terraform — Terraform Provider

For client CLI see `netbird-cli`. For K8s operator see `netbird-operator`.

**Provider:** `netbirdio/netbird` (registry.terraform.io)  
**Latest:** v0.0.9  
**API:** NetBird Management public API (PAT-based)

## Provider Setup

```hcl
variable "netbird_token" {
  sensitive = true
  description = "NetBird Management Access Token (service user PAT)"
}

terraform {
  required_providers {
    netbird = {
      source  = "netbirdio/netbird"
      version = "~> 0.0.9"
    }
  }
}

provider "netbird" {
  token          = var.netbird_token        # or NB_PAT env
  management_url = "https://api.netbird.io" # or NB_MANAGEMENT_URL env
  # tenant_account = "<account-id>"         # or NB_ACCOUNT env (impersonate)
}
```

Env vars: `NB_PAT`, `NB_MANAGEMENT_URL`, `NB_ACCOUNT`. Values in `.tf` take precedence.

## Resources (20)

| Resource | Purpose |
|----------|---------|
| `netbird_group` | Group + peer/resource membership |
| `netbird_peer` | Manage an EXISTING peer (id, name, ssh_enabled, approval, expiry) |
| `netbird_route` | Route to networks (network/domains, peer or peer_groups, masquerade, metric) |
| `netbird_network` | Network container (map of an environment) |
| `netbird_network_resource` | Resource inside a network (address/domain, groups) |
| `netbird_network_router` | Routing peer for a network |
| `netbird_policy` | Access policy (rules: sources/destinations, protocol, ports, SSH) |
| `netbird_setup_key` | Setup key (one-off/reusable, usage limit, auto groups, expiry) |
| `netbird_token` | Personal access token (service user only) |
| `netbird_user` | Invite/manage users (import existing) |
| `netbird_account_settings` | Account-wide settings |
| `netbird_dns_zone` | Custom DNS zone |
| `netbird_dns_record` | DNS record in a zone |
| `netbird_dns_settings` | Account DNS settings (ONLY ONE per provider) |
| `netbird_nameserver_group` | Nameserver group for DNS |
| `netbird_posture_check` | Posture checks (OS version, geo, process, netbird version) |
| `netbird_identity_provider` | IdP integration (self-hosted) |
| `netbird_scim` | SCIM integration (IdP sync) |
| `netbird_reverse_proxy_domain` | Custom reverse-proxy domain |
| `netbird_reverse_proxy_service` | Reverse-proxy service (expose apps publicly) |

## Data Sources (22)

`netbird_peer`, `netbird_group`, `netbird_route`, `netbird_network`, `netbird_network_resource`, `netbird_policy`, `netbird_setup_key`, `netbird_user`, `netbird_token`, `netbird_account_settings`, `netbird_dns_zone`, `netbird_dns_record`, `netbird_dns_settings`, `netbird_nameserver_group`, `netbird_posture_check`, `netbird_identity_provider`, `netbird_scim`, `netbird_reverse_proxy_domain`, `netbird_reverse_proxy_service` (+ lookup helpers). `netbird_setup_key` data source **cannot read the plain key value** — only metadata.

## Common Patterns

### Groups + Peer (manage an existing peer)

```hcl
data "netbird_peer" "example" {
  ip = "1.2.3.4"
}

resource "netbird_group" "example" {
  name  = "TF Test"
  peers = [data.netbird_peer.example.id]
}
```

### Policy (access control)

```hcl
resource "netbird_policy" "example" {
  name    = "TF Test"
  enabled = true

  rule {
    action        = "accept"
    bidirectional = true
    enabled       = true
    protocol      = "tcp"
    name          = "TF Test"
    sources       = [netbird_group.example.id]
    destinations  = [netbird_group.example.id]
  }
}
```

### Network + resource + router (Networks pattern)

```hcl
resource "netbird_network" "example" {
  name        = "TF Test"
  description = "TF Test"
}

resource "netbird_network_resource" "example" {
  network_id  = netbird_network.example.id
  name        = "TF Test"
  description = "TF Test"
  address     = "www.example.com"   # IP, CIDR, or domain/wildcard
  groups      = [netbird_group.example.id]
  enabled     = true
}
```

### Route (legacy routes / exit nodes)

```hcl
resource "netbird_route" "example" {
  network_id            = "Example"
  groups                = [data.netbird_group.example_a.id]
  access_control_groups = [data.netbird_group.example_b.id]
  description           = "Example Route"
  network               = "10.0.0.0/8"     # conflicts with domains
  # domains = ["www.example.com"]           # max 32, dynamic resolution
  peer        = data.netbird_peer.example.id    # XOR peer_groups
  # peer_groups = [data.netbird_group.example_c.id]
  masquerade = true
  metric     = 9999
  enabled    = true
}
```

Route fields: `network` (CIDR) XOR `domains`; `peer` XOR `peer_groups`; `keep_route`, `skip_auto_apply` (exit-node 0.0.0.0/0), `metric` (lower = higher priority).

### SSH policy (netbird-ssh protocol)

```hcl
resource "netbird_policy" "ssh_example" {
  name    = "SSH Access"
  enabled = true

  rule {
    action        = "accept"
    bidirectional = true
    enabled       = true
    protocol      = "netbird-ssh"          # NOT tcp:22 — use this protocol
    name          = "SSH Rule"
    sources       = [netbird_group.example.id]
    destinations  = [netbird_group.example.id]

    authorized_groups = {
      (netbird_group.example.id) = ["example"]   # local usernames per source group
    }
  }
}
```

### Setup key (bulk provisioning)

```hcl
resource "netbird_setup_key" "example" {
  name        = "TF Key"
  type        = "reusable"        # one-off | reusable
  usage_limit = 5                 # 0 = unlimited
  expires_in  = 24                # hours
  auto_groups = [netbird_group.example.id]
  # ephemeral = false
}
```

### Posture check

```hcl
resource "netbird_posture_check" "example" {
  name        = "TF Test"
  description = "Meow"

  netbird_version_check {
    min_version = "0.1.0"
  }

  os_version_check {
    linux_min_kernel_version   = "0.0.0"
    windows_min_kernel_version = "0.0.1"
    darwin_min_version         = "0.0.0"
    ios_min_version            = "0.0.0"
    android_min_version        = "0.0.0"
  }

  geo_location_check {
    locations = [{ country_code = "EG" }, { country_code = "DE" }]
    action    = "allow"
  }

  peer_network_range_check {
    ranges = ["0.0.0.0/0"]
    action = "allow"
  }

  process_check {
    linux_path = "/some/path/in/linux"
    mac_path   = "/some/path/in/mac"
  }
}
```

## Import

```bash
terraform import netbird_group.example <group_id>
terraform import netbird_policy.example <policy_id>
terraform import netbird_route.example <route_id>
terraform import netbird_network_resource.example <network_id>/<resource_id>
terraform import netbird_user.example <user_id>   # existing users MUST be imported
```

## Common Mistakes

- **`netbird_setup_key` data source can't read the key** — use the resource output for provisioning, or fetch the key from the management UI.
- **`netbird_peer` doesn't create peers** — it manages an existing peer (by id/ip). Peers appear when clients register; use setup keys for automation.
- **Policy `protocol: netbird-ssh`** — SSH access is NOT `tcp` + port 22. The provider has a dedicated protocol; use it with `authorized_groups`.
- **`dns_settings` uniqueness** — exactly ONE `netbird_dns_settings` resource per provider/account. A second plan fights it.
- **`route.network` vs `domains` are mutually exclusive** — pick one per route. Same for `peer` vs `peer_groups`.
- **Only one source of truth per resource** — if the operator's `SetupKey` CRD manages a key, don't also manage it in Terraform (drift war).
- **Existing users must be imported** — `netbird_user` create invites; re-creating existing users fails. Import them.
