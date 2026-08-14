---
name: tekton
description: Use when working with Tekton — route to the correct sub-skill based on what you need: PipelineRun/TaskRun authoring, the Tekton operator (install/config), or Pipelines-as-Code (Git-native CI).
---

# Tekton — Skill Router

**Operator:** tektoncd/operator — latest v0.80.0 (2026-06-09); LTS v0.78.x (pipeline v1.6.x, EOL 2026-12-08), v0.77.x (EOL 2026-08-21). Min K8s 1.28.
**Pipelines-as-Code:** tektoncd/pipelines-as-code — latest v0.49.0 (2026-07-06).
**API version:** `tekton.dev/v1` (authoring), `operator.tekton.dev/v1alpha1` (operator CRs), `pipelinesascode.tekton.dev/v1alpha1` (PaC Repository CR).

## Overview

Tekton is Kubernetes-native CI/CD. Three surfaces:

- **Authoring** — writing PipelineRun/TaskRun YAML (resolvers, when/CEL, matrix, podTemplate, timeouts) in `tekton.dev/v1`
- **Operator** — installs/manages Tekton components (Pipelines, Triggers, Dashboard, Chains, Results, PaC) via `TektonConfig` + component CRs
- **Pipelines-as-Code (PaC)** — Git-native CI: PipelineRuns live in `.tekton/` in the repo, triggered by Git events (push/PR/comment), results reported back as PR checks

## Which Sub-Skill?

**Match the user's request against the trigger keywords below, then call `skill(name="<matched-skill>")` to load the sub-skill's full content.**

### Routing Table

| Trigger keywords | User wants to... | Skill to load |
|---|---|---|
| pipelinerun, taskrun, resolver, git-resolver, taskRef, when, CEL, matrix, podTemplate, nodeSelector, tolerations, timeout, finally, workspaces, tekton.dev/v1 | Author/write Tekton pipeline YAML | `tekton-pipelines` |
| operator, TektonConfig, TektonPipeline, TektonTrigger, TektonDashboard, TektonChain, TektonResult, TektonPruner, profile, install tekton, upgrade tekton, component CR | Install/configure/manage Tekton components on a cluster | `tekton-operator` |
| pipelines-as-code, pipelinesascode, PaC, .tekton/, Repository CR, on-event, on-target-branch, on-path-change, on-comment, on-cel-expression, ChatOps, /retest, /test, tkn pac, git_auth_secret, trigger_comment | Git-native CI: pipelines in repo, triggered by Git events | `tekton-pac` |

### Decision Tree

If routing is ambiguous:

1. **User is writing YAML for a pipeline** (taskRef, when, matrix, podTemplate) → `tekton-pipelines`
2. **User is installing/configuring Tekton itself** (TektonConfig CR, profiles, components) → `tekton-operator`
3. **User is setting up Git-triggered CI** (Repository CR, .tekton annotations, webhooks, ChatOps) → `tekton-pac`
4. **Combination?** → Load the one matching the primary action. Common pairs: operator installs Pipelines → PaC connects repos → pipelines authoring writes the runs. Authoring (`.tekton/` files) + PaC (annotations on those files) are especially tight — load both for PaC work.

### When NOT to load this router

- User is working on a **different CI system** (GitHub Actions, Gitea Actions, GitLab CI) → those are their own skills/tools
- User needs **Kubernetes cluster automation** (Flux, ArgoCD) → Tekton runs workloads; GitOps is a separate concern (`flux` skill)
- User asks about **generic GitOps pipelines** without Tekton → check if they mean Flux/ArgoCD first
- User just needs a **quick version/overview** → the Overview above is sufficient
