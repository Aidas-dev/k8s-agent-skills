# MariaDB Operator — Helm Charts

**Repository:** `https://mariadb-operator.github.io/mariadb-operator`  
**OCI:** `oci://ghcr.io/mariadb-operator/charts`  

## Charts (3)

| Chart | Latest | Purpose |
|-------|--------|---------|
| `mariadb-operator-crds` | 26.6.0 | CRDs only (recommended: manage separately) |
| `mariadb-operator` | 26.6.0 | Controller + cert-controller + webhook + metrics |
| `mariadb-cluster` | 26.6.0 | MariaDB cluster + MaxScale + backups + users |

**App version:** 26.6.0  
**Images:** `ghcr.io/mariadb-operator/mariadb-operator:26.6.0` (single image used for controller, webhook, cert-controller)

## Quick Install

```bash
helm repo add mariadb-operator https://mariadb-operator.github.io/mariadb-operator
helm repo update

# 1. Install CRDs (separate chart to prevent accidental deletion)
helm install mariadb-operator-crds mariadb-operator/mariadb-operator-crds \
  --version 26.6.0

# 2. Install operator
helm install mariadb-operator mariadb-operator/mariadb-operator \
  --namespace mariadb-operator --create-namespace \
  --version 26.6.0

# 3. (Optional) Provision a MariaDB cluster
helm install mariadb-cluster mariadb-operator/mariadb-cluster \
  --namespace mariadb-operator \
  --version 26.6.0
```

### Using OCI Registry

```bash
helm pull oci://ghcr.io/mariadb-operator/charts/mariadb-operator --version 26.6.0
helm pull oci://ghcr.io/mariadb-operator/charts/mariadb-operator-crds --version 26.6.0
helm pull oci://ghcr.io/mariadb-operator/charts/mariadb-cluster --version 26.6.0
```

## mariadb-operator-crds

Manages CRDs independently. Safer than `crds.enabled: true` on the operator chart.

```bash
helm install mariadb-operator-crds mariadb-operator/mariadb-operator-crds \
  --version 26.6.0
```

This chart has no values — it only ships CRDs.

## mariadb-operator

Deploys the controller, webhook, cert-controller, RBAC, service accounts, and ServiceMonitors.

### Requirements

- Kubernetes >= 1.26
- Prometheus operator (optional, for ServiceMonitors)
- cert-manager (optional, for webhook certs — otherwise cert-controller handles them)

### Installation with HA

```bash
helm install mariadb-operator mariadb-operator/mariadb-operator \
  --namespace mariadb-operator \
  --set ha.enabled=true \
  --set 'metrics.enabled=true' \
  --set 'webhook.ha.enabled=true' \
  --set 'certController.ha.enabled=true'
```

### Key Values

#### Controller

| Parameter | Default | Description |
|-----------|---------|-------------|
| `image.repository` | `ghcr.io/mariadb-operator/mariadb-operator` | Controller image |
| `image.tag` | `""` (chart appVersion) | Image tag |
| `image.pullPolicy` | `IfNotPresent` | Pull policy |
| `logLevel` | `INFO` | Log level |
| `clusterName` | `cluster.local` | Cluster DNS name |
| `currentNamespaceOnly` | `false` | Watch only operator namespace |
| `ha.enabled` | `false` | Enable controller HA |
| `ha.replicas` | `3` | HA replicas |
| `crds.enabled` | `false` | Let operator manage CRDs (⚠️ WARNING: uninstall deletes CRDs + all MariaDB instances) |
| `resources` | `{}` | Container resources |
| `affinity` | `{}` | Pod affinity |
| `tolerations` | `[]` | Node tolerations |
| `topologySpreadConstraints` | `[]` | Topology spread |
| `nodeSelector` | `{}` | Node selector |
| `priorityClassName` | `""` | Priority class |
| `podAnnotations` | `{}` | Pod annotations |
| `podSecurityContext` | `{}` | Pod security context |
| `securityContext` | `{}` | Container security context |
| `extraArgs` | `[]` | Extra controller args |
| `extraEnv` | `[]` | Extra env vars |
| `extraEnvFrom` | `[]` | Extra envFrom |
| `extraVolumes` | `[]` | Extra volumes |
| `extraVolumeMounts` | `[]` | Extra volume mounts |
| `revisionHistoryLimit` | `10` | Revision history limit |
| `strategy` | `{}` | Deployment strategy |
| `serviceAccount.annotations` | `{}` | SA annotations |
| `serviceAccount.automount` | `true` | Auto-mount token |
| `serviceAccount.extraLabels` | `{}` | SA extra labels |
| `serviceAccount.name` | `""` | SA name |
| `rbac.enabled` | `true` | Create RBAC |
| `rbac.aggregation.enabled` | `true` | Aggregate to view/edit roles |
| `pdb.enabled` | `false` | PDB for controller |
| `pdb.maxUnavailable` | `1` | PDB max unavailable |

