---
name: harbor-helm
description: Use when deploying, configuring, or upgrading Harbor on Kubernetes via Helm chart — values configuration, external database, TLS/certificates, storage, ingress, authentication, and production patterns.
---

# Harbor Helm Chart

Source: `https://helm.goharbor.io`. Latest chart version: **1.19.1** (app version 2.15.1). Images: all v2.15.1 (core, portal, jobservice, registry, trivy, nginx, log, database, redis, exporter).

## Quick Install

```bash
helm repo add harbor https://helm.goharbor.io
helm repo update

helm install harbor harbor/harbor \
  --namespace harbor \
  --create-namespace \
  --set expose.tls.auto.commonName=harbor.example.com \
  --set externalURL=https://harbor.example.com \
  --set harborAdminPassword=admin123
```

## Values Overview

### Expose / Ingress

| Parameter | Default | Description |
|-----------|---------|-------------|
| `expose.type` | `ingress` | `ingress`, `clusterIP`, `nodePort`, `loadBalancer` |
| `expose.tls.auto.commonName` | — | Auto-generate cert for this hostname |
| `expose.tls.secretName` | — | Use existing TLS secret |
| `expose.tls.certSource` | `auto` | `auto`, `secret`, `none` |
| `expose.ingress.hosts.core` | `core.harbor.domain` | Core ingress host |
| `expose.ingress.hosts.notary` | `notary.harbor.domain` | Notary ingress host |
| `expose.ingress.className` | — | Ingress class name |
| `expose.ingress.annotations` | `{}` | Ingress annotations |

### External URL

| Parameter | Default | Description |
|-----------|---------|-------------|
| `externalURL` | `https://core.harbor.domain` | Full URL users access Harbor at |

### Auth

| Parameter | Default | Description |
|-----------|---------|-------------|
| `harborAdminPassword` | `Harbor12345` | Initial admin password |
| `database.internal` | `true` | Use internal PostgreSQL |
| `database.type` | `postgresql` | `postgresql` or `external` |

### External Database

```yaml
database:
  type: external
  external:
    host: postgres.example.com
    port: 5432
    username: harbor
    password: secret
    database: harbor
    sslmode: require
    maxIdleConns: 100
    maxOpenConns: 900
```

### External Redis

```yaml
redis:
  type: external
  external:
    addr: redis.example.com:6379
    password: secret
    sentinelMaster: mymaster  # if using sentinel
```

### Storage

| Component | Default PVC | Parameter |
|-----------|-------------|-----------|
| Registry | `200Gi` | `persistence.persistentVolumeClaim.registry.size` |
| Jobservice | `1Gi` | `persistence.persistentVolumeClaim.jobservice.size` |
| Database | `1Gi` | `persistence.persistentVolumeClaim.database.size` |
| Redis | `1Gi` | `persistence.persistentVolumeClaim.redis.size` |
| Trivy | `5Gi` | `persistence.persistentVolumeClaim.trivy.size` |

Object storage (S3-compatible) for registry:

```yaml
persistence:
  imageChartStorage:
    type: s3
    s3:
      region: us-east-1
      bucket: harbor-registry
      accesskey: AKIA...
      secretkey: ...
      rootdirectory: /registry
```

### Trivy Scanner

| Parameter | Default | Description |
|-----------|---------|-------------|
| `trivy.enabled` | `true` | Enable Trivy vulnerability scanner |
| `trivy.image.repository` | `goharbor/trivy-adapter` | Scanner image |
| `trivy.image.tag` | `v0.35.1` | Adapter version (Harbor 2.15.x) |
| `trivy.gitHubToken` | — | GitHub token for Trivy DB download (avoid rate limits) |
| `trivy.skipUpdate` | `false` | Skip Trivy DB update on startup |
| `trivy.offlineScan` | `false` | Disable vulnerability DB updates |

### Components

| Parameter | Description |
|-----------|-------------|
| `portal.enabled` | Enable Harbor web UI (core depends on it) |
| `core.replicas` | Core API replicas |
| `jobservice.replicas` | Job service replicas |
| `registry.replicas` | Registry replicas |
| `exporter.enabled` | Enable Prometheus metrics exporter |
| `chartmuseum.enabled` | Enable Helm Chart Museum |
| `notary.enabled` | Enable Notary (deprecated, disabled by default) |
| `notary.disabled` | Notary v1 removed in v2.9+ |

## Production Values Example

```yaml
expose:
  type: ingress
  tls:
    certSource: secret
    secretName: harbor-tls
  ingress:
    className: cilium
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
    hosts:
      core: harbor.example.com
      notary: notary.example.com

externalURL: https://harbor.example.com

harborAdminPassword: changeme

database:
  type: external
  external:
    host: postgres-cluster-rw.db.svc
    port: 5432
    username: harbor
    password: "${DB_PASSWORD}"
    database: harbor
    sslmode: require
    maxIdleConns: 50
    maxOpenConns: 500

redis:
  type: external
  external:
    addr: redis-cluster.redis.svc:6379
    password: "${REDIS_PASSWORD}"

persistence:
  enabled: true
  resourcePolicy: keep
  imageChartStorage:
    type: s3
    s3:
      region: us-east-1
      bucket: harbor-registry
      accesskey: "${AWS_ACCESS_KEY}"
      secretkey: "${AWS_SECRET_KEY}"
      rootdirectory: /registry
  persistentVolumeClaim:
    registry:
      size: 500Gi
    jobservice:
      size: 10Gi
    trivy:
      size: 20Gi

trivy:
  enabled: true
  gitHubToken: "${GITHUB_TOKEN}"
  replicas: 2

core:
  replicas: 3
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2
      memory: 4Gi

registry:
  replicas: 3
  resources:
    requests:
      cpu: 500m
      memory: 1Gi

jobservice:
  replicas: 2

exporter:
  enabled: true
```

## Upgrading

```bash
helm repo update
helm upgrade harbor harbor/harbor \
  --namespace harbor \
  --values values.yaml \
  --version 1.19.1
```

### Migration Path
- Harbor v2.11.0+ → v2.15.0 directly (via `goharbor/prepare` Docker image)
- < v2.11.0 requires sequential upgrades through intermediate versions
- **Must backup ALL data before migration**
- External PostgreSQL must be ≥ v12

## Common Mistakes

- **Admin password change** — Changing `harborAdminPassword` after initial deploy does NOT update the password. Change via UI or API.
- **Internal DB in production** — Internal PostgreSQL is single-Pod. Use external PostgreSQL with HA for production.
- **Notary v1 deprecated** — Disabled since v2.9. Don't enable unless you still need it.
- **Trivy DB download** — Without `trivy.gitHubToken`, Trivy DB downloads are rate-limited to 60 req/hr. Set a GitHub token.
- **S3 region mismatch** — Registry S3 bucket and IAM credentials must match the configured region.
- **Exporter credentials** — Prometheus exporter uses the same admin credentials. Set `exporter.secret` for a dedicated monitoring password.
- **Upgrade schema migration** — `goharbor/prepare` must run during upgrade to apply DB schema migrations. The Helm chart handles this automatically.
