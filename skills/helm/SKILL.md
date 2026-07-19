---
name: helm
description: Use when working with Helm — route to the correct sub-skill based on what you need: writing Helm charts or production operations (GitOps, release inspection, OCI, hooks). **Manual helm CLI for prod deploys is NOT recommended — prefer GitOps (Flux, ArgoCD).**
---

# Helm — Skill Router

**Latest:** v4.2.3 (v3.21.3 support until Nov 2026)

## ⚠️ Production Warning

**Do not use `helm install` / `helm upgrade` manually or from CI for production.** Manual Helm bypasses drift detection, audit trail, Git-as-truth, and rollback-via-revert.

| Use GitOps instead |
|---|
| Flux: `HelmRelease` (helm.toolkit.fluxcd.io/v2) |
| ArgoCD: `Application` with Helm source |

Helm CLI is for **development, chart writing, debugging, and emergency recovery** — not day-2 operations.

## Which Sub-Skill?

**Match the user's request against the trigger keywords below, then call `skill(name="<matched-skill>")` to load the sub-skill's full content.**

### Routing Table

| Trigger keywords | User wants to... | Skill to load |
|---|---|---|
| chart, template, HelmRelease values, _helpers.tpl, values.yaml, schema, write, create, modify, library chart, chart testing, helm unittest | Write or modify a Helm chart | `helm-chart` |
| install, upgrade, rollback, inspect, release, status, get values, get manifest, history, failed, troubleshoot, debug, OCI, push, pull, hook, GitOps, Flux, ArgoCD, HelmRelease, drift | Deploy, inspect, or troubleshoot releases | `helm-ops` |

### Decision Tree

If routing is ambiguous:

1. **User is writing code** (templates, YAML, values, helpers) → `helm-chart`
2. **User is running commands** (install, inspect, debug, push) → `helm-ops`
3. **User is asking about both?** → Load `helm-chart` first (chart must exist before deploying), then `helm-ops`

### When NOT to load this router

- User question is about a **specific product's Helm chart** (e.g. Higress, Vault, MariaDB) → load that product's `*-helm` skill instead (e.g. `higress-helm`, `vault-helm`)
- User needs **Terraform** for Helm releases → load `harbor-terraform` or similar
- User is working with **ArgoCD or Flux generally** (not Helm-specific) → load their respective skills
- User just needs a **quick CLI reference** → router's Quick CLI Reference section below is sufficient

## Architecture

```
Chart → template engine (Go + Sprig) → rendered manifests → kubectl apply (SSA in v4) → release stored in Secrets
```

No server-side component (Helm 3+ client-only). Releases live as Secrets in the target namespace.

### Helm 4 vs 3

| Feature | Helm 3 | Helm 4 |
|---------|--------|--------|
| Apply mode | Client-side patch | Server-side apply (SSA) default |
| `--atomic` | ✓ | Renamed to `--rollback-on-failure` |
| `--force` | Delete+recreate pods | Renamed to `--force-replace` |
| Post-renderer | Arbitrary binary | Plugin name required |
| Status | Bug fixes until Jul 2026, security until Nov 2026 | Current stable |

## Quick CLI Reference

| Command | Purpose |
|---------|---------|
| `helm install <rel> <chart> -n <ns>` | Install (dev/test only) |
| `helm upgrade --install <rel> <chart> -n <ns> -f vals.yaml` | Idempotent install-or-upgrade |
| `helm rollback <rel> [rev] -n <ns>` | Rollback |
| `helm uninstall <rel> -n <ns>` | Delete |
| `helm list -n <ns> [-a]` | List releases |
| `helm history <rel> -n <ns>` | Revision history |
| `helm status <rel> -n <ns>` | Release status |
| `helm template <rel> ./c -f vals.yaml` | Render locally (no cluster) |
| `helm lint ./c -f vals.yaml` | Validate chart structure |
| `helm get values\|manifest\|notes\|all <rel> -n <ns>` | Inspect release |
| `helm show values\|chart\|readme\|all <chart>` | Inspect chart |
| `helm push .tgz oci://r.example.com/repo` | Push to OCI |
| `helm dep update\|build\|list` | Manage dependencies |
| `helm test <rel> -n <ns>` | Run test pods |

### Values Override

| Flag | Example |
|------|---------|
| `-f, --values` | `-f prod.yaml -f override.yaml` (later wins) |
| `--set` | `--set image.tag=v2.0` |
| `--set-string` | `--set-string image.tag=v2.0.0-rc1` |

**Prefer `-f` over `--set`.** Audit-friendly, composable for environments.
