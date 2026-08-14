---
name: tekton-pac
description: Pipelines-as-Code (tektoncd/pipelines-as-code) — Git-native CI for Tekton. PipelineRuns in .tekton/ triggered by Git events; Repository CR, matching annotations, dynamic variables, ChatOps commands, Gitea/Forgejo/GitHub/GitLab providers, and the tkn pac CLI.
---

# tekton-pac — Pipelines-as-Code

For authoring pipeline YAML see `tekton-pipelines`. For installing Tekton components see `tekton-operator`.

**Repo:** github.com/tektoncd/pipelines-as-code  
**Latest:** v0.49.0 (2026-07-06). **No official Helm chart** — install via the vendored `release.k8s.yaml` release asset or the operator's `OpenShiftPipelinesAsCode` CR (K8s since operator v0.80.0).  
**API group:** `pipelinesascode.tekton.dev/v1alpha1`

## Model

Pipelines live **in the repo** (`.tekton/` dir), versioned with your code. A `Repository` CR tells PaC which repo events to handle and where to run. PaC controller: receives webhook → matches event to Repository CR → fetches `.tekton/` → resolves remote tasks (bundles each PipelineRun self-contained) → creates PipelineRuns → reports status back as PR checks/comments.

**Providers:** GitHub (App + Webhook), GitLab, Bitbucket Cloud/DC, Forgejo, Gitea.

## Install

```bash
# Standalone (vendored release — pins exact version, no chart)
kubectl apply -f release.k8s.yaml    # tektoncd/pipelines-as-code release asset

# Or via operator (v0.80.0+)
# TektonConfig spec.platforms.kubernetes.pipelinesAsCode.enable: true
```

## Repository CR

One per git repo; created in the namespace where PipelineRuns run. **Cannot be created in the PaC namespace itself.**

```yaml
apiVersion: pipelinesascode.tekton.dev/v1alpha1
kind: Repository
metadata:
  name: my-repo
  namespace: tekton-tasks
spec:
  url: "https://git.example.com/org/my-repo"
  git_provider:
    type: "gitea"                # gitea | forgejo | github | gitlab | bitbucket-cloud | bitbucket-datacenter
    url: "http://gitea-http.gitea:3000"   # provider API base
    secret:
      name: "gitea-webhook-config"
      key: "provider.token"
    webhook_secret:
      name: "gitea-webhook-config"
      key: "webhook.secret"
  params:                        # optional custom params → {{ param }} / CEL variables
    - name: docker_registry
      value: "registry.example.com"
      filter: pac.event_type == "pull_request"
```

- A **mutating admission webhook** enforces one Repository CR per URL cluster-wide (disable = security risk: repo hijacking).
- `pipelinesascode.tekton.dev/target-namespace` annotation on a PipelineRun pins execution namespace.

## Matching Annotations (on the PipelineRun in `.tekton/`)

| Annotation | Purpose |
|------------|---------|
| `pipelinesascode.tekton.dev/on-event` | `[pull_request, push, incoming]` |
| `pipelinesascode.tekton.dev/on-target-branch` | `[main]`, `[refs/heads/*]`, `[refs/tags/1.*]` (globs OK) |
| `pipelinesascode.tekton.dev/on-path-change` | glob patterns, comma-separated `[terraform/environments/cluster/**]`; `on-path-change-ignore` to skip |
| `pipelinesascode.tekton.dev/on-comment` | regex match on PR comment, e.g. `^/apply` (sets `{{ trigger_comment }}`) |
| `pipelinesascode.tekton.dev/on-label` | match PR labels `[bug, defect]` |
| `pipelinesascode.tekton.dev/on-cel-expression` | full CEL — **takes priority, ignores other annotations** when present |
| `pipelinesascode.tekton.dev/cancel-in-progress` | `"true"` — cancel older run on new push (after new run starts) |
| `pipelinesascode.tekton.dev/task` | fetch Task from repo-relative path: `./infrastructure/.../task.yaml` |
| `pipelinesascode.tekton.dev/target-namespace` | pin execution namespace |

**Important:** matching annotations are REQUIRED — without them PaC doesn't match. Multiple matching PipelineRuns run in parallel.

**on-comment gotcha:** don't use built-in GitOps commands (`/test`, `/retest`, `/cancel`, `/ok-to-test`) as on-comment patterns — they're processed before on-comment matching.

## CEL Expressions

```yaml
pipelinesascode.tekton.dev/on-cel-expression: |
  event == "pull_request" && target_branch == "main" && source_branch == "wip"
```

Available vars: `event`, `event_type`, `target_branch`, `source_branch`, `target_url`, `source_url`, `event_title`, `body` (full webhook payload), `headers`, `files.all|added|deleted|modified|renamed`, `.pathChanged` glob suffix (GitHub/GitLab), custom Repository params. CEL string/list functions (`join()`, `split()`, `contains()`, etc.) since v0.47. Test locally: `tkn pac cel`.

## Dynamic Variables (`{{ var }}` in PipelineRun spec)

| Variable | Example output |
|----------|---------------|
| `{{ repo_url }}` | `https://git.example.com/org/repo` |
| `{{ revision }}` | full commit SHA |
| `{{ event }}` / `{{ event_type }}` | `push` / provider-specific |
| `{{ target_branch }}` / `{{ source_branch }}` | `main` / `feature-123` |
| `{{ sender }}` | triggering user |
| `{{ git_tag }}` | tag for tag-push events |
| `{{ pull_request_number }}` | PR/MR number |
| `{{ git_auth_secret }}` | auto-generated secret with provider token (for private-repo checkout) |
| `{{ trigger_comment }}` | full comment text for on-comment runs |
| `{{ body }}` / `{{ headers }}` | raw payload / headers (use block scalar `\|-` for objects!) |
| `{{ pull_request_labels }}` | newline-separated labels |

