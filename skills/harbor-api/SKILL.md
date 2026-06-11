---
name: harbor-api
description: Use when working with the Harbor REST API v2 — project management, artifact operations, robot accounts, replication, vulnerability scanning, OIDC/LDAP config, garbage collection, and general API automation with curl.
---

# Harbor REST API v2

Base: `/api/v2.0`. Latest stable: **Harbor v2.15.1** (May 2026). API spec: Swagger 2.0 at `api/v2.0/swagger.yaml`. Built-in Swagger UI: `https://<harbor>/devcenter-api-2.0`.

## Authentication

| Method | Header / Usage | Use Case |
|--------|---------------|----------|
| Basic Auth | `-u username:password` | Direct admin/developer API access |
| Bearer Token | `Authorization: Bearer <token>` | Obtained from `/service/token` per Docker Registry v2 spec |
| Robot Account | `-u robot$<prefix><name>:<secret>` | Automated CI/CD with scoped permissions |
| OIDC ID Token | `Authorization: Bearer <oidc_id_token>` | OIDC-authenticated users (basic auth not supported for OIDC) |

### Bearer Token Flow
```bash
# Get token for push/pull access to a repo
TOKEN=$(curl -s -u "username:password" \
  "https://harbor.example.com/service/token?service=harbor-registry&scope=repository:project/repo:pull,push" \
  | jq -r '.token')

# Use for registry operations
curl -H "Authorization: Bearer $TOKEN" https://harbor.example.com/v2/_catalog
```

### Robot Account Notes
- Secret shown **only once** at creation — Harbor does not store it
- Username format: `robot$<prefix><account_name>`
- System-level (v2.2.0+) or project-level scope
- Permissions: granular RBAC (push, pull, create, read, delete, list, etc.)

## Key Endpoints

### Health & Status

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/health` | Component health (no auth) |
| GET | `/statistics` | Project & repo statistics |
| GET | `/search?q=<query>` | Search projects, repos, helm charts |

### Project Management

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/projects` | List projects (filter: name, public, owner) |
| POST | `/projects` | Create project |
| HEAD | `/projects` | Check project name exists |
| GET | `/projects/{name_or_id}` | Get project |
| PUT | `/projects/{name_or_id}` | Update project |
| DELETE | `/projects/{name_or_id}` | Delete project |
| GET | `/projects/{name_or_id}/_deletable` | Check if deletable |
| GET | `/projects/{name_or_id}/summary` | Project summary |
| GET | `/projects/{name_or_id}/metadatas` | List metadata |
| POST | `/projects/{name_or_id}/metadatas` | Add metadata |
| GET/PUT/DELETE | `/projects/{name_or_id}/metadatas/{meta_name}` | CRUD metadata entry |
| GET | `/projects/{name_or_id}/members` | List members |
| POST | `/projects/{name_or_id}/members` | Add member |
| GET/PUT/DELETE | `/projects/{name_or_id}/members/{mid}` | CRUD member |

### Repository Management

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/repositories` | List all authorized repos |
| GET | `/projects/{project}/repositories` | List repos in project |
| GET | `/projects/{project}/repositories/{repo}` | Get repo |
| PUT | `/projects/{project}/repositories/{repo}` | Update repo description |
| DELETE | `/projects/{project}/repositories/{repo}` | Delete repo |

### Artifact Management

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/projects/{proj}/repositories/{repo}/artifacts` | List artifacts (`?q=tags=*`, labels, etc.) |
| POST | `/projects/{proj}/repositories/{repo}/artifacts` | Copy artifact |
| GET | `/projects/{proj}/repositories/{repo}/artifacts/{ref}` | Get artifact by digest or tag |
| DELETE | `/projects/{proj}/repositories/{repo}/artifacts/{ref}` | Delete artifact |
| PUT | `/projects/{proj}/repositories/{repo}/artifacts/{ref}/add-label` | Add label |
| DELETE | `/projects/{proj}/repositories/{repo}/artifacts/{ref}/labels/{label_id}` | Remove label |
| GET | `/projects/{proj}/repositories/{repo}/artifacts/{ref}/tags` | List tags |
| POST | `/projects/{proj}/repositories/{repo}/artifacts/{ref}/tags` | Create tag |
| DELETE | `/projects/{proj}/repositories/{repo}/artifacts/{ref}/tags/{tag}` | Delete tag |

### Vulnerability Scanning

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/projects/{proj}/repositories/{repo}/artifacts/{ref}/scan` | Trigger scan |
| POST | `/projects/{proj}/repositories/{repo}/artifacts/{ref}/scan/stop` | Stop scan |
| GET | `/scanners` | List scanners |
| GET | `/scanners/{id}` | Get scanner metadata |
| POST | `/scanners/ping` | Ping scanner adapter |
| GET | `/projects/{proj}/repositories/{repo}/artifacts/{ref}/scan/{report_id}` | Get scan report |

### Robot Accounts

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/robots` | List robot accounts |
| POST | `/robots` | Create robot account |
| GET | `/robots/{id}` | Get robot |
| PUT | `/robots/{id}` | Update robot |
| DELETE | `/robots/{id}` | Delete robot |
| PATCH | `/robots/{id}` | Refresh robot secret |

