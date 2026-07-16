---
name: mariadb-operator
description: Create and manage MariaDB operator CRDs — 12 CRDs under k8s.mariadb.com, Galera HA, MaxScale, backups, PITR, monitoring, and connection pooling.
---

# MariaDB Operator CRDs

**Repository:** `github.com/mariadb-operator/mariadb-operator`  
**Latest release:** v26.6.0 (Jun 5, 2026)  
**API version:** `k8s.mariadb.com/v1alpha1`  
**CRDs:** 12  
**Kubernetes:** >=1.26  
**Go:** 1.26, controller-runtime 0.23.3  

## Overview

```
MariaDB CR ──► StatefulSet with Galera HA
MaxScale CR ──► MaxScale proxy Deployment (read/write splitting)
  │
Database CR ──► Database creation
User CR ──► User creation + password rotation
Grant CR ──► Privilege assignment
  │
Backup CR ──► Scheduled backups (S3/Azure/Volume)
PhysicalBackup CR ──► Physical snapshot backups
Restore CR ──► Restore from backup
PointInTimeRecovery CR ──► PITR to specific timestamp
  │
Connection CR ──► Connection string in Secret
SqlJob CR ──► Scheduled SQL execution
  │
ExternalMariaDB CR ──► Connect to external MariaDB instances
```

## CRDs

| CRD | Short Name | Scope | Description |
|-----|-----------|-------|-------------|
| `MariaDB` | `mdb` | Namespaced | Galera cluster |
| `MaxScale` | `mxs` | Namespaced | MaxScale proxy |
| `Backup` | `bmdb` | Namespaced | Logical backup |
| `PhysicalBackup` | `pbmdb` | Namespaced | Physical snapshot |
| `Restore` | `rmdb` | Namespaced | Restore from Backup |
| `PointInTimeRecovery` | `pitr` | Namespaced | PITR to timestamp |
| `Database` | `dmdb` | Namespaced | Database creation |
| `User` | `umdb` | Namespaced | User management |
| `Grant` | `gmdb` | Namespaced | Privilege grants |
| `Connection` | `cmdb` | Namespaced | Connection secret |
| `SqlJob` | `smdb` | Namespaced | SQL execution |
| `ExternalMariaDB` | `emdb` | Namespaced | External DB connection |

### MariaDB

The primary CRD defining a MariaDB Galera cluster.

```yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: MariaDB
metadata:
  name: mariadb-cluster
spec:
  # Image (v26.3.0+: image.repo + image.tag split)
  image:
    repository: mariadb
    tag: 11.8.8

  # Galera HA
  galera:
    enabled: true
    primary:
      podIndex: 0
      automaticFailover: true
    config:
      reuseStorageVolume: false
      # Additional galera config
    init:
      job:
        image:
          repository: mariadb
          tag: 11.8.8
      resources: {}

  # Volume claim
  storage:
    size: 10Gi
    storageClassName: standard
    # accessModes:
    #   - ReadWriteOnce
    #   - ReadWriteMany
    #   - ReadOnlyMany
    #   - ReadWriteOncePod
    resizeInUseVolumes: true
    waitForVolume:
      enabled: false

  # Replicas
  replicas: 3

  # Service
  service:
    type: ClusterIP
    # Additional service template

  # Connection
  rootPasswordSecretKeyRef:
    name: mariadb-root
    key: password
    # generate: true  # auto-generate
  username: ""
  passwordSecretKeyRef: {}
  database: ""

  # Metrics
  metrics:
    enabled: true
    exporter:
      image:
        repository: prom/mysqld-exporter
        tag: v0.15.1
      resources: {}

  # Connection pool
  connectionPool:
    port: 3306
    maxConnections: 100
    minPoolSize: 1

  # Update strategy
  updateStrategy:
    type: RollingUpdate
    autoSwapPvc:
      - statefulset-1

  # Pod spec
  podAnnotations: {}
  imagePullSecrets: []
  priorityClassName: ""
  affinity: {}
  tolerations: []
  nodeSelector: {}
  topologySpreadConstraints: []

  # Resource limits
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

  # TLS
  tls:
    enabled: false
    # ssl:
    #   secretName: mariadb-tls
    #   caSecretName: mariadb-ca
    # clientCertSecretName: mariadb-client

  # MaxScale reference
  maxScaleRef: {}

  # Replication
  replication:
    # Source connection ref for replica
    source: {}

  # Maintenance
  maintenance:
    enabled: false
    # Requeue interval during maintenance
    requeueInterval: 30s

  # HA
  ha:
    enable: true
    cert: {}
  #   cert:
  #     certManager:
  #       enabled: true
  #       duration: 2160h

  # Default configuration
  myCnf: |
    [mysqld]
    character-set-server=utf8mb4
    collation-server=utf8mb4_unicode_ci
    max_connections=100

  # PodDisruptionBudget
  podDisruptionBudget:
    enabled: true
    maxUnavailable: 1

  # Progress deadlines
  progressDeadlineSeconds: 600
```

