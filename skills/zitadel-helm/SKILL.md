---
name: zitadel-helm
description: Use when deploying, configuring, or upgrading ZITADEL on Kubernetes via Helm. Covers HelmRelease values, CNPG database, Gateway API routing, caches, masterkey, SMTP, and production patterns. NOT for API-level operations (orgs/apps/users).
---

# ZITADEL Helm

## Overview

ZITADEL deployed via HelmRelease (Flux) with external PostgreSQL (CNPG), Gateway API routing (Cilium Gateway), memory caching. Two containers: main ZITADEL API (Go) and Login UI (Next.js). Init + setup jobs bootstrap the FirstInstance on install.

**Deployed version:** chart 9.34.1, see `apps/base/zitadel/release.yaml`.

## Architecture

```
Cilium Gateway (https)
  ├── HTTPRoute "zitadel"         → zitadel:8080 (h2c)
  ├── HTTPRoute "zitadel-login"   → zitadel-login:3000  (/ui/v2/login)
  └── GRPCRoute (optional)        → zitadel:8080 (gRPC)

ZITADEL Deployment (zitadel:8080)
  ├── Init job (schema bootstrap)
  ├── Setup job (FirstInstance creation)
  └── Env vars → DB credentials from K8s secrets

ZITADEL Login Deployment (login:3000)
  └── Communicates with main ZITADEL API via PAT

CNPG Cluster (zitadel-db, PostgreSQL 18)
  └── zitadel-db-rw:5432 → ZITADEL connects via DSN
```

## Key Configuration

### Database (CNPG — External)

Disabled bundled PostgreSQL subchart. CNPG cluster created separately:

```yaml
# HelmRelease values
postgresql:
  enabled: false

zitadel:
  configmapConfig:
    Database:
      Postgres:
        Host: zitadel-db-rw.zitadel.svc.cluster.local
        Port: 5432
        Database: zitadel
        User:
          Username: zitadel
          SSL:
            Mode: disable   # or require/verify-full with dbSslCaCrt
        Admin:
          Username: zitadel
          SSL:
            Mode: disable
  env:
    - name: ZITADEL_DATABASE_POSTGRES_USER_PASSWORD
      valueFrom:
        secretKeyRef:
          name: zitadel-db-app
          key: password
    - name: ZITADEL_DATABASE_POSTGRES_ADMIN_PASSWORD
      valueFrom:
        secretKeyRef:
          name: zitadel-db-app
          key: password
```

Alternative DSN-based config (replaces `configmapConfig.Database.Postgres`):

```yaml
zitadel:
  env:
    - name: ZITADEL_DATABASE_POSTGRES_DSN
      valueFrom:
        secretKeyRef:
          name: zitadel-db-credentials
          key: dsn
```

### Secrets Management

Sensitive values via Flux `valuesFrom` (not plaintext in release.yaml):

```yaml
# release.yaml
valuesFrom:
  - kind: Secret
    name: zitadel-shared
    valuesKey: masterkey
    targetPath: zitadel.masterkey
  - kind: Secret
    name: zitadel-db-app
    valuesKey: password
    targetPath: zitadel.configmapConfig.Database.Postgres.User.Password
  - kind: Secret
    name: zitadel-shared
    valuesKey: sharedSecret
    targetPath: global.sharedSecret
```

**Masterkey:** Must be exactly 32 bytes printable ASCII. Generate:
```bash
tr -dc A-Za-z0-9 </dev/urandom | head -c 32
```
Set via `zitadel.masterkey` or `zitadel.masterkeySecretName`. Loss = data loss.

### Gateway API Routing

```yaml
ingress:
  enabled: false   # Use Gateway API instead

gateway:
  grpcRoute:
    enabled: true
    parentRefs:
      - kind: Gateway
        name: cilium-gateway
        namespace: cilium-gateway
        sectionName: https
```

Separate HTTPRoute resources for main API and login:

| Route | Host | Backend | Purpose |
|---|---|---|---|
| `zitadel` (HTTPRoute) | auth.kubexa.tech | zitadel:8080 | Main API, console |
| `zitadel-login` (HTTPRoute) | auth.kubexa.tech | zitadel-login:3000 | Login UI at /ui/v2/login |
| `zitadel-grpc` (GRPCRoute, auto) | auth.kubexa.tech | zitadel:8080 | gRPC (if gateway.grpcRoute.enabled) |

Both HTTPRoutes match the same hostname with path-based splitting.

