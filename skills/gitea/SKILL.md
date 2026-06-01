---
name: gitea
description: Use when working with Gitea — route to the correct sub-skill based on what the user needs: API automation, runner management, container registry, webhooks, or tea CLI.
---

# Gitea (v1.26) — Skill Router

Pick the right sub-skill.

## Which Sub-Skill?

| User wants to... | Load skill |
|---|---|
| Hit REST API endpoints, manage tokens, automate via curl | `gitea-api` |
| Configure gitea-runner, register/deploy runners, manage Actions | `gitea-runner` |
| Push/pull OCI images, manage container registry, configure packages | `gitea-registry` |
| Create/manage webhooks, verify payloads, handle events | `gitea-webhooks` |
| Use `tea` CLI for issues/PRs/repos/releases | `gitea-tea` |

## Quick Map

| Task | Skill |
|---|---|
| "Create a token via API" | `gitea-api` |
| "Deploy a runner on Kubernetes" | `gitea-runner` |
| "Push a Docker image" | `gitea-registry` |
| "Set up a webhook for CI" | `gitea-webhooks` |
| "List my repos with tea" | `gitea-tea` |
| "Delete a package version" | `gitea-api` |
| "Register an ephemeral runner" | `gitea-runner` |
| "Multi-arch build and push" | `gitea-registry` |
| "Verify webhook HMAC signature" | `gitea-webhooks` |
