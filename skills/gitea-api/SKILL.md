---
name: gitea-api
description: Use when working with the Gitea REST API — authentication, token management, repository/issue/PR CRUD, package management, admin operations, and general API automation with curl.
---

# Gitea REST API (v1.26)

Base: `https://gitea.example.com/api/v1`. Swagger: `https://gitea.example.com/api/swagger`. OpenAPI spec: `https://gitea.example.com/swagger.v1.json`.

## Authentication

| Method | Header / Param | Use Case |
|--------|---------------|----------|
| Token | `Authorization: token <token>` | API tokens from Settings > Applications |
| Basic Auth | `-u username:password` | Creating tokens (`POST /users/:name/tokens`) |
| OAuth2 Bearer | `Authorization: bearer <token>` | OAuth2 access tokens |
| URL param | `?token=<token>` | Simple scripts |
| SSH signatures | `Signature` header with SSH key | mTLS-style auth via SSH |

2FA: add header `X-Gitea-OTP: 123456` with Basic Auth.

## Token Scopes

`"scopes": ["all"]` or fine-grained: `["repo", "write:repository", "read:user", "read:organization"]`. Create via UI or `POST /api/v1/users/{username}/tokens`.

## Key Endpoints

### Users & Auth

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/user` | Current user info |
| GET | `/users/{username}` | Get user |
| POST | `/users/{username}/tokens` | Create API token (Basic Auth required) |

### Repositories

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/repos/{owner}/{repo}` | Get repository |
| POST | `/repos/{owner}/{repo}` | Create repository |
| POST | `/repos/{owner}/{repo}/mirrors` | Sync mirror |

### Issues & Pull Requests

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/repos/{owner}/{repo}/issues` | Create issue |
| GET | `/repos/{owner}/{repo}/issues` | List issues (filters: state, labels, created_by) |
| POST | `/repos/{owner}/{repo}/pulls` | Create pull request |
| GET | `/repos/{owner}/{repo}/pulls` | List PRs |

### Organizations

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/orgs/{org}` | Get organization |
| POST | `/orgs/{org}/teams` | Create team |

### Admin

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/admin/users` | List users (supports sorting/filtering) |
| POST | `/admin/users` | Create user |
| POST | `/admin/hooks` | Create system/default webhook |
| GET | `/admin/hooks?type=system\|default` | List admin webhooks |

### Packages

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/packages/{owner}` | List packages |
| DELETE | `/packages/{owner}/{type}/{name}/{version}` | Delete package version |

## Examples

```bash
# Create an issue
curl -X POST https://git.example.com/api/v1/repos/myorg/myrepo/issues \
  -H "Authorization: token <token>" \
  -H "Content-Type: application/json" \
  -d '{"title": "Bug", "body": "Description", "labels": [1, 2]}'

# List PRs with filters
curl "https://git.example.com/api/v1/repos/myorg/myrepo/pulls?state=open&sort=recentupdate" \
  -H "Authorization: token <token>"

# Create API token (Basic Auth required)
curl -X POST https://git.example.com/api/v1/users/myuser/tokens \
  -u myuser:mypassword \
  -H "Content-Type: application/json" \
  -d '{"name": "ci-token", "scopes": ["repo", "write:repository"]}'

# Delete a package version
curl -X DELETE "https://git.example.com/api/v1/packages/myorg/container/myapp/1.0.0" \
  -H "Authorization: token <token>"
```

## Common Mistakes

- **2FA users** — must pass `X-Gitea-OTP` header with Basic Auth when creating tokens
- **Token as query param** — `?token=...` works but logs expose it. Prefer header.
- **Package delete needs all segments** — owner, type, name, version. Missing one = 404.