**YAML gotcha:** objects/multiline values must use block scalars:

```yaml
spec:
  params:
    - name: targets
      value: |-
        {{ trigger_comment }}
```

Inline `value: {{ body }}` → YAML validation error.

## ChatOps / GitOps Commands

| Command | Effect |
|---------|--------|
| `/retest` | Restart only FAILED PipelineRuns for the commit (skips succeeded) |
| `/retest <name>` | Force rerun regardless of status |
| `/test <name>` | Run a specific PipelineRun |
| `/cancel` | Cancel running PipelineRuns |
| `/ok-to-test` | Authorize CI for external contributors (PR only; GitHub may require SHA: `/ok-to-test 1A2B3C4`) |
| `[skip ci]` in commit msg | Skip CI on that push |

Works from PR comments and (for `/test`/`/retest`) commit messages on pushed commits (`branch:<name>` syntax supported). **Build-in journal** — every command lives in PR comment history.

## Security & ACLs

- Unauthorized triggers blocked, `'Pending'` status posted; `/ok-to-test` or `OWNERS` approval required for external contributors
- Policy: author = repo owner/collaborator/org member/OWNERS-listed
- `require-ok-to-test-sha` ConfigMap option closes the GitHub timing-window attack
- GitHub App: short-lived, repo-scoped tokens, auto-refresh

## tkn pac CLI

```bash
brew install --cask openshift-pipelines/pipelines-as-code/tektoncd-pac   # install (tkn plugin)

tkn pac bootstrap          # install PaC + create GitHub App
tkn pac create repo        # create Repository CR interactively
tkn pac webhook add        # add/update provider webhook (Gitea/Forgejo support since v0.49)
tkn pac generate           # scaffold starter PipelineRun in .tekton/
tkn pac resolve            # process a PipelineRun locally as the server would
tkn pac cel                # evaluate CEL against a webhook payload locally
tkn pac list / describe / logs
tkn pac info globbing "[PATTERN]"   # test glob patterns locally
tkn pac delete repo
```

## Reference Deployment Pattern (talos-stack)

Production pattern (GitOps Terraform + worker deploys):

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: cluster-tf-plan
  annotations:
    pipelinesascode.tekton.dev/on-event: "[pull_request, push]"
    pipelinesascode.tekton.dev/on-target-branch: "[main]"
    pipelinesascode.tekton.dev/on-path-change: "[terraform/environments/cluster/**]"
spec:
  serviceAccountName: default
  params:
    - name: repo_url
      value: "{{repo_url}}"
    - name: revision
      value: "{{revision}}"
  workspaces:
    - name: repo
      volumeClaimTemplate:
        spec:
          accessModes: [ReadWriteOnce]
          resources: { requests: { storage: 1Gi } }
          storageClassName: ceph-block
    - name: basic-auth
      secret:
        secretName: "{{ git_auth_secret }}"
  timeouts: { pipeline: 20m }
  pipelineSpec:
    # ... fetch-source task clones at {{ revision }}, then terraform plan/apply
```

**Key patterns:**
- **Plan/apply split** — plan on PR+push (`on-path-change` scoped to env dir), apply ONLY on `/apply` comment (`on-comment` + `on-target-branch: [main]`). Human reviews plan in PR checks; apply never auto-runs on push.
- **In-cluster git rewrite** — PaC mounts provider creds (`pac-gitauth-*`) bound to the EXTERNAL FQDN; pod→gateway→gitea hairpin 503s. Remap to internal service: `sed 's#^https://#http://#; s#@git.example.com#@gitea-http.gitea:3000#'` on the credentials, force with `-c "credential.helper=store --file=$HOME/.git-credentials"` so ambient creds-init can't interfere.
- **Repo-root task fetch** — `pipelinesascode.tekton.dev/task: "./infrastructure/controllers/tekton/tasks/task.yaml"` loads shared Tasks from the repo (no in-cluster Task prerequisite).
- **Shared cluster Tasks** — Task CRs (kaniko-build, wrangler-deploy) live in-cluster, referenced by `taskRef` from PaC runs.
- **`{{ trigger_comment }}` → apply targets** — block scalar so embedded quotes in `-target=` addresses survive.

## Common Mistakes

- **No matching annotations** — PaC ignores the PipelineRun entirely without on-event/on-target-branch.
- **Duplicate PipelineRun names** — never matched. Give each a unique name.
- **Inline object vars** — `value: {{ body }}` fails YAML validation; use block scalar `|-`.
- **Repository CR in PaC namespace** — forbidden. Create it in the namespace where runs execute.
- **`on-cel-expression` + other annotations** — CEL wins, others silently ignored. Pick one style.
- **Hairpin 503 on self-hosted Git** — pod→ingress→gitea loop. Rewrite credentials to the internal service (see pattern above).
- **Built-in commands as on-comment regex** — `/test`/`/retest` are consumed before on-comment matching; use custom commands like `/apply` or `/merge-pr`.
- **Standalone PaC + operator PaC both enabled** — duplicate controllers fighting over webhooks. Pick one.
- **GitHub App install on org** — PaC only triggers when a Repository CR URL matches an installed-org repo.
- **3+ tags pushed at once** — GitHub doesn't send webhooks for >3 simultaneous tag pushes; tag-triggered runs may not fire.
