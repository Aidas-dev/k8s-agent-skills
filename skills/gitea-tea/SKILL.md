---
name: gitea-tea
description: Work with Gitea using tea CLI for auth, repo, issue, pull request, release, actions, webhook, and notification workflows. Use when user references Gitea, self-hosted git forges, or asks for tea commands.
---

# Gitea + tea CLI

Use this skill when tasks target Gitea and the `tea` CLI, especially if the user would normally use `gh` on GitHub.

## When To Use

- User mentions Gitea, self-hosted git, forgejo-compatible flows, or `tea`
- You need CLI automation for issues, pull requests, repos, comments, or releases
- You need host-specific auth profiles (multiple Gitea instances)
- Managing Gitea Actions (secrets, variables, runners, workflow runs)
- Managing webhooks, notifications, or admin tasks

## Core Rules

1. Prefer `tea` over `gh` for Gitea targets.
2. Always scope commands to target explicitly when ambiguity exists:
   - `--repo owner/name`
   - `--login <profile>`
   - `--remote <remote-name>`
3. If command flags differ by tea version, run `tea <cmd> --help` first and follow local help text.
4. For PR/review actions, ensure local branches are pushed before create/merge operations.

## Auth Workflow

### Create login profile (token)

```bash
tea logins add --name <profile> --url https://git.example.com --token <token>
```

### Verify logins / switch default

```bash
tea logins ls
tea logins default <profile>
```

### Quick auth check

```bash
tea whoami                         # Show current user
```

### Optional env-based auth (automation)

- `GITEA_SERVER_URL`
- `GITEA_SERVER_TOKEN`
- `GITEA_SERVER_USER`
- `GITEA_SERVER_PASSWORD`

## Command Mapping (gh -> tea)

- Repo list: `gh repo list` -> `tea repos ls`
- Issue list: `gh issue list` -> `tea issues ls`
- Issue create: `gh issue create` -> `tea issues create`
- PR list: `gh pr list` -> `tea pulls ls` or `tea pr ls`
- PR create: `gh pr create` -> `tea pulls create`
- PR checkout: `gh pr checkout <n>` -> `tea pulls checkout <n>`
- Comment on PR/issue: `gh pr comment` / `gh issue comment` -> `tea comment <index> [body]`
- Release list/create: `gh release list/create` -> `tea releases ls` / `tea releases create`

## Practical Patterns

### Work on a specific repo

```bash
tea issues ls --repo owner/name --login <profile>
tea pulls ls --repo owner/name --login <profile>
```

### Create issue / PR

```bash
tea issues create --repo owner/name --title "Title" --description "Details"
tea pulls create --repo owner/name --head <branch> --title "Title" --description "Details"
```

### Review / merge PR

```bash
tea pulls review <index>
tea pulls approve <index>
tea pulls merge <index>
```

### Release workflow

```bash
tea releases create --repo owner/name --tag vX.Y.Z --title "vX.Y.Z"
tea releases assets --help
```

### Clone repo

```bash
tea clone <repo-slug> [target-dir]
tea clone gitea/tea                                 # Short form
tea clone https://git.example.com/owner/repo.git    # Full URL
```

Supports various slug formats: `owner/repo`, `host/owner/repo`, full URLs. Overrides login when host specified.

## Gitea Actions

### Repository actions (secrets, variables, runs, workflows)

```bash
tea actions secrets ls --repo owner/name
tea actions secrets create --repo owner/name <name> <value>

tea actions variables ls --repo owner/name
tea actions variables set --repo owner/name <key> <value>

tea actions runs ls --repo owner/name
tea actions runs view <run-id> --repo owner/name
tea actions runs logs <run-id> --repo owner/name

tea actions workflows ls --repo owner/name
tea actions workflows dispatch <workflow-id> --repo owner/name
tea actions workflows enable <workflow-id> --repo owner/name
tea actions workflows disable <workflow-id> --repo owner/name
```

### Host-mode runner on Talos

Talos has no Docker socket, so standard dind runners won't work. Use host-mode runner:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitea-runner
spec:
  template:
    spec:
      containers:
        - name: runner
          image: gitea/runner:latest
          env:
            - name: GITEA_INSTANCE_URL
              value: https://git.example.com
            - name: GITEA_RUNNER_REGISTRATION_TOKEN
              valueFrom:
                secretKeyRef:
                  name: gitea-runner-token
                  key: token
            - name: GITEA_RUNNER_LABELS
              value: "ubuntu-latest:host"    # Host mode = no Docker
```

- Label must include `:host` suffix for host-mode execution
- Job steps run as processes inside the runner container
- Runner container needs tools that workflows require (curl, git, etc.)
- For Kubernetes-specific workflows, mount kubeconfig or use SA token

## Notifications

```bash
tea notifications ls               # List unread + pinned (default)
tea notifications ls --types issue,pull  # Filter by type
tea notifications read <id>        # Mark as read
tea notifications unread <id>      # Mark as unread
```

Defaults to current repo only. Use `--mine` for cross-repo notifications.

## Webhooks

```bash
tea webhooks ls --repo owner/name
tea webhooks create https://hook.example.com/endpoint --repo owner/name --type gitea --events push
tea webhooks create --org my-org --type slack --events issues --secret <secret>
tea webhooks delete <id> --repo owner/name
```

`--type` options: `gitea`, `gogs`, `slack`, `discord`, `dingtalk`, `telegram`, `msteams`, `feishu`, `wechatwork`, `packagist`. Default events: `push`. Supports `--org` (org-level hooks) and `--global` (instance-wide).

## Admin

```bash
tea admin users ls                 # List registered users
tea admin users list --login admin # Requires admin token
```

Only `users` subcommand currently. Requires login with admin privileges.

## Safety + Verification

- Run read-only commands first (`ls`, `list`, `view`) before mutating operations.
- If repo context unclear, require explicit `--repo` and `--login`.
- Before destructive actions (`delete`, `close`, `merge`), confirm target index and repository.
- For actions: check workflow runs with `ls` before `cancel`/`delete`.
- If command fails, capture `tea <subcommand> --help` output and adapt flags to installed version.

## Common Mistakes

- **Wrong login profile** — Running `tea` commands without `--login` uses the default profile. If multiple Gitea instances are configured, always pass `--login <profile>`.
- **Missing push before PR** — `tea pulls create` does not push your branch. Run `git push -u origin <branch>` first.
- **teav1 vs v0 flags** — Ensure `tea` version supports the flags you're using. Run `tea <cmd> --help` to verify before scripting.
- **Webhook secrets in output** — `tea webhooks create` with `--secret` may leak the secret in shell history. Use `--secret-stdin` or env vars.
- **Actions runner registration** — Token must be pre-created in Gitea admin UI (Settings → Actions → Runners). The CLI `tea actions` commands don't create tokens.

## Troubleshooting

- Auth failures: re-check login profile URL/token; verify `tea logins ls`.
- Wrong target host/repo: pass both `--login` and `--repo` explicitly.
- PR creation errors: ensure branch is pushed and upstream is configured.
- TLS issues on internal hosts: prefer proper CA trust; use insecure flags only when user explicitly accepts risk.
- Actions runner registration: pre-create the runner token in Gitea UI (Settings > Actions > Runners) or use API.
