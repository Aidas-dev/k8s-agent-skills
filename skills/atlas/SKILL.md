---
name: atlas
description: Use when working with Atlas Operator on Kubernetes — creating or troubleshooting AtlasSchema or AtlasMigration resources; configuring DevDB, migration policies, credential patterns, or schema diff/lint rules.
---

# Atlas Operator

## Overview

Database schema management on Kubernetes via `db.atlasgo.io/v1alpha1`. Two approaches: **AtlasSchema** (declarative — define desired schema, operator diffs and applies) and **AtlasMigration** (versioned — ordered SQL scripts). Deployed via OCI Helm chart: `ghcr.io/ariga/charts/atlas-operator` (latest: 0.7.32).

## CRD Reference

| CRD | Approach | Use When |
|-----|----------|----------|
| `AtlasSchema` | Declarative | Schema should match a desired state (operator auto-diffs) |
| `AtlasMigration` | Versioned | Need sequential migration files, rollback support, audit trail |

## AtlasSchema (Declarative)

```yaml
apiVersion: db.atlasgo.io/v1alpha1
kind: AtlasSchema
metadata:
  name: app-schema
spec:
  urlFrom:
    secretKeyRef:
      name: db-creds
      key: url              # postgres://user:pass@host:5432/db?sslmode=require

  schema:
    sql: |
      CREATE TABLE users (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        email TEXT UNIQUE NOT NULL,
        created_at TIMESTAMPTZ DEFAULT now()
      );
      CREATE TABLE posts (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        user_id UUID REFERENCES users(id),
        title TEXT NOT NULL
      );

  policy:
    lint:
      destructive:
        error: true          # Block DROP COLUMN / DROP TABLE
    diff:
      skip:
        - drop_column        # Skip dropping columns (compatible changes only)
    exclude:
      - "audit_*"           # Glob patterns to ignore

  prewarmDevDB: true          # Default — ephemeral dev container for diff
  allowCustomConfig: false    # Don't allow DB config changes via atlas
```

## AtlasMigration (Versioned)

```yaml
apiVersion: db.atlasgo.io/v1alpha1
kind: AtlasMigration
metadata:
  name: app-migrations
spec:
  urlFrom:
    secretKeyRef:
      name: db-creds
      key: url

  dir:
    configMapRef:
      name: migration-files   # ConfigMap with SQL migration scripts

  execOrder: linear           # linear | linear-skip | non-linear
  baseline: "000000"          # Start from this version (skip earlier)

  policy:
    lint:
      destructive:
        error: true

  prewarmDevDB: true
  allowCustomConfig: false
```

Migration files in ConfigMap must be lexicographically ordered:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: migration-files
data:
  000001_init.sql: |
    CREATE TABLE users (...);
  000002_add_posts.sql: |
    CREATE TABLE posts (...);
  000003_add_indexes.sql: |
    CREATE INDEX idx_users_email ON users(email);
```

## Credential Patterns

| Pattern | Example |
|---------|---------|
| Connection URL in Secret | `urlFrom.secretKeyRef.key: url` |
| Credentials struct | `credentials.host`, `.user`, `.passwordFrom`, `.database`, `.port` |
| Extra connection parameters | `credentials.parameters: { sslmode: require }` |

Use `credentials` struct when the URL is constructed from multiple Secret keys:

```yaml
spec:
  credentials:
    host: myhost.example.com
    user: app
    passwordFrom:
      secretKeyRef:
        name: db-creds
        key: password
    database: mydb
    port: 5432
```

## Policy Configuration

| Policy | Field | Effect |
|--------|-------|--------|
| `lint.destructive.error` | `true`/`false` | Block (true) or warn (false) on destructive changes |
| `diff.skip` | `[drop_column, drop_index, ...]` | Operations to skip in diff (safe for roll-forward) |
| `exclude` | `["pattern_*"]` | Glob patterns to exclude from management |

## Migration Directory Sources

| Source | Location | Use Case |
|--------|----------|----------|
| `dir.configMapRef` | ConfigMap in same namespace | Simple, no external deps |
| `dir.local.path` | Mounted volume path | Sidecar / init container pattern |
| `dir.remote` | Atlas Cloud URL | Team-managed schema registry |
| Inline (`spec.schema.sql`) | Embedded in CR | Quick prototyping |

## DevDB

Ephemeral database used to compute diff between desired and current schema. If `devURL` not set, operator spins a temporary container. Set explicitly to reuse or for air-gapped:

```yaml
spec:
  devURL: postgres://user:pass@dev-db-host:5432/template?sslmode=disable
```

Or via Secret: `devURLFrom.secretKeyRef`.

## Common Mistakes

- **`allowCustomConfig: true` without need** — allows arbitrary DB config changes; keep `false` unless you need `ALTER SYSTEM SET`
- **No lint policy** — destructive changes silently applied; always set `lint.destructive.error: true` for production
- **Wrong execOrder** — `non-linear` skips version ordering; use `linear` for strict migration order
- **Missing `baseline`** — operator tries to run all migrations including already-applied ones; set `baseline` to skip
- **ConfigMap not updated** — AtlasMigration reads migration dir once at reconcile; if ConfigMap changes, operator picks it up on next loop
- **DevDB connection issues** — operator needs network access to a temporary or explicit devDB; ensure `prewarmDevDB: true` works in your cluster

## Supported Databases

MySQL, MariaDB, PostgreSQL, SQLite, SQL Server, ClickHouse, CockroachDB, TiDB, YugabyteDB.
