---
name: cert-manager
description: Use when creating or managing cert-manager CRDs — Certificate, Issuer, ClusterIssuer, CertificateRequest. Covers ACME DNS-01/HTTP-01, self-signed, CA issuers, Gateway API integration, and common patterns.
---

# Cert-Manager CRDs

Chart from `https://charts.jetstack.io` or `oci://quay.io/jetstack/charts/cert-manager`  
Latest chart: **v1.20.2** (Apr 2026) | API: `cert-manager.io/v1`, `acme.cert-manager.io/v1`  
CRDs: Certificate, Issuer, ClusterIssuer, CertificateRequest | ACME: Order, Challenge | Alpha: ListenerSet

Install CRDs: `--set crds.enabled=true` or `installCRDs: true`

## CRD Reference

| CRD | API | Scope | Purpose | Key Fields |
|-----|-----|-------|---------|------------|
| **Issuer** | `cert-manager.io/v1` | Namespaced | Certificate authority within a namespace. ACME, self-signed, CA, Vault, Venafi. | `spec.acme`, `spec.ca`, `spec.selfSigned`, `spec.vault`, `spec.venafi` |
| **ClusterIssuer** | `cert-manager.io/v1` | Cluster | Cluster-wide issuer — can issue certs in any namespace. Same spec as Issuer. | Same as Issuer |
| **Certificate** | `cert-manager.io/v1` | Namespaced | Request a TLS certificate from an issuer. Creates Secret with cert+key. | `spec.secretName`, `spec.issuerRef`, `spec.dnsNames[]`, `spec.commonName`, `spec.duration`, `spec.renewBefore`, `spec.privateKey` (algorithm, size, rotationPolicy), `spec.usages[]`, `spec.subject` |
| **CertificateRequest** | `cert-manager.io/v1` | Namespaced | Low-level CSR resource. Usually created automatically by Certificate. | `spec.issuerRef`, `spec.request` (base64 CSR), `spec.duration`, `spec.isCA` |

### ACME Resources

| CRD | API | Purpose |
|-----|-----|---------|
| **Order** | `acme.cert-manager.io/v1` | ACME order lifecycle. Created automatically by Certificate with ACME issuer. |
| **Challenge** | `acme.cert-manager.io/v1` | ACME challenge (DNS-01, HTTP-01). Created automatically per domain in Order. |

## Issuer Types

| Type | Configuration | Use Case |
|------|--------------|----------|
| **ACME** | `spec.acme.server` + `spec.acme.solvers[]` | Let's Encrypt, ZeroSSL, Buypass |
| **SelfSigned** | `spec.selfSigned: {}` | Internal CA bootstrap, testing |
| **CA** | `spec.ca.secretName` | Sign with existing CA key+cert in Secret |
| **Vault** | `spec.vault` | HashiCorp Vault PKI |
| **Venafi** | `spec.venafi` | Venafi TPP/Cloud |

## Deployed Pattern — Let's Encrypt + Cloudflare DNS-01

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-production-account-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token-secret
              key: api-token
```

## Deployed Pattern — Gateway Wildcard Certificate

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-gateway
  namespace: cilium-gateway
spec:
  secretName: example-tls
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  dnsNames:
    - "*.example.com"
    - "example.com"
  duration: 168h         # 7 days
  renewBefore: 144h      # Renew 6 days before expiry
  privateKey:
    algorithm: ECDSA
    size: 256
    rotationPolicy: Always
  usages:
    - server auth
```

## Deployed Pattern — Self-Signed CA + Issuer

```yaml
# 1. Self-signed root CA
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ca-root
  namespace: cert-manager
spec:
  secretName: ca-root-secret
  isCA: true
  commonName: "cluster-ca"
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned
    kind: ClusterIssuer
---
# 2. CA issuer from root
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: ca-issuer
spec:
  ca:
    secretName: ca-root-secret
```

## Key Config Details

```yaml
# ACME DNS-01 solver options
spec:
  acme:
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:    # API token (not API key)
              name: cloudflare-token
              key: api-token
          # Or route53:
          route53:
            region: us-east-1
            accessKeyIDSecretRef:
              name: aws-creds
              key: access-key-id
            secretAccessKeySecretRef:
              name: aws-creds
              key: secret-access-key
            hostedZoneID: Z123456
          # Or Azure:
          azureDNS:
            subscriptionID: <id>
            resourceGroupName: <rg>
            hostedZoneName: example.com
            tenantID: <tenant>
            clientID: <client>
            clientSecretSecretRef:
              name: azure-creds
              key: client-secret
      - http01:
          ingress:
            class: nginx        # Ingress class for HTTP-01 solver
```

## Gateway API Integration

cert-manager v1.20+ supports Gateway API for ACME HTTP-01 challenges. No `parentRefs` required (v1.20):

```yaml
spec:
  acme:
    solvers:
      - http01:
          gatewayHTTPRoute:
            parentRefs:
              - name: my-gateway
                namespace: istio-system
```

## Important Fields — Certificate

```yaml
spec:
  duration: 2160h                   # 90 days default
  renewBefore: 720h                  # 30 days before expiry
  privateKey:
    algorithm: ECDSA                # ECDSA > RSA for performance
    size: 256                       # P-256 curve
    rotationPolicy: Never           # or "Always" to rekey on renewal
  usages:                           # Extended Key Usage
    - server auth
    - client auth
  subject:
    organizations:
      - MyOrg
  emailAddresses:
    - admin@example.com
  keystores:                        # Create JKS/PKCS12 keystores
    pkcs12:
      create: true
      passwordSecretRef:
        name: keystore-pass
        key: password
```

## Common Mistakes

- **DNS-01 `apiTokenSecretRef` vs `apiKeySecretRef`** — Cloudflare uses API **tokens** (scoped), not API keys. Use `apiTokenSecretRef`.
- **Certificate renewBefore > duration** — cert-manager will constantly renew. Keep `renewBefore < duration`.
- **ClusterIssuer in wrong namespace** — ClusterIssuer is not namespaced. Certificate references it via `issuerRef.kind: ClusterIssuer`.
- **EC key with older clients** — Some clients don't support ECDSA. Use RSA if compatibility needed: `algorithm: RSA`, `size: 2048`.
- **DNS-01 recursive nameservers** — cert-manager needs authoritative DNS. Set `--dns01-recursive-nameservers=8.8.8.8:53,1.1.1.1:53` and `--dns01-recursive-nameservers-only` to avoid public resolver issues.
- **ACME rate limits** — Let's Encrypt has 50 certs/week/domain for production, 5/week for staging. Use staging for testing.
- **Secret overwrite** — cert-manager overwrites the Secret on renewal. Don't manually edit the TLS secret.
- **dnsNames vs commonName** — SANs (`dnsNames`) are required. `commonName` is deprecated in CA/B guidelines.
- **HTTP-01 on port 80** — cert-manager needs port 80 reachable. Behind Cilium Gateway, ensure HTTP listener exists.
