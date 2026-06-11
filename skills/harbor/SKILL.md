---
name: harbor
description: Use when working with Harbor container registry — route to the correct sub-skill based on what the user needs: API calls, Helm deployment, or Terraform management.
---

# Harbor — Skill Router

Pick the right sub-skill.

## Which Sub-Skill?

| User wants to... | Load skill |
|---|---|
| Hit REST API endpoints, manage projects/artifacts/robots via curl | `harbor-api` |
| Deploy, configure, upgrade Harbor on K8s with Helm | `harbor-helm` |
| Manage Harbor infrastructure as code with Terraform | `harbor-terraform` |

## Quick Map

| Task | Skill |
|---|---|
| "Create a project via API" | `harbor-api` |
| "Set up a robot account for CI" | `harbor-api` |
| "Deploy Harbor on Kubernetes" | `harbor-helm` |
| "Configure external database for Harbor" | `harbor-helm` |
| "Manage Harbor resources with Terraform" | `harbor-terraform` |
| "Create a replication rule" | `harbor-api` |
| "Configure Trivy scanner" | `harbor-api` |
| "Upgrade Harbor Helm release" | `harbor-helm` |
| "Provision projects + robot accounts as code" | `harbor-terraform` |
| "Manage retention policies" | `harbor-api` |
| "Configure OIDC auth" | `harbor-api` |
