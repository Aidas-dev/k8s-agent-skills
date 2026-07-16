---
name: nextcloud-helm
description: Use when deploying, configuring, or upgrading Nextcloud on Kubernetes via Helm. Covers HelmRelease values, database (external PostgreSQL/MySQL or bundled MariaDB), Gateway API/Ingress routing, Redis cache, S3/Swift primary storage, Collabora Online, Imaginary previews, cron jobs, and metrics. NOT for API-level user/group operations.
---

# Nextcloud Helm

## Overview

Nextcloud deployed via HelmRelease (Flux) with optional external PostgreSQL, Redis caching, S3 primary storage. Chart uses Docker images from `docker.io/library/nextcloud` with `apache` or `fpm` flavor. Built-in HTTPRoute (Gateway API v1) support, Ingress, nginx sidecar when using fpm image.

**Latest stable:** chart 9.2.x → Nextcloud v34. See [artifacthub](https://artifacthub.io/packages/helm/nextcloud/nextcloud) for latest version.

**Dependencies:**
- PostgreSQL 16.7.4 (bitnami, OCI)
- MariaDB 20.5.5 (bitnami, OCI)
- Redis 21.1.3 (bitnami, OCI)
- Collabora Online 1.1.60

## Architecture

```
Cilium Gateway / Ingress
  └── HTTPRoute / Ingress → nextcloud:8080

Nextcloud Deployment (apache image)
  ├── Sidecar: crond (optional, sidecar mode)
  ├── InitContainer: DB wait (PostgreSQL or MariaDB)
  └── Volume: /var/www/html (PVC)

External PostgreSQL (recommended)
  └── nextcloud-db-rw:5432

External Redis (caching)
  └── redis:6379

S3 Object Store (primary storage, optional)
  └── S3 bucket via env vars

Imaginary (image previews, optional)
  └── nextcloud-imaginary:9000
```

## Key Configuration

### Image

```yaml
image:
  registry: docker.io
  repository: library/nextcloud
  flavor: apache        # "apache" or "fpm"
  tag: ""                # generated from appVersion
  pullPolicy: IfNotPresent
```

Two flavors: `apache` (all-in-one, simpler) or `fpm` (needs nginx sidecar, more configurable).

### Gateway API (HTTPRoute)

Built-in HTTPRoute support — no Ingress needed:

```yaml
httpRoute:
  enabled: true
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  hostnames:
    - "nextcloud.example.com"
  parentRefs:
    - name: cilium-gateway
      namespace: cilium-gateway
      sectionName: https
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: "/"
  # Automatically adds .well-known/carddav and .well-known/caldav redirects
  wellKnown:
    enabled: true
```

The chart's `wellKnown.enabled: true` (default) automatically adds Gateway API `RequestRedirect` rules for `/.well-known/carddav` and `/.well-known/caldav` → `/remote.php/dav`.

### Database

**External PostgreSQL (recommended for production):**

```yaml
internalDatabase:
  enabled: false

externalDatabase:
  enabled: true
  type: postgresql
  host: nextcloud-db-rw.nextcloud.svc.cluster.local:5432
  user: nextcloud
  password: ""  # via valuesFrom secret
  database: nextcloud
  existingSecret:
    enabled: true
    secretName: nextcloud-db-credentials
    usernameKey: db-username
    passwordKey: db-password
    hostKey: db-host
    databaseKey: db-name
```

**Bundled MariaDB (for testing/small deployments):**

```yaml
internalDatabase:
  enabled: false

mariadb:
  enabled: true
  auth:
    database: nextcloud
    username: nextcloud
    password: changeme
  primary:
    persistence:
      enabled: true
      size: 8Gi
```

### Redis Caching

**External Redis (recommended):**

```yaml
externalRedis:
  enabled: true
  host: redis-master.redis.svc.cluster.local
  port: "6379"
  password: ""  # via valuesFrom secret
  existingSecret:
    enabled: false
```

**Bundled Redis:**

```yaml
redis:
  enabled: true
  auth:
    password: changeme
  master:
    persistence:
      enabled: true
```

### SMTP / Mail

```yaml
nextcloud:
  mail:
    enabled: true
    fromAddress: noreply
    domain: example.com
    smtp:
      host: smtp-relay.brevo.com
      secure: ssl
      port: 465
      authtype: LOGIN
      name: noreply@example.com
      password: ""  # via valuesFrom secret
```

### S3 Primary Object Store

```yaml
nextcloud:
  objectStore:
    s3:
      enabled: true
      host: s3.eu-central-1.amazonaws.com
      ssl: true
      port: "443"
      region: eu-central-1
      bucket: nextcloud-data
      accessKey: ""  # ignored if existingSecret set
      secretKey: ""
      usePathStyle: false
      autoCreate: true
      existingSecret: nextcloud-s3-credentials
```

### Imaginary (Image Previews)

High-performance image preview server. Dramatically faster than PHP-based previews:

```yaml
imaginary:
  enabled: true
  image:
    registry: docker.io
    repository: h2non/imaginary
    tag: 1.2.4
  resources:
    requests:
      memory: 256Mi
      cpu: 100m
```

Also needs config to enable previews:

```yaml
nextcloud:
  configs:
    previews.config.php: |
      <?php
      $CONFIG = array (
        'enable_previews' => true,
        'enabledPreviewProviders' => array (
          'OC\Preview\Imaginary',
          'OC\Preview\Movie',
          'OC\Preview\PNG',
          'OC\Preview\JPEG',
          'OC\Preview\GIF',
          'OC\Preview\BMP',
          'OC\Preview\MP3',
          'OC\Preview\TXT',
          'OC\Preview\MarkDown',
          'OC\Preview\PDF',
        ),
      );
```

And Imaginary support config:

```yaml
nextcloud:
  defaultConfigs:
    imaginary.config.php: true
```

### Cron Jobs

Two modes — `sidecar` (crond alongside Nextcloud, requires root) or `cronjob` (Kubernetes CronJob, can run non-root):

```yaml
cronjob:
  enabled: true
  type: cronjob  # or "sidecar"
  cronjob:
    schedule: "*/5 * * * *"
    command:
      - php
      - -f
      - /var/www/html/cron.php
      - --
      - --verbose
    affinity:
      podAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
                - key: app.kubernetes.io/name
                  operator: In
                  values:
                    - nextcloud
                - key: app.kubernetes.io/component
                  operator: In
                  values:
                    - app
            topologyKey: kubernetes.io/hostname
```

CronJob mode requires `persistence.enabled: true` (same PVC).

### Metrics / Monitoring

**Nextcloud built-in OpenMetrics** (exposes at `/index.php/apps/helm-metrics/metrics`):

```yaml
nextcloud:
  openmetrics:
    allowedClients:
      - "127.0.0.1"
      - "10.42.0.0/16"
```

**nextcloud-exporter sidecar service**:

```yaml
metrics:
  enabled: true
  https: false
  info:
    apps: true
    update: true
  serviceMonitor:
    enabled: true
    interval: 30s
```

### Persistence

```yaml
persistence:
  enabled: true
  storageClass: ceph-block
  accessMode: ReadWriteOnce
  size: 50Gi
  # Optional: separate data PVC on different storageClass
  nextcloudData:
    enabled: false
    storageClass: ""
    size: 50Gi
```

## Upgrade

1. Update `version:` in HelmRelease to new chart version
2. Nextcloud handles its own DB migrations via `occ upgrade` on startup
3. If using `oc_appconfig` entries for apps, run `occ app:update --all` after upgrade
4. Default strategy is `Recreate` — expect brief downtime
5. PHP config changes may require manual occ commands after upgrade
6. For major Nextcloud version bumps, check [UPGRADE.md](https://github.com/nextcloud/server/blob/master/UPGRADE.md)

### Post-Upgrade Verification

```bash
# Inside the pod
occ status
occ db:add-missing-indices
occ db:convert-filecache-bigint
occ maintenance:repair
```

## Quick Reference

| Need | Config path |
|---|---|
| Nextcloud admin user | `nextcloud.username`, `nextcloud.password` or `nextcloud.existingSecret` |
| External hostname | `nextcloud.host` |
| Trusted domains | `nextcloud.trustedDomains` |
| External DB host | `externalDatabase.host` |
| External DB credentials | `externalDatabase.password` or `externalDatabase.existingSecret` |
| S3 bucket | `nextcloud.objectStore.s3` |
| Redis host | `externalRedis.host` or `redis.*` |
| SMTP | `nextcloud.mail.*` |
| Image previews | `imaginary.enabled: true` + configs |
| Gateway API | `httpRoute.*` |
| Cron type | `cronjob.type`: `sidecar` or `cronjob` |
| ServiceMonitor | `metrics.serviceMonitor.enabled: true` |
| Nginx sidecar | `nginx.enabled: true` (requires fpm image flavor) |
| PHP config overrides | `nextcloud.phpConfigs` |
| Custom Nextcloud config | `nextcloud.configs` |
| Upload size limit | nginx `client_max_body_size` in `nginx.config.serverBlockCustom` |

## OIDC / OpenID Connect Integration

Chart has **no native OIDC section** in values. OIDC is handled by the `user_oidc` app (official, [apps.nextcloud.com/apps/user_oidc](https://apps.nextcloud.com/apps/user_oidc), v8.10.1 for NC34).

### Post-install setup

```bash
# Install the app (usually bundled, enable via occ)
kubectl exec deploy/nextcloud -- php occ app:enable user_oidc

# Register OIDC provider (ZITADEL, Keycloak, etc.)
kubectl exec deploy/nextcloud -- php occ user_oidc:provider zitadel \
  --clientid="<client-id>" \
  --clientsecret="<client-secret>" \
  --discoveryuri="https://auth.example.com/.well-known/openid-configuration"
```

### Auto-provisioning (default: on)

New users created on first OIDC login. Disable in `config.php`:

```yaml
nextcloud:
  configs:
    20-oidc.config.php: |
      <?php
      $CONFIG = array (
        'user_oidc' => array (
          'auto_provision' => false,          # Don't auto-create users
          'soft_auto_provision' => true,       # Accept existing users from other backends
          'disable_account_creation' => false,  # Don't create accounts for new users
        ),
      );
```

### Via lifecycle hooks

Automate provider registration on deploy:

```yaml
nextcloud:
  hooks:
    post-installation:
      - |
        php occ app:enable user_oidc
        php occ user_oidc:provider zitadel \
          --clientid="${OIDC_CLIENT_ID}" \
          --clientsecret="${OIDC_CLIENT_SECRET}" \
          --discoveryuri="https://auth.example.com/.well-known/openid-configuration"
  extraEnv:
    - name: OIDC_CLIENT_ID
      valueFrom:
        secretKeyRef:
          name: nextcloud-oidc
          key: client-id
    - name: OIDC_CLIENT_SECRET
      valueFrom:
        secretKeyRef:
          name: nextcloud-oidc
          key: client-secret
```

### ZITADEL OIDC app config

Create OIDC app in ZITADEL (Authorization Code + client_secret_basic):

```bash
# Redirect URI
https://nextcloud.example.com/index.php/apps/user_oidc/code?flow=1

# Post-logout redirect
https://nextcloud.example.com/
```

### Bearer token validation

For API access via OIDC tokens:

```yaml
nextcloud:
  configs:
    21-oidc-bearer.config.php: |
      <?php
      $CONFIG = array (
        'user_oidc' => array (
          'userinfo_bearer_validation' => true,
        ),
      );
```

### Key occ commands

```bash
# List providers
occ user_oidc:provider

# Show provider details
occ user_oidc:provider zitadel

# Delete provider
occ user_oidc:provider:delete zitadel

# Disable default claims (quota, name, email, groups)
occ config:app:set --value=0 user_oidc enable_default_claims

# Single logout (default: on)
# config.php: 'user_oidc' => ['single_logout' => false]

# PKCE (default: on)
# config.php: 'user_oidc' => ['use_pkce' => false]
```

## Common Mistakes

- **Flavor mismatch** — `flavor: apache` bundles Apache + PHP-FPM in one image. `flavor: fpm` needs `nginx.enabled: true` sidecar. Mixing them gets blank pages.
- **Recreate strategy** — Default `strategy.type: Recreate` means brief downtime on upgrades. Set to `RollingUpdate` if zero-downtime is needed (requires careful readiness probe tuning).
- **Redis password** — Bundled Redis uses `auth.password`. External Redis uses `externalRedis.password`. Mismatch causes Nextcloud to silently degrade to file-based caching.
- **S3 endpoint trailing slash** — Do NOT include trailing slash in S3 `host` field. Nextcloud appends path.
- **Database upgrade timeout** — Large Nextcloud DB migrations can exceed the default liveness probe timeout. Increase `livenessProbe.initialDelaySeconds` or `timeoutSeconds` during upgrades.
- **Config files order** — `nextcloud.configs` files are loaded alphabetically. Prefix with numbers if ordering matters (e.g., `10-previews.config.php`).
- **CronJob needs same PVC** — When using `cronjob.type: cronjob`, the CronJob pod must access the same PVC. Use pod affinity to ensure same node scheduling.
- **HTTPRoute wellKnown redirects** — `httpRoute.wellKnown.enabled: true` adds `RequestRedirect` rules for `.well-known/carddav` and `.well-known/caldav`. Disable if you handle these differently.
- **OpenMetrics IP allowlist** — `nextcloud.openmetrics.allowedClients` must include Prometheus pod CIDR. Default includes common K8s CIDRs (10.42.0.0/16, 10.43.0.0/16).
