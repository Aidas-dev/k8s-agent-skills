---
name: tekton
description: Use when authoring Tekton PipelineRun/TaskRun YAML, configuring resolvers, setting node affinity/nodeSelector for CI workloads, using matrix+param dynamic scheduling, writing when/CEL expressions, or troubleshooting Tekton v1 API patterns.
---

# Tekton Pipelines

Kubernetes-native CI/CD. Covers **tekton.dev/v1** patterns.

## When to Use

- Authoring PipelineRun / TaskRun YAML
- Configuring git/cluster/hub resolvers
- Setting nodeSelector/tolerations/affinity for CI pods
- Dynamic per-task node targeting via taskRunSpecs
- Matrix + param dynamic scheduling
- CEL when expressions
- Timeout hierarchy
- finally tasks

**Not for:** Tekton operator install/upgrade, CRDs, cluster-level config.

## Core Patterns

### Resolver-Based Task Refs

Prefer resolvers over embedded TaskSpecs:

```yaml
tasks:
  - name: build
    taskRef:
      resolver: git
      params:
        - name: url
          value: https://github.com/org/repo
        - name: revision
          value: main
        - name: pathInRepo
          value: ci/tasks/build.yaml
```

Resolvers: `git`, `cluster`, `hub`. See TTL section below for cache config.

### Git Authentication (Private Repos)

Authenticate against private Git repositories via the ServiceAccount's `secrets` — Tekton's `creds-init` init container reads `kubernetes.io/basic-auth` secrets automatically and configures git before any task step runs. No workspace plumbing needed.

```yaml
# 1. Create the secret with annotation targeting the host
apiVersion: v1
kind: Secret
metadata:
  name: git-creds
  annotations:
    tekton.dev/git-0: https://git.example.com  # host-wide, any repo
type: kubernetes.io/basic-auth
stringData:
  username: <user-or-bot-name>
  password: <pat-or-token>

# 2. Attach to the PipelineRun's ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default  # or a dedicated pipeline SA
secrets:
  - name: git-creds
```

For multiple hosts, increment the annotation index:
```yaml
metadata:
  annotations:
    tekton.dev/git-0: https://github.com
    tekton.dev/git-1: https://gitlab.com
```

Per-repo targeting (full path):
```yaml
metadata:
  annotations:
    tekton.dev/git-0: https://github.com/org/repo
```

**git-clone task with basic-auth workspace** (alternative approach):
```yaml
workspaces:
  - name: basic-auth
    secret:
      secretName: git-basic-auth-creds  # Opaque secret with .gitconfig + .git-credentials keys
```

The SA-based approach is preferred — it works for ALL git operations in the PipelineRun (clone, tag, push), not just the git-clone task. It also works automatically with the hub resolver's git resolution.

### Node Targeting via podTemplate

Uniform placement across pipeline:

```yaml
taskRunTemplate:
  podTemplate:
    nodeSelector:
      role: ci-worker
    tolerations:
      - key: ci
        operator: Exists
        effect: NoSchedule
```

Per-task override via `taskRunSpecs`:

```yaml
taskRunSpecs:
  - pipelineTaskName: gpu-build
    taskPodTemplate:
      nodeSelector:
        node-type: gpu
```

### Matrix + Param Dynamic Scheduling

Drive node selection from array params:

```yaml
params:
  - name: architectures
    type: array
    default: ["amd64", "arm64"]

tasks:
  - name: build
    params:
      - name: arch
        value: "$(params.arch)"
    matrix:
      params:
        - name: arch
          value: "$(params.architectures)"
    taskRunTemplate:
      podTemplate:
        nodeSelector:
          kubernetes.io/arch: "$(params.arch)"
```

### When / CEL

Standard conditions:
```yaml
when:
  - input: "$(params.deploy-env)"
    operator: in
    values: ["staging", "production"]
```

CEL (v1.13+):
```yaml
when:
  - cel: "body.action in ['opened', 'synchronize'] && body.repository.fork == false"
```

### Finally Tasks

```yaml
finally:
  - name: notify
    taskRef:
      resolver: git
      params:
        - name: url
          value: https://github.com/org/repo
        - name: pathInRepo
          value: ci/tasks/notify.yaml
    when:
      - input: "$(tasks.build.status)"
        operator: in
        values: ["Succeeded", "Failed"]
```

### Timeout Hierarchy

```
PipelineRun (1h) → TaskRun (1h) → Step
```

Set step timeouts for external ops:
```yaml
steps:
  - name: git-clone
    timeout: "2m"
    script: |
      git clone $(params.repo-url)
```

### Resolver Cache TTL

Caches resolved task/pipeline refs to reduce source API calls and speed up runs. Default: 5min TTL, 1000 entries.

Two levels:

**Global** — ConfigMap `resolver-cache-config`:
```yaml
data:
  default-ttl: "5m"   # s/m/h suffixes
  max-size: "1000"
```

**Per-resolver** — `default-ttl` in resolver's own ConfigMap (v1.13+):
```yaml
metadata:
  name: git-resolver-config
data:
  default-ttl: "1h"
  fetch-timeout: "1m"
```

Per-resolver `default-cache-mode`: `always`, `never`, `auto`.

## Quick Reference

| Pattern | Mechanism | Scope |
|---------|-----------|-------|
| Uniform node placement | `taskRunTemplate.podTemplate` | Pipeline-wide |
| Per-task node placement | `taskRunSpecs[].taskPodTemplate` | Single task |
| Dynamic arch/OS scheduling | matrix + param substitution | Task matrix |
| Conditional execution | `when` / CEL | Per task/finally |
| Task reference | resolver (git/cluster/hub) | Task definition |
| Post-pipeline cleanup | finally tasks | Pipeline end |
| Timeout control | timeout fields | Pipeline/Task/Step |

## Common Mistakes

- **Using `v1beta1`** → Use `tekton.dev/v1`
- **Inline TaskSpec** → Use resolvers
- **podTemplate on every task** → taskRunSpecs for per-task, podTemplate for uniform
- **CEL in wrong field** → Use `cel:` not `input:`/`values:`
- **forgetting finally timeout** → Set explicitly; inherits pipeline default
- **matrix with scalar param** → matrix requires array-typed params
