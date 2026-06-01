---
name: flux
description: Use when working with Flux CD — debugging live clusters, writing/auditing Flux resources, or asking Flux CD questions. Routes to 3 sub-skills depending on the task.
---

# Flux CD

Skill router. Pick the right sub-skill based on what user is doing.

## Which Sub-Skill?

| User wants to... | Load skill | What it does |
|---|---|---|
| Debug a failing HelmRelease/Kustomization on a live cluster | `gitops-cluster-debug` | Inspects Flux resource status, controller logs, traces dependency chains on live K8s |
| Write/understand Flux CRDs, generate YAML, ask what a concept means | `gitops-knowledge` | CRD schemas, YAML patterns, API versions, Flux concepts, FluxInstance/ResourceSet |
| Audit a GitOps repo for best practices, deprecated APIs, security | `gitops-repo-audit` | Scans repo files, validates schemas, flags deprecated APIs, checks RBAC/secrets |

## Two Domains

**Cluster (live):** HelmRelease stuck, Kustomization failing, FluxInstance broken → `gitops-cluster-debug`

**Repo (static):** Write new Flux YAML, understand CRD fields, audit existing repo → `gitops-knowledge` for reference, `gitops-repo-audit` for validation

## Common Task Map

| Task | Sub-skill |
|---|---|
| "Why is my HelmRelease failing?" | `gitops-cluster-debug` |
| "Show me how to write a ResourceSet" | `gitops-knowledge` |
| "Audit my GitOps repo" | `gitops-repo-audit` |
| "What API version for Flux CRDs?" | `gitops-knowledge` |
| "Check if my repo has deprecated APIs" | `gitops-repo-audit` |
| "FluxInstance not ready" | `gitops-cluster-debug` |
| "Explain OCI-based GitOps" | `gitops-knowledge` |
| "Check HelmRelease valuesFrom refs" | `gitops-repo-audit` |
| "Monitor reconciliation, set up alerts" | `gitops-knowledge` |