**Important:** ZITADEL requires end-to-end HTTP/2 (h2c). Cilium Gateway handles this automatically when `sectionName: https` targets the HTTPS listener.

### Caching

Memory cache for small deployments:

```yaml
zitadel:
  configmapConfig:
    Caches:
      Connectors:
        Memory:
          Enabled: true
      Instance:
        Connector: memory
        MaxAge: 1h
      Organization:
        Connector: memory
        MaxAge: 1h
```

Production (Redis/Valkey):

```yaml
Caches:
  Connectors:
    Redis:
      Enabled: true
      Addr: redis-cluster:6379
  Instance:
    Connector: redis
    MaxAge: 10m
```

### FirstInstance / Bootstrapping

Configured in `configmapConfig.FirstInstance`. Helm chart auto-generates Machine Keys + PAT secrets:

```yaml
zitadel:
  configmapConfig:
    FirstInstance:
      Org:
        Human:
          UserName: admin
          Password: "Admin123!"
          FirstName: Admin
          LastName: User
          Email: admin@kubexa.tech
          PasswordChangeRequired: false
        Machine:
          Machine:
            Username: iam-admin
          Pat:
            ExpirationDate: "2029-01-01T00:00:00Z"
```

After install, secrets created:
- `iam-admin` — JWT machine key
- `iam-admin-pat` — Personal Access Token
- `login-client` — Login UI's PAT

### Node Selection

```yaml
nodeSelector:
  kubernetes.io/hostname: worker-proxmox

login:
  nodeSelector:
    kubernetes.io/hostname: worker-proxmox
```

### SMTP / Notifications

```yaml
zitadel:
  configmapConfig:
    Notifications:
      SMTP:
        Host: smtp-relay.brevo.com
        Port: 587
        User: admin@kubexa.tech
        Password: ""     # via valuesFrom secret
        SenderAddress: noreply@kubexa.tech
        SenderName: Zitadel Auth
```

## Upgrade

1. Update `version:` in HelmRelease (chart 9.x)
2. ZITADEL handles its own DB migrations via init + setup jobs
3. Helm hooks orchestrate order: init → setup → deployment
4. No manual migration steps needed for patch/minor versions
5. **Check release notes** for breaking changes between major chart versions (v8→v9 dropped CockroachDB templates, v9 defaults to ZITADEL v4)
6. ZITADEL supports PG 14–18. PG 18 requires ZITADEL v4.11.0+
7. Masterkey never changes after initial install
8. After upgrade, verify console login, OIDC app auth, and DB migration logs

## Quick Reference

| Need | Config path |
|---|---|
| External domain | `zitadel.configmapConfig.ExternalDomain` |
| TLS termination | At Gateway, set `TLS.Enabled: false` |
| Database host | `zitadel.configmapConfig.Database.Postgres.Host` |
| DB SSL mode | `Database.Postgres.User.SSL.Mode` (disable/require/verify-full) |
| Masterkey | `zitadel.masterkey` or `zitadel.masterkeySecretName` |
| Admin bootstrap | `zitadel.configmapConfig.FirstInstance.Org.Human` |
| SMTP | `zitadel.configmapConfig.Notifications.SMTP.*` |
| Memory cache | `zitadel.configmapConfig.Caches.Connectors.Memory.Enabled: true` |
| Node pinning | `nodeSelector`, `login.nodeSelector` |
| Disable bundled PG | `postgresql.enabled: false` |
| Disable Ingress | `ingress.enabled: false` (use Gateway API) |
| Enable gRPC route | `gateway.grpcRoute.enabled: true` |

## Common Mistakes

- **Masterkey loss = permanent data loss.** Cannot recover. Store in SealedSecret or external vault.
- **Bundled PostgreSQL subchart NOT for production.** Always disable (`postgresql.enabled: false`) and use external CNPG cluster.
- **Gateway must support h2c backend.** Cilium Gateway does. Without h2c, ZITADEL console fails.
- **Login route must precede main route** on the same hostname, otherwise /ui/v2/login matches the catch-all `/` rule and hits the main API instead of the login UI.
- **env vars take precedence over configmapConfig.** If both `ZITADEL_DATABASE_POSTGRES_DSN` env var and `configmapConfig.Database.Postgres` are set, the env var wins.
- **PasswordChangeRequired: true** on FirstInstance.Human will block initial console login until admin changes password.
- **SMTP password** must be via secret, never in values.yaml. Use `valuesFrom` in HelmRelease.
- **FirstInstance fields are install-only.** Changing them post-install has no effect. Modify via API or DB directly.