#### Spec Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `image.repository` | string | `mariadb` | MariaDB image repo |
| `image.tag` | string | `11.8.8` | MariaDB image tag |
| `image.pullPolicy` | string | `IfNotPresent` | Pull policy |
| `galera.enabled` | bool | `false` | Enable Galera cluster |
| `galera.primary.podIndex` | int | `0` | Primary pod index |
| `galera.primary.automaticFailover` | bool | `true` | Auto failover |
| `galera.config.reuseStorageVolume` | bool | `false` | Reuse PVC on rejoin |
| `galera.init.job.image` | object | `{}` | Init job image |
| `storage.size` | string | — | PVC size |
| `storage.storageClassName` | string | — | Storage class |
| `storage.resizeInUseVolumes` | bool | `true` | Resize PVC in use |
| `storage.waitForVolume.enabled` | bool | `false` | Wait for volume attach |
| `replicas` | int | `3` | Replica count |
| `service.type` | string | `ClusterIP` | Service type |
| `rootPasswordSecretKeyRef` | object | — | Root password secret ref |
| `rootPasswordSecretKeyRef.generate` | bool | `false` | Auto-generate password |
| `username` | string | `""` | Regular user |
| `passwordSecretKeyRef` | object | — | User password secret |
| `database` | string | `""` | Database name |
| `metrics.enabled` | bool | `false` | Enable exporter |
| `metrics.exporter.image` | object | — | Exporter image |
| `connectionPool.port` | int | `3306` | DB port |
| `connectionPool.maxConnections` | int | `100` | Max connections |
| `updateStrategy.type` | string | `RollingUpdate` | Update strategy |
| `updateStrategy.autoSwapPvc` | []string | `[]` | Auto-swap PVCs |
| `resources` | object | `{}` | Container resources |
| `tls.enabled` | bool | `false` | Enable TLS |
| `tls.ssl.secretName` | string | — | TLS secret |
| `myCnf` | string | — | my.cnf config |
| `podDisruptionBudget.enabled` | bool | `false` | PDB |
| `podDisruptionBudget.maxUnavailable` | int | `1` | Max unavailable |
| `progressDeadlineSeconds` | int | `600` | Progress deadline |
| `maintenance.enabled` | bool | `false` | Maintenance mode |
| `maintenance.requeueInterval` | string | `30s` | Requeue interval |
| `ha.enable` | bool | `true` | Enable HA |
| `ha.cert` | object | — | HA TLS cert config |
| `replication.source` | object | — | Replication source ref |

### MaxScale

MariaDB MaxScale proxy for read/write splitting and connection routing.

```yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: MaxScale
metadata:
  name: maxscale
spec:
  image:
    repository: mariadb/maxscale
    tag: 23.08.5

  replicas: 2

  # MariaDB reference
  mariaDbRef:
    name: mariadb-cluster
    waitForIt: true

  # Configuration
  config:
    maxscale:
      # global MaxScale config
      admin_host: 0.0.0.0
      admin_port: 8989
    servers:
      - address: mariadb-cluster
        port: 3306
        protocol: MariaDBBackend
    monitors:
      - module: galeramon
        servers:
          - mariadb-cluster
        user: maxscale
        password: maxscale
    services:
      - router: readwritesplit
        servers:
          - mariadb-cluster
        user: maxscale
        password: maxscale
    listeners:
      - protocol: MariaDBClient
        service: readwritesplit
        port: 3306
      - protocol: HTTP
        service: restapi
        port: 8989

  # Authentication
  auth:
    adminUsername: admin
    adminPasswordSecretKeyRef:
      name: maxscale-admin
      key: password
    # serverCredentials:
    #   - user: maxscale
    #     passwordSecretKeyRef:
    #       name: maxscale
    #       key: password

  # Metrics
  metrics:
    enabled: true
    exporter:
      image:
        repository: mariadb/maxscale-prometheus-exporter-ubi
        tag: v0.0.1
      resources: {}

  # Kubernetes Service
  kubernetesService:
    type: LoadBalancer
    annotations: {}
    metadata: {}

  # Services (non-K8s, MaxScale internal services)
  services:
    - name: restapi
      router: restapi
      port: 8989

  # TLS
  tls:
    enabled: false

  # Resources
  resources:
    requests:
      cpu: 200m
      memory: 256Mi

  # Pod spec
  affinity: {}
  tolerations: []
  nodeSelector: {}
  priorityClassName: ""
  podAnnotations: {}
  updateStrategy: {}

  # MaxScale configuration without full override
  # Use config overrides vs full config
```

#### Spec Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `image` | object | — | MaxScale image |
| `replicas` | int | `1` | Replica count |
| `mariaDbRef` | object | — | Reference to MariaDB CR |
| `mariaDbRef.waitForIt` | bool | `true` | Wait for MariaDB |
| `config` | object | — | Inline MaxScale config |
| `auth.adminUsername` | string | `admin` | Admin username |
| `auth.adminPasswordSecretKeyRef` | object | — | Admin password ref |
| `auth.serverCredentials` | []object | — | Server creds |
| `metrics.enabled` | bool | `false` | Enable metrics |
| `metrics.exporter` | object | — | Exporter image |
| `kubernetesService.type` | string | `ClusterIP` | K8s service type |
| `services` | []object | — | MaxScale internal services |
| `tls.enabled` | bool | `false` | Enable TLS |
| `resources` | object | `{}` | Container resources |
| `updateStrategy` | object | — | Update strategy |
| `podDisruptionBudget` | object | — | PDB config |

### Backup

Scheduled logical backups.

```yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: Backup
metadata:
  name: backup-s3
spec:
  mariaDbRef:
    name: mariadb-cluster
    waitForIt: true
    waitForItTimeout: 30s

  schedule:
    cron: "0 */6 * * *"
    suspend: false
    # immediate: true

  compression: gzip
  maxRetention: 720h  # 30 days

  storage:
    s3:
      bucket: mariadb-backups
      prefix: prod
      endpoint: s3.eu-west-1.amazonaws.com
      region: eu-west-1
      accessKeyIdSecretKeyRef:
        name: s3-credentials
        key: access-key-id
      secretAccessKeySecretKeyRef:
        name: s3-credentials
        key: secret-access-key
      tls:
        enabled: true
        caSecretKeyRef:
          name: s3-ca
          key: ca.crt

  # Also supports Azure Blob:
  # storage:
  #   azure:
  #     containerName: backups
  #     storageAccount: myaccount
  #     accessKeySecretKeyRef:
  #       name: azure-credentials
  #       key: access-key

  # Or emptyDir volume:
  # storage:
  #   volumes:
  #     persistentVolumeClaim:
  #       claimName: backup-pvc

  # Backup job resources
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

  # Log level
  logLevel: info
```

#### Spec Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `mariaDbRef.name` | string | — | MariaDB CR name |
| `mariaDbRef.waitForIt` | bool | `true` | Wait for MariaDB |
| `mariaDbRef.waitForItTimeout` | string | — | Wait timeout |
| `schedule.cron` | string | — | Cron expression |
| `schedule.suspend` | bool | `false` | Suspend schedule |
| `compression` | string | — | `gzip` or `none` |
| `maxRetention` | string | — | Retention duration |
| `storage.s3` | object | — | S3 storage config |
| `storage.s3.bucket` | string | — | Bucket name |
| `storage.s3.prefix` | string | — | Key prefix |
| `storage.s3.endpoint` | string | — | S3 endpoint |
| `storage.s3.region` | string | — | S3 region |
| `storage.s3.accessKeyIdSecretKeyRef` | object | — | Access key ref |
| `storage.s3.secretAccessKeySecretKeyRef` | object | — | Secret key ref |
| `storage.s3.tls` | object | — | S3 TLS config |
| `storage.azure` | object | — | Azure Blob config |
| `storage.volumes` | object | — | PVC volume config |
| `resources` | object | — | Job resources |
| `logLevel` | string | `info` | Log level |