#### Metrics

| Parameter | Default | Description |
|-----------|---------|-------------|
| `metrics.enabled` | `false` | Enable internal metrics |
| `metrics.serviceMonitor.enabled` | `true` | Create ServiceMonitor |
| `metrics.serviceMonitor.interval` | `30s` | Scrape interval |
| `metrics.serviceMonitor.scrapeTimeout` | `25s` | Scrape timeout |
| `metrics.serviceMonitor.additionalLabels` | `{}` | Extra labels |

#### Webhook

| Parameter | Default | Description |
|-----------|---------|-------------|
| `webhook.enabled` | `true` | Enable webhook |
| `webhook.port` | `9443` | Webhook port |
| `webhook.hostNetwork` | `false` | Host network |
| `webhook.ha.enabled` | `false` | Webhook HA |
| `webhook.ha.replicas` | `3` | Webhook replicas |
| `webhook.resources` | `{}` | Webhook resources |
| `webhook.cert.certManager.enabled` | `false` | Use cert-manager instead of cert-controller |
| `webhook.cert.certManager.issuerRef` | `{}` | Issuer reference |
| `webhook.cert.certManager.duration` | `""` | Cert duration |
| `webhook.cert.path` | `/tmp/k8s-webhook-server/serving-certs` | Cert mount path |
| `webhook.cert.secretAnnotations` | `{}` | TLS secret annotations |
| `webhook.cert.secretLabels` | `{}` | TLS secret labels |
| `webhook.serviceMonitor.enabled` | `true` | Webhook ServiceMonitor |
| `webhook.annotations` | `{}` | Webhook config annotations |
| `webhook.extraArgs` | `[]` | Extra webhook args |
| `webhook.pdb.enabled` | `false` | Webhook PDB |

#### Cert-Controller

| Parameter | Default | Description |
|-----------|---------|-------------|
| `certController.enabled` | `true` | Enable cert-controller |
| `certController.caLifetime` | `26280h` (3 years) | CA cert lifetime |
| `certController.certLifetime` | `2160h` (90 days) | Cert lifetime |
| `certController.renewBeforePercentage` | `33` | Renew at 33% remaining |
| `certController.requeueDuration` | `5m` | Cert renewal check interval |
| `certController.ha.enabled` | `false` | Cert-controller HA |
| `certController.ha.replicas` | `3` | Cert-controller replicas |
| `certController.resources` | `{}` | Cert-controller resources |
| `certController.pdb.enabled` | `false` | Cert-controller PDB |

#### Config (operator defaults)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `config.galeraLibPath` | `/usr/lib/galera/libgalera_smm.so` | Galera library path |
| `config.mariadbDefaultVersion` | `11.8` | Default MariaDB version |
| `config.mariadbImage.repository` | `mariadb` | Default image repo |
| `config.mariadbImage.tag` | `11.8.8` | Default image tag |
| `config.mariadbImageName` | `mariadb` | Default image name |
| `config.maxscaleImage.repository` | `mariadb/maxscale` | Default MaxScale repo |
| `config.maxscaleImage.tag` | `23.08.5` | Default MaxScale tag |
| `config.exporterImage.repository` | `prom/mysqld-exporter` | Exporter repo |
| `config.exporterImage.tag` | `v0.15.1` | Exporter tag |
| `config.exporterMaxscaleImage.repository` | `mariadb/maxscale-prometheus-exporter-ubi` | MaxScale exporter |
| `config.exporterMaxscaleImage.tag` | `v0.0.1` | MaxScale exporter tag |

### HA Production Example

```yaml
ha:
  enabled: true
  replicas: 3
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
webhook:
  enabled: true
  ha:
    enabled: true
    replicas: 3
  cert:
    certManager:
      enabled: true
      issuerRef:
        name: letsencrypt-prod
        kind: ClusterIssuer
certController:
  enabled: true
  ha:
    enabled: true
    replicas: 3
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app.kubernetes.io/name
              operator: In
              values:
                - mariadb-operator
        topologyKey: kubernetes.io/hostname
resources:
  requests:
    cpu: 100m
    memory: 64Mi
  limits:
    cpu: 500m
    memory: 256Mi
pdb:
  enabled: true
  maxUnavailable: 1
```

## mariadb-cluster

Provisions a MariaDB cluster (CRD resources). Generates MariaDB, Database, User, Grant, Backup, etc. CRs.

### Requirements

- `mariadb-operator-crds` installed
- `mariadb-operator` deployed and running

### Installation

```bash
helm install mariadb-cluster mariadb-operator/mariadb-cluster \
  --namespace mariadb-operator \
  --version 26.6.0 \
  --set mariadb.replicas=3 \
  --set mariadb.galera.enabled=true \
  --set mariadb.storage.size=10Gi
```

