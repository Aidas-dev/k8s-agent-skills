---
name: sealed-secrets
description: Use when working with Sealed Secrets — encrypting Kubernetes Secrets for safe Git storage. Covers SealedSecret CRD, kubeseal, key management, encryption flow, Helm values, controller config, key rotation.
---

# Sealed Secrets

## Overview

Sealed Secrets provides one-way encryption for Kubernetes Secrets. A `SealedSecret` custom resource can be safely stored in public Git repos. Only the controller running in the target cluster can decrypt it, recovering the original `Secret`.

**CRD:** `SealedSecret` (`bitnami.com/v1alpha1`).

**Latest:** chart 2.18.5, controller v0.36.6 (Apr 2026).

## Architecture

```
Developer workstation:
  kubeseal --fetch-cert         ← fetches controller's public key
    → creates SealedSecret YAML   ← encrypts Secret data with public key
      → committed to Git

Cluster controller:
  watches SealedSecret resources
    → decrypts using private key
      → creates/reconciled K8s Secret
```

## CRD: SealedSecret

`apiVersion: bitnami.com/v1alpha1`, `kind: SealedSecret`

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: mysecret
  namespace: myapp
spec:
  encryptedData:
    password: AgBy7x...base64-encrypted...
  template:
    metadata:
      labels:
        app: myapp
    type: Opaque   # or kubernetes.io/dockerconfigjson, etc.
```

Key fields:

| Field | Description |
|-------|-------------|
| `spec.encryptedData` | Map of key → encrypted value (the sealed ciphertext) |
| `spec.template.metadata` | Labels/annotations applied to the decrypted Secret |
| `spec.template.type` | Secret type (Opaque, kubernetes.io/dockerconfigjson, etc.) |
| `spec.template.data` | Unencrypted key-value pairs merged into the output Secret |

The namespace in `metadata.namespace` is **binding** — the controller will only create the Secret in the same namespace as the SealedSecret.

## kubeseal CLI

### Install

```bash
# Linux amd64
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.36.6/kubeseal-0.36.6-linux-amd64.tar.gz
tar xzf kubeseal-*-linux-amd64.tar.gz
sudo install kubeseal /usr/local/bin/
```

### Common Operations

```bash
# Fetch the public certificate (for offline sealing)
kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  > pub-cert.pem

# Seal a Secret from stdin
kubectl create secret generic mysecret \
  --dry-run=client -o yaml \
  --from-literal=password=my-value \
  | kubeseal \
    --controller-name=sealed-secrets-controller \
    --controller-namespace=kube-system \
    --format=yaml \
  > mysealedsecret.yaml

# Seal offline (no cluster access needed)
kubectl create secret generic mysecret \
  --dry-run=client -o yaml \
  --from-literal=password=my-value \
  | kubeseal \
    --cert=pub-cert.pem \
    --format=yaml \
  > mysealedsecret.yaml

# Re-encrypt existing SealedSecret with latest key
kubeseal --re-encrypt \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  < mysealedsecret.yaml \
  > mysealedsecret-reencrypted.yaml

# Validate a SealedSecret
kubeseal --validate \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  < mysealedsecret.yaml

# Merge with existing Secret (update individual keys)
# Create new SealedSecret with only the changed key, controller patches it
```

## Controller Configuration

### Key Management

Sealed Secrets controller maintains a key registry of RSA key pairs (4096-bit):

```yaml
controller:
  keyrenewperiod: "30d"    # Generate new key every 30 days
  # keyttl: "10y"          # Certificate validity (default 10 years)
  # keycutofftime: ""      # Force key generation at specific time (RFC1123)
```

Keys are stored as Kubernetes Secrets named `sealed-secrets-key` in the controller namespace, with labels `sealedsecrets.bitnami.com/key=(active|compromised)`.

**Key lifecycle:**
1. Initial install → controller generates first key pair
2. Every `keyrenewperiod` → new key generated, added to registry
3. Old keys retained → existing SealedSecrets remain decryptable
4. `--key-cutoff-time` → force immediate key generation (early renewal)
5. Key never deleted automatically — manual cleanup via `kubeseal --re-encrypt`

### Namespace Scope

```yaml
controller:
  additionalNamespaces: []