### PhysicalBackup

Physical snapshot backup (filesystem-level, faster restore).

```yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: PhysicalBackup
metadata:
  name: physical-backup-s3
spec:
  mariaDbRef:
    name: mariadb-cluster
    waitForIt: true

  schedule:
    cron: "0 0 * * *"
    suspend: false
    immediate: false

  compression: gzip
  maxRetention: 720h  # 30 days

  storage:
    s3:
      bucket: mariadb-physical-backups
      prefix: prod
      endpoint: s3.eu-west-1.amazonaws.com
      region: eu-west-1
      accessKeyIdSecretKeyRef:
        name: s3-credentials
        key: access-key-id
      secretAccessKeySecretKeyRef:
        name: s3-credentials
        key: secret-access-key

  resources:
    requests:
      cpu: 200m
      memory: 256Mi
```

#### Spec Fields

Same as Backup but for physical snapshots. Uses mariadb-backup tool internally. Faster to restore than logical Backup.

| Field | Type | Description |
|-------|------|-------------|
| `mariaDbRef` | object | MariaDB reference |
| `schedule` | object | Cron schedule |
| `compression` | string | Compression method |
| `maxRetention` | string | Retention duration |
| `storage.s3` | object | S3 storage (same as Backup) |
| `storage.azure` | object | Azure Blob |
| `resources` | object | Job resources |

### Restore

Restore a MariaDB from a Backup.

```yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: Restore
metadata:
  name: restore-from-backup
spec:
  mariaDbRef:
    name: mariadb-cluster
    waitForIt: true

  backupRef:
    name: backup-s3

  # OR restore from PhysicalBackup:
  # physicalBackupRef:
  #   name: physical-backup-s3

  resources:
    requests:
      cpu: 200m
      memory: 256Mi
```

#### Spec Fields

| Field | Type | Description |
|-------|------|-------------|
| `mariaDbRef` | object | Target MariaDB reference |
| `backupRef.name` | string | Backup CR name (logical restore) |
| `physicalBackupRef.name` | string | PhysicalBackup CR name |
| `resources` | object | Job resources |

### PointInTimeRecovery

Recover to a specific timestamp.

```yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: PointInTimeRecovery
metadata:
  name: pitr-recovery
spec:
  mariaDbRef:
    name: mariadb-cluster
    waitForIt: true

  backupRef:
    name: backup-s3

  targetRecoveryTime: "2026-06-10T14:30:00Z"

  resources:
    requests:
      cpu: 200m
      memory: 256Mi
```

#### Spec Fields

| Field | Type | Description |
|-------|------|-------------|
| `mariaDbRef` | object | Target MariaDB |
| `backupRef.name` | string | Base Backup CR |
| `targetRecoveryTime` | string | RFC3339 timestamp to recover to |
| `resources` | object | Job resources |

### Database

Create a database within a MariaDB instance.

```yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: Database
metadata:
  name: app-db
spec:
  mariaDbRef:
    name: mariadb-cluster
    waitForIt: true

  name: myapp
  characterSet: utf8mb4
  collate: utf8mb4_unicode_ci

  cleanupPolicy: Delete  # Delete | Skip

  requeueInterval: 30s
  retryInterval: 5s
```

#### Spec Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `mariaDbRef` | object | — | MariaDB reference |
| `name` | string | — | Database name |
| `characterSet` | string | — | Character set |
| `collate` | string | — | Collation |
| `cleanupPolicy` | string | `Delete` | `Delete` or `Skip` |
| `requeueInterval` | string | — | Reconciliation interval |
| `retryInterval` | string | `5s` | Retry interval on error |

### User

Create a database user.

```yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: User
metadata:
  name: app-user
spec:
  mariaDbRef:
    name: mariadb-cluster
    waitForIt: true

  name: myapp
  passwordSecretKeyRef:
    name: app-password
    key: password
  # generate: true   # auto-generate password
  # passwordRotation:  # v26.6.0+
  #   enabled: true
  #   rotationInterval: 720h

  maxUserConnections: 30
  host: "%"

  cleanupPolicy: Delete
  requeueInterval: 30s
  retryInterval: 5s
```

#### Spec Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `mariaDbRef` | object | — | MariaDB reference |
| `name` | string | — | Username |
| `passwordSecretKeyRef` | object | — | Password secret ref |
| `passwordSecretKeyRef.generate` | bool | `false` | Auto-generate |
| `passwordRotation.enabled` | bool | `false` | Rotate password |
| `passwordRotation.rotationInterval` | string | — | Rotation interval |
| `maxUserConnections` | int | — | Connection limit |
| `host` | string | `%` | Allowed host |
| `cleanupPolicy` | string | `Delete` | Cleanup on CR delete |
| `requeueInterval` | string | — | Reconcile interval |
| `retryInterval` | string | `5s` | Retry interval |

### Grant

Assign privileges to a user on a database.

```yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: Grant
metadata:
  name: app-grant
spec:
  mariaDbRef:
    name: mariadb-cluster
    waitForIt: true

  privileges:
    - "ALL PRIVILEGES"
  database: "myapp"
  table: "*"
  username: myapp
  grantOption: true
  host: "%"

  cleanupPolicy: Delete
  requeueInterval: 30s
  retryInterval: 5s
```

#### Spec Fields

| Field | Type | Description |
|-------|------|-------------|
| `mariaDbRef` | object | MariaDB reference |
| `privileges` | []string | List of privileges (`ALL PRIVILEGES`, `SELECT`, `INSERT`, etc.) |
| `database` | string | Database name |
| `table` | string | Table name (`*` for all) |
| `username` | string | Username |
| `grantOption` | bool | Allow GRANT OPTION |
| `host` | string | Host pattern |
| `cleanupPolicy` | string | Cleanup on CR delete |
| `requeueInterval` | string | Reconcile interval |
| `retryInterval` | string | Retry interval |

### Connection

Expose a connection string as a Kubernetes Secret.

```yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: Connection
metadata:
  name: app-connection
spec:
  mariaDbRef:
    name: mariadb-cluster
    waitForIt: true

  host: mariadb-cluster
  port: 3306
  userName: myapp
  passwordSecretKeyRef:
    name: app-password
    key: password
  database: myapp

  # Template for the connection string
  # template: "mysql://{{ .userName }}:{{ .password }}@{{ .host }}:{{ .port }}/{{ .database }}"

  # Secret name to output
  secretName: app-dsn

  # Health check
  healthCheck:
    interval: 30s
    retryInterval: 5s
```

#### Spec Fields

| Field | Type | Description |
|-------|------|-------------|
| `mariaDbRef` | object | MariaDB reference |
| `host` | string | Database host |
| `port` | int | Database port |
| `userName` | string | Username |
| `passwordSecretKeyRef` | object | Password secret ref |
| `database` | string | Database name |
| `secretName` | string | Output Secret name |
| `template` | string | Go template for DSN string |
| `healthCheck` | object | Health check config |

### SqlJob

Execute SQL statements, optionally on a schedule.

```yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: SqlJob
metadata:
  name: migrate-schema
spec:
  mariaDbRef:
    name: mariadb-cluster
    waitForIt: true

  # One-time execution
  sql: |
    CREATE TABLE IF NOT EXISTS migrations (
      id INT AUTO_INCREMENT PRIMARY KEY,
      name VARCHAR(255) NOT NULL,
      applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );

  # OR from a ConfigMap:
  # sqlConfigMapKeyRef:
  #   name: migration-sql
  #   key: migrate.sql

  # OR on a schedule:
  # schedule:
  #   cron: "0 0 * * 0"

  # Arguments:
  args: []

  # Database and user context
  database: myapp
  username: myapp
  passwordSecretKeyRef:
    name: app-password
    key: password

  # Log level
  logLevel: info

  resources:
    requests:
      cpu: 100m
      memory: 64Mi

  # Retry
  retryInterval: 5s
```

