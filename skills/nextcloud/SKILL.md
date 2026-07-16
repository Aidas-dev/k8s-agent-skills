---
name: nextcloud
description: Use when working with Nextcloud — deploying on Kubernetes via Helm (nextcloud-helm) or managing users/groups/apps via the Provisioning API (nextcloud-api). Triggers: nextcloud deploy, nextcloud OCC, NC user provisioning, nextcloud-helm.
---

# Nextcloud

Skill router. Pick sub-skill based on task.

## Which Sub-Skill?

| Task | Load skill | What it does |
|---|---|---|
| Deploy, configure, upgrade Nextcloud on K8s with Helm chart | `nextcloud-helm` | HelmRelease values, DB (external/internal), Gateway API/Ingress, Redis, S3, Collabora, cron, metrics, Imaginary previews |
| Manage users, groups, apps via Provisioning API | `nextcloud-api` | OCS API endpoints, auth (Basic + OCS-APIRequest), curl examples for users/groups/apps CRUD |

## Decision Flow

- Installing or upgrading Nextcloud on Kubernetes? → `nextcloud-helm`
- Need to create/edit/delete users or groups programmatically? → `nextcloud-api`
- Configuring external DB (PostgreSQL/MySQL), S3 object storage, Redis cache? → `nextcloud-helm`
- Setting up SMTP, Collabora, Imaginary previews, cron jobs? → `nextcloud-helm`
- Adding a user from a script or CI pipeline? → `nextcloud-api`
- Need to enable/disable an app or list installed apps via API? → `nextcloud-api`
- Bumping chart version or modifying values.yaml? → `nextcloud-helm`
- Setting up Gateway API (HTTPRoute) for Nextcloud? → `nextcloud-helm`
