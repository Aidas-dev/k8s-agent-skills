---
name: gitea-runner
description: Use when configuring Gitea Actions runners — registration, deployment modes, Docker vs host mode, Talos/Kubernetes host-mode, ephemeral runners, and v1.26 Actions features.
---

# Gitea Runner (v1.0.6)

Formerly `act_runner`. Renamed in v1.0.0 (May 2026). Binary: `gitea-runner`. Image: `gitea/runner`. Latest: **v1.0.6**.

## Registration

Tokens at instance/admin, org, or repo level in Settings > Actions > Runners.

```bash
# Interactive
gitea-runner register

# Non-interactive
gitea-runner register --no-interactive \
  --instance https://git.example.com \
  --token <token> \
  --name my-runner \
  --labels ubuntu-latest:docker://node:20-bookworm

# Ephemeral (single-job runner, then exits)
gitea-runner register --no-interactive --ephemeral \
  --instance https://git.example.com \
  --token <token>
```

Token also via CLI: `gitea --config /etc/gitea/app.ini actions generate-runner-token`.

## Running

```bash
gitea-runner daemon --config config.yaml
```

Generate default config: `gitea-runner generate-config > config.yaml`.

## Labels Format

```
<label>:<scheme>://<image>
```

- `:host` suffix — runs step on host (no Docker required)
- `:docker://image` — runs step in container
- `ubuntu-latest:docker://node:20-bookworm` — label `ubuntu-latest` runs in `node:20-bookworm` container

## Deployment Modes

| Mode | Description | Docker Required |
|------|-------------|----------------|
| **Host** | Steps run directly on host process | No |
| **Docker** | Steps in sibling containers via socket | Yes (`/var/run/docker.sock`) |
| **DinD** | Rootless, own Docker daemon in container | Yes (privileged container) |

### Kubernetes / Talos (Host Mode)

Talos has no Docker socket. Must use host mode with `:host` labels:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitea-runner
  namespace: gitea
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitea-runner
  template:
    metadata:
      labels:
        app: gitea-runner
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
                  name: runner-token
                  key: token
            - name: GITEA_RUNNER_LABELS
              value: "ubuntu-latest:host"
```

Runner container must include tools its workflows need (curl, git, node, etc.).

### Docker Mode

```bash
docker run -d \
  -e GITEA_INSTANCE_URL=https://git.example.com \
  -e GITEA_RUNNER_REGISTRATION_TOKEN=<token> \
  -e GITEA_RUNNER_NAME=runner-1 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitea/runner:latest
```

## Ephemeral Runners

Set `GITEA_RUNNER_EPHEMERAL=1`. Runner runs exactly one job then exits. Use `workflow_job` webhook to autoscale (spin up new runner per job).

## v1.26 Actions Features

- **Concurrency groups** — GitHub-style `concurrency:` in workflows
- **Per-runner disable/pause** — toggle via UI, no unregister needed
- **Rerun failed jobs** — button in UI
- **Configurable token permissions** — `permissions:` block in workflows
- **Non-zipped artifacts** — action v7 format
- **Private repo workflows** — reusable workflows from private repos
- **Workflow dependencies graph** — visual dependency tree in UI

## Common Mistakes

- **Stale image** — `gitea/act_runner` is deprecated. Use `gitea/runner`.
- **No `:host` on Talos** — Without Docker socket, labels must use `:host` suffix. Runner will fail trying to spawn containers.
- **Registration token via GET removed** — v1.26 removed `GET /api/v1/admin/runners/registration-token`. Use UI or `gitea actions generate-runner-token`.
- **Cache config** — `cache.external_server` now requires `cache.external_secret` (v1.0.0+). Set both or cache fails.