```

By default, the controller only watches the namespace it's installed in. `additionalNamespaces` extends watch scope. To watch **all** namespaces, set `global.namespace: ""`.

### Security Context

```yaml
controller:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1001
    runAsGroup: 1001
    allowPrivilegeEscalation: false
    capabilities:
      drop: [ALL]
  podSecurityContext:
    fsGroup: 1001
    seccompProfile:
      type: RuntimeDefault
```

### Logging & Metrics

```yaml
controller:
  logInfoStdout: true
  logLevel: INFO
  logFormat: text        # or json
  metrics:
    enabled: true
    serviceMonitor:
      enabled: false      # Deployed uses pod annotations instead
    podAnnotations:
      prometheus.io/scrape: "true"
      prometheus.io/port: "8080"
      prometheus.io/path: "/metrics"
```

## Encryption Flow

1. **Developer** creates `Secret` YAML locally (dry-run)
2. **kubeseal** encrypts each value in `data`/`stringData` with the controller's public key
3. **SealedSecret** YAML is committed to Git (safe — only public key can encrypt)
4. **Controller** detects SealedSecret → decrypts with private key → creates K8s Secret
5. **Secret** is reconciled — if deleted, controller recreates it from SealedSecret

The encryption scope is bound to:
- **Namespace** (default): SealedSecret only decrypts in the same namespace
- **Cluster-wide**: Use `--scope cluster-wide` with kubeseal to allow any namespace
- **Strict**: Namespace + name must match exactly

## Helm Values

| Value | Default | Description |
|-------|---------|-------------|
| `global.namespace` | — | Controller namespace |
| `controller.image.tag` | latest | Controller image version |
| `controller.keyrenewperiod` | `""` | Key renewal interval |
| `controller.logLevel` | `INFO` | Log level |
| `controller.additionalNamespaces` | `[]` | Extra namespaces to watch |
| `controller.create` | `true` | Create controller deployment |
| `secretName` | `sealed-secrets-key` | Existing TLS secret for key |
| `service.type` | `ClusterIP` | Service type |
| `service.port` | `8080` | Service port |
| `networkPolicy.create` | `false` | Network policy |
| `crds.create` | `true` | Install SealedSecret CRD |
| `rbac.create` | `true` | RBAC resources |

## Common Mistakes

- **Controller not in expected namespace.** `kubeseal` defaults to `kube-system` and controller name `sealed-secrets-controller`. If installed with a different name/namespace, pass `--controller-name` and `--controller-namespace`.
- **SealedSecret in wrong namespace = won't decrypt.** The namespace is embedded in the encryption. A SealedSecret created for `namespace: foo` will NOT decrypt in `namespace: bar`. Re-seal with `--scope cluster-wide` if needed.
- **Key renewal ≠ secret rotation.** Renewing the sealing key only adds a new key pair. Old keys remain valid. This does NOT rotate your actual passwords/API keys. You must rotate secrets separately.
- **Re-encryption doesn't update in-cluster objects.** `kubeseal --re-encrypt` outputs new YAML to stdout. It does NOT patch the SealedSecret in the cluster. Apply the output manually.
- **Compromised key: re-encryption is not enough.** If a sealing key leaked, assume all secrets encrypted with that key are compromised. Rotate actual secrets (passwords, tokens) not just re-encrypt.
- **`skipRecreate: false` (default).** When a managed Secret is deleted, the controller recreates it. Set `skipRecreate: true` if you want to allow manual deletion without auto-recreation.
- **Metrics require Service or Pod annotations.** The deployed config uses `prometheus.io/*` pod annotations for scraping. If using Prometheus Operator, enable `serviceMonitor`.
- **Helm chart version ≠ controller version.** Chart 2.18.4 bundles controller 0.36.0. Always check the `image.tag` value to know which controller version runs.
