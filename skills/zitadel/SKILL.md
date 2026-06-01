---
name: zitadel
description: Use when working with ZITADEL identity platform — API operations (orgs, apps, users, roles) or Helm deployment (install, configure, upgrade on K8s). Triggers: auth.kubexa.tech, zitadel OIDC app creation, SAML/API app, zitadel deployment.
---

# ZITADEL

Skill router. Pick sub-skill based on task.

## Which Sub-Skill?

| Task | Load skill | What it does |
|---|---|---|
| Create/manage OIDC/SAML/API apps, users, orgs, roles via API | `zitadel-api` (user-installed) | Auth, org/project/app CRUD, role grants, DB queries, Connect RPC + REST endpoints |
| Install, configure, upgrade ZITADEL on K8s | `zitadel-helm` | HelmRelease, values, CNPG, Gateway API, caches, masterkey, SMTP, upgrades |

## Decision Flow

- Using curl/API calls to `auth.kubexa.tech`? → `zitadel-api`
- Editing `release.yaml`, values, CNPG cluster, Gateway routes? → `zitadel-helm`
- Need SAML or API app config? → `zitadel-api` (covers all 3 types with examples)
- FirstInstance bootstrapping, masterkey rotation, SMTP setup? → `zitadel-helm`