#### Spec Fields

| Field | Type | Description |
|-------|------|-------------|
| `mariaDbRef` | object | MariaDB reference |
| `sql` | string | Inline SQL |
| `sqlConfigMapKeyRef` | object | SQL from ConfigMap |
| `schedule.cron` | string | Cron schedule (optional) |
| `args` | []string | SQL args |
| `database` | string | Target database |
| `username` | string | SQL user |
| `passwordSecretKeyRef` | object | Password ref |
| `logLevel` | string | Log level |
| `resources` | object | Job resources |
| `retryInterval` | string | Retry on error |

### ExternalMariaDB

Connect to an external (out-of-cluster) MariaDB instance.

```yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: ExternalMariaDB
metadata:
  name: external-db
spec:
  # Connection parameters
  host: db.example.com
  port: 3306
  username: admin
  passwordSecretKeyRef:
    name: external-db-creds
    key: password

  # TLS
  tls:
    enabled: true
    caSecretKeyRef:
      name: external-db-ca
      key: ca.crt

  # Connection pool
  connectionPool:
    port: 3307
    maxConnections: 50

  # Used by MaxScale, Backup, SqlJob as a valid mariaDbRef target
```

#### Spec Fields

| Field | Type | Description |
|-------|------|-------------|
| `host` | string | External DB host |
| `port` | int | External DB port |
| `username` | string | DB username |
| `passwordSecretKeyRef` | object | Password secret ref |
| `tls.enabled` | bool | Enable TLS |
| `tls.caSecretKeyRef` | object | CA cert ref |
| `connectionPool.port` | int | Local proxy port |
| `connectionPool.maxConnections` | int | Max connections |

## V26.3.0 Config Format Change

Before v26.3.0, MariaDB image was specified as a single string:

```yaml
# OLD (pre-v26.3.0)
spec:
  image: mariadb:11.8.8
```

From v26.3.0+, image is split:

```yaml
# NEW (v26.3.0+)
spec:
  image:
    repository: mariadb
    tag: 11.8.8
```

## Common Patterns

### MariaDB with MaxScale

```yaml
# MariaDB
apiVersion: k8s.mariadb.com/v1alpha1
kind: MariaDB
metadata:
  name: mariadb-cluster
spec:
  image:
    repository: mariadb
    tag: 11.8.8
  galera:
    enabled: true
  replicas: 3
  storage:
    size: 10Gi
  metrics:
    enabled: true
---
# MaxScale
apiVersion: k8s.mariadb.com/v1alpha1
kind: MaxScale
metadata:
  name: maxscale
spec:
  mariaDbRef:
    name: mariadb-cluster
  replicas: 2
  config:
    maxscale:
      admin_host: 0.0.0.0
      admin_port: 8989
  auth:
    adminUsername: admin
    adminPasswordSecretKeyRef:
      name: maxscale-admin
      key: password
  kubernetesService:
    type: LoadBalancer
---
# Database
apiVersion: k8s.mariadb.com/v1alpha1
kind: Database
metadata:
  name: myapp
spec:
  mariaDbRef:
    name: mariadb-cluster
  name: myapp
  characterSet: utf8mb4
---
# User
apiVersion: k8s.mariadb.com/v1alpha1
kind: User
metadata:
  name: myapp-user
spec:
  mariaDbRef:
    name: mariadb-cluster
  name: myapp
  passwordSecretKeyRef:
    name: myapp-password
    key: password
  maxUserConnections: 30
---
# Grant
apiVersion: k8s.mariadb.com/v1alpha1
kind: Grant
metadata:
  name: myapp-grant
spec:
  mariaDbRef:
    name: mariadb-cluster
  privileges:
    - "ALL PRIVILEGES"
  database: "myapp"
  table: "*"
  username: myapp
  grantOption: true
```

### Scheduled S3 Backup

```yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: Backup
metadata:
  name: hourly-backup
spec:
  mariaDbRef:
    name: mariadb-cluster
  schedule:
    cron: "0 * * * *"
  compression: gzip
  maxRetention: 168h  # 7 days
  storage:
    s3:
      bucket: mariadb-backups
      prefix: cluster-hourly
      endpoint: s3.eu-west-1.amazonaws.com
      region: eu-west-1
      accessKeyIdSecretKeyRef:
        name: s3-creds
        key: access-key-id
      secretAccessKeySecretKeyRef:
        name: s3-creds
        key: secret-access-key
```

### Point-in-Time Recovery

```yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: PointInTimeRecovery
metadata:
  name: pitr-recovery
spec:
  mariaDbRef:
    name: mariadb-cluster
  backupRef:
    name: hourly-backup
  targetRecoveryTime: "2026-06-10T14:30:00Z"
```

### SqlJob for Schema Migrations

```yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: SqlJob
metadata:
  name: schema-migrate
spec:
  mariaDbRef:
    name: mariadb-cluster
  database: myapp
  username: myapp
  passwordSecretKeyRef:
    name: myapp-password
    key: password
  sql: |
    CREATE TABLE IF NOT EXISTS users (
      id BIGINT AUTO_INCREMENT PRIMARY KEY,
      email VARCHAR(255) UNIQUE NOT NULL,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    CREATE INDEX idx_users_email ON users(email);
```

### Password Rotation (v26.6.0)

```yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: User
metadata:
  name: app-user
spec:
  mariaDbRef:
    name: mariadb-cluster
  name: app
  passwordRotation:
    enabled: true
    rotationInterval: 720h  # 30 days
  host: "%"
```

### Connection Secret

```yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: Connection
metadata:
  name: app-connection
spec:
  mariaDbRef:
    name: mariadb-cluster
  host: maxscale
  port: 3306
  userName: myapp
  passwordSecretKeyRef:
    name: myapp-password
    key: password
  database: myapp
  secretName: app-dsn
```

### External MariaDB

```yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: ExternalMariaDB
metadata:
  name: legacy-db
spec:
  host: db.legacy.example.com
  port: 3306
  username: readonly
  passwordSecretKeyRef:
    name: legacy-creds
    key: password
  tls:
    enabled: true
    caSecretKeyRef:
      name: legacy-ca
      key: ca.crt
```

## Common Mistakes

- **Image format change in v26.3.0** — Pre-v26.3.0 used `image: mariadb:11.8`. v26.3.0+ uses `image.repository` + `image.tag`. Check your operator version.
- **Galera requires 3+ replicas** — Galera cluster needs at least 3 replicas. 2 replicas causes split-brain.
- **Galera `reuseStorageVolume`** — When re-joining a Galera node, the PVC must be empty. Set `reuseStorageVolume: true` to wipe and reuse.
- **Root password auto-generate** — `rootPasswordSecretKeyRef.generate: true` auto-generates but the password is only available in the Secret. Extract before losing it.
- **MaxScale config complexity** — MaxScale CR uses inline config, not external files. The config structure mirrors MaxScale's native config format.
- **Backup retention precision** — `maxRetention` is a Go duration (720h = 30 days). Old backups are cleaned based on this, not just scheduling.
- **S3 TLS for MinIO** — When using MinIO with self-signed certs, use `storage.s3.tls.caSecretKeyRef` to provide the CA.
- **PhysicalBackup speed** — PhysicalBackup is faster than Backup for large databases but requires filesystem-level access. Use for databases > 50GB.
- **SqlJob idempotency** — SqlJob doesn't track which SQL has been applied. Use `IF NOT EXISTS` / `IF EXISTS` patterns.
- **Password rotation impact** — `passwordRotation` changes the password in both the Secret and MariaDB. Applications using the old password will lose connection.
- **WaitForIt timeout** — Large MariaDB clusters can take minutes to bootstrap. Set `mariaDbRef.waitForItTimeout` appropriately.
- **Connection secret update** — The Connection CR creates a Secret with the DSN. Deleting the Connection CR deletes the Secret.
- **Multi-cluster topology** — v26.6.0 supports `replication.source` for cross-cluster replication between MariaDB CRs.
- **Maintenance mode** — Set `maintenance.enabled: true` to pause reconciliation during manual operations.
