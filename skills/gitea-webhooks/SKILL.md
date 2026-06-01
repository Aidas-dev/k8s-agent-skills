---
name: gitea-webhooks
description: Use when managing Gitea webhooks — creating webhooks via API, understanding event types, system vs repo vs org webhooks, payload verification with HMAC-SHA256, and supported webhook formats.
---

# Gitea Webhooks (v1.26)

## Events

| Event | Trigger |
|-------|---------|
| `push` | Git push |
| `create` / `delete` | Branch/tag created or deleted |
| `fork` | Repo forked |
| `issues` | Issue opened/closed/reopened/edited/deleted |
| `issue_assign` | Issue assigned/unassigned |
| `issue_label` | Issue labels updated |
| `issue_comment` | Comment on issue |
| `pull_request` | PR opened/closed/reopened/edited/synced |
| `pull_request_review_approved` | PR approved |
| `pull_request_review_rejected` | PR rejected |
| `pull_request_review_comment` | PR review comment |
| `pull_request_review_request` | Review requested |
| `wiki` | Wiki edited |
| `repository` | Repo created/deleted |
| `release` | Release published/updated/deleted |
| `package` | Package published |
| `status` | Commit status updated |
| `workflow_run` | Actions workflow run completed |
| `workflow_job` | Actions workflow job ready (autoscaling trigger) |

## API

```bash
# Create repo webhook
curl -X POST https://git.example.com/api/v1/repos/{owner}/{repo}/hooks \
  -H "Authorization: token <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "gitea",
    "config": {
      "url": "https://example.com/webhook",
      "content_type": "json",
      "secret": "my-secret"
    },
    "events": ["push", "pull_request"],
    "active": true
  }'

# List webhooks
curl https://git.example.com/api/v1/repos/{owner}/{repo}/hooks \
  -H "Authorization: token <token>"

# Delete webhook
curl -X DELETE https://git.example.com/api/v1/repos/{owner}/{repo}/hooks/{id} \
  -H "Authorization: token <token>"
```

## System vs Repo Webhooks

| Type | Scope | API Endpoint |
|------|-------|-------------|
| Repository | Single repo | `POST /api/v1/repos/{owner}/{repo}/hooks` |
| Organization | All repos in org | `POST /api/v1/orgs/{org}/hooks` |
| System | All repos instance-wide | `POST /api/v1/admin/hooks` (`config.is_system_webhook: true`) |
| Default | Template for new repos | `POST /api/v1/admin/hooks` (`config.is_system_webhook: false`) |

## Payload Verification

Gitea signs the raw request body with HMAC-SHA256 using the configured secret.

**Header:** `X-Gitea-Signature`

**Verify server-side:**
```python
import hmac, hashlib

signature = request.headers.get("X-Gitea-Signature")
expected = hmac.new(secret.encode(), request.body, hashlib.sha256).hexdigest()
if not hmac.compare_digest(signature, expected):
    abort(401)
```

## Webhook Types (format)

`gitea` (generic JSON), `gogs`, `slack`, `discord`, `dingtalk`, `telegram`, `msteams`, `feishu`, `wechatwork`, `packagist`.

## Common Mistakes

- **HMAC algorithm** — Gitea uses SHA256, not SHA1. Check `X-Gitea-Signature` header.
- **System webhook flag** — Must set `config.is_system_webhook` to `true` for instance-wide webhooks. Defaults to false (template webhook).
- **Event names** — Use snake_case in API: `pull_request`, `issue_comment`. Verify against docs.
- **workflow_job for autoscaling** — Use `workflow_job` event (not `workflow_run`) to trigger ephemeral runner spin-up.