### Key Values

#### MariaDB CR

| Parameter | Default | Description |
|-----------|---------|-------------|
| `mariadb.replicas` | `3` | Replica count |
| `mariadb.galera.enabled` | `true` | Enable Galera |
| `mariadb.storage.size` | `1Gi` | PVC size |
| `mariadb.rootPasswordSecretKeyRef.name` | `""` (auto-generated) | Root password secret name |
| `mariadb.rootPasswordSecretKeyRef.key` | `root` | Secret key |
| `mariadb.rootPasswordSecretKeyRef.generate` | `true` | Auto-generate password |

#### Databases, Users, Grants

| Parameter | Type | Description |
|-----------|------|-------------|
| `databases` | `[]object` | List of Database CRs (see operator skill for fields) |
| `users` | `[]object` | List of User CRs |
| `grants` | `[]object` | List of Grant CRs |

#### Backups

| Parameter | Type | Description |
|-----------|------|-------------|
| `backups` | `[]object` | List of Backup CRs |
| `physicalBackups` | `[]object` | List of PhysicalBackup CRs |

### Cluster with Apps Example

```yaml
mariadb:
  replicas: 3
  galera:
    enabled: true
  storage:
    size: 50Gi
  rootPasswordSecretKeyRef:
    generate: true

databases:
  - name: myapp
    characterSet: utf8mb4
    collate: utf8mb4_unicode_ci

users:
  - name: myapp
    passwordSecretKeyRef:
      name: myapp-password
      key: password
      generate: true
    maxUserConnections: 50

grants:
  - name: myapp-rw
    privileges:
      - "ALL PRIVILEGES"
    database: "myapp"
    table: "*"
    username: myapp
    grantOption: true

backups:
  - name: nightly-s3
    schedule:
      cron: "0 0 * * *"
    compression: gzip
    maxRetention: 720h
    storage:
      s3:
        bucket: mariadb-backups
        prefix: cluster
        endpoint: s3.amazonaws.com
        region: us-east-1
        accessKeyIdSecretKeyRef:
          name: s3-creds
          key: access-key-id
        secretAccessKeySecretKeyRef:
          name: s3-creds
          key: secret-access-key
```

## Upgrading

```bash
helm repo update

# Upgrade CRDs first
helm upgrade mariadb-operator-crds mariadb-operator/mariadb-operator-crds \
  --version 26.6.0

# Then upgrade operator
helm upgrade mariadb-operator mariadb-operator/mariadb-operator \
  --namespace mariadb-operator \
  --version 26.6.0 \
  --values values.yaml
```

### Upgrade Notes

- Upgrade CRDs **before** the operator to ensure schema compatibility.
- Controller restarts are expected during upgrade (webhook certs may need re-issuance).
- MariaDB clusters managed by the operator are **not** affected by operator upgrades.
- Check the [GitHub releases](https://github.com/mariadb-operator/mariadb-operator/releases) for breaking changes.

## Uninstalling

```bash
# Uninstall cluster first (this deletes MariaDB CRs = data loss)
helm uninstall mariadb-cluster --namespace mariadb-operator

# Uninstall operator
helm uninstall mariadb-operator --namespace mariadb-operator

# Uninstall CRDs (⚠️ DELETES ALL MariaDB CRs in the cluster)
helm uninstall mariadb-operator-crds
```

**⚠️ WARNING:** If `crds.enabled: true` was set on `mariadb-operator`, uninstalling it deletes ALL CRDs, which cascades to delete ALL MariaDB instances across ALL namespaces.

## Common Mistakes

- **CRDs not installed first** — Without `mariadb-operator-crds`, the operator and cluster chart fail with "no matching CRD found".
- **crds.enabled: true risk** — Only set during initial deploy. If you uninstall the operator chart with this enabled, all MariaDB instances are deleted.
- **mariadb-cluster without operator** — The cluster chart CRs require the operator running to reconcile.
- **Metrics ServiceMonitor needs prometheus-operator** — ServiceMonitors require the prometheus-operator CRDs installed.
- **cert-manager vs cert-controller** — Default is cert-controller (built-in). Set `webhook.cert.certManager.enabled: true` to use cert-manager instead.
- **HA without anti-affinity** — Controller replicas may schedule on the same node. Set `affinity.podAntiAffinity` for true HA.
- **Default image tags** — Operator defaults to mariadb 11.8.8 and maxscale 23.08.5. Override `config.*` to change defaults.
- **logLevel** — Valid values: `DEBUG`, `INFO`, `WARN`, `ERROR`. Default is `INFO`.
- **currentNamespaceOnly** — When `true`, operator only watches CRs in its own namespace. Doesn't affect cluster-scoped CRDs.
- **Root password auto-generation** — `generate: true` on rootPasswordSecretKeyRef creates the Secret automatically. Don't create a Secret with the same name.