### Replication

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/replication/policies` | List policies |
| POST | `/replication/policies` | Create policy |
| GET/PUT/DELETE | `/replication/policies/{id}` | CRUD policy |
| GET | `/replication/executions` | List executions |
| POST | `/replication/executions` | Start replication |
| GET | `/replication/executions/{id}` | Get execution status |
| GET | `/replication/executions/{id}/tasks` | List execution tasks |
| GET | `/registries` | List registries |
| POST | `/registries` | Create registry endpoint |
| GET/PUT/DELETE | `/registries/{id}` | CRUD registry |
| POST | `/registries/ping` | Ping registry endpoint |

### Garbage Collection

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/system/gc` | List GC schedules |
| POST | `/system/gc` | Create GC schedule |
| GET | `/system/gc/{id}` | Get GC job |
| GET | `/system/gc/{id}/log` | Get GC log |

### OIDC / LDAP

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/ldap/ping` | Ping LDAP |
| GET | `/ldap/users/search` | Search LDAP users |
| POST | `/ldap/users/import` | Import LDAP users |
| GET | `/ldap/groups/search` | Search LDAP groups |
| POST | `/system/oidc/ping` | Ping OIDC provider |
| GET | `/configurations` | Get system config (auth_mode, oidc, ldap) |
| PUT | `/configurations` | Update system config |

### System & Admin

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/systeminfo` | System info |
| GET | `/systeminfo/volumes` | Storage volume info |
| GET | `/internalconfig` | Internal config (admin only) |
| GET | `/labels` | List labels |
| POST | `/labels` | Create label |
| GET/PUT/DELETE | `/labels/{id}` | CRUD label |
| GET | `/usergroups` | List user groups |
| POST | `/usergroups` | Create user group |
| GET/PUT/DELETE | `/usergroups/{id}` | CRUD user group |
| GET | `/preheat/policies` | List preheat policies |
| POST | `/preheat/policies` | Create preheat policy |
| GET | `/preheat/instances` | List preheat instances |
| POST | `/preheat/instances` | Create preheat instance |
| GET | `/audit-logs` | List audit logs |
| GET | `/quota` | List storage quotas |

## Examples

```bash
# Create a project
curl -X POST https://harbor.example.com/api/v2.0/projects \
  -u "admin:Harbor12345" \
  -H "Content-Type: application/json" \
  -d '{"project_name": "myapp", "public": false, "storage_limit": -1}'

# Create a robot account (system-level)
curl -X POST https://harbor.example.com/api/v2.0/robots \
  -u "admin:Harbor12345" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "ci-deploy",
    "description": "CI/CD deployment robot",
    "level": "system",
    "permissions": [{
      "kind": "project",
      "namespace": "myapp",
      "access": [
        {"resource": "repository", "action": "pull"},
        {"resource": "repository", "action": "push"}
      ]
    }]
  }'

# Trigger artifact scan
curl -X POST "https://harbor.example.com/api/v2.0/projects/myapp/repositories/nginx/artifacts/latest/scan" \
  -u "admin:Harbor12345"

# Create a replication rule
curl -X POST https://harbor.example.com/api/v2.0/replication/policies \
  -u "admin:Harbor12345" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "backup-to-dr",
    "description": "Replicate to DR site",
    "trigger": {"type": "event_based"},
    "dest_registry": {"id": 2},
    "filters": [{"type": "name", "value": "myapp/**"}],
    "deletion": true,
    "override": true
  }'

# Search across Harbor
curl -s "https://harbor.example.com/api/v2.0/search?q=nginx" \
  -u "admin:Harbor12345"

# Trigger garbage collection
curl -X POST https://harbor.example.com/api/v2.0/system/gc \
  -u "admin:Harbor12345" \
  -H "Content-Type: application/json" \
  -d '{"schedule": {"type": "Weekly", "weekday": 0, "offtime": 0}}'
```

## Common Mistakes

- **Robot secret not saved** — Secret is only returned on creation. Store it immediately.
- **API version path** — Always use `/api/v2.0/`, not `/api/` (v1.x legacy path).
- **OIDC users can't use basic auth** — Must use OIDC ID token as Bearer token.
- **Robot tokens auto-expire** — Set `duration` in days on creation (default: no expiry).
- **Scan reports deleted on v2.2 upgrade** — Schema migration clears old scan data. Re-scan after upgrade.
- **Bearer token scope** — Token is scoped to the `scope` param in the `/service/token` request. Use `repository:*:pull` for read-only, `repository:*:pull,push` for write.
