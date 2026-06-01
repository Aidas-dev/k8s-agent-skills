---
name: external-dns
description: Use when working with ExternalDNS — synchronizing Kubernetes resources with DNS providers (Cloudflare). Covers providers, sources, registry, RBAC, Gateway API integration, Helm values. No CRDs.
---

# ExternalDNS

## Overview

ExternalDNS synchronizes exposed Kubernetes Services, Ingresses, and Gateway API routes with DNS providers. It watches resources via the K8s watch API and creates/updates/deletes DNS records to match.

**No CRDs.** Controlled via CLI flags, sources, and provider-specific config.

**Latest:** chart 1.21.1, app v0.21.0.

## Architecture

```
K8s Sources (Ingress, HTTPRoute, Service...)
  → ExternalDNS watches for changes
    → Resolves to DNS endpoints
      → Creates/updates/deletes DNS records (Cloudflare, Route53, etc.)
        → Registry (TXT records) tracks ownership
```

## Sources

ExternalDNS queries one or more source types for DNS endpoints:

| Source | Flag | Supported |
|--------|------|-----------|
| Ingress | `--source=ingress` | ✅ |
| Service (LoadBalancer) | `--source=service` | ✅ (no NodePort) |
| Gateway HTTPRoute | `--source=gateway-httproute` | ✅ |
| Gateway GRPCRoute | `--source=gateway-grpcroute` | ✅ |
| Gateway TLSRoute | `--source=gateway-tlsroute` | ✅ (v1alpha2) |
| Gateway TCPRoute | `--source=gateway-tcproute` | ✅ (experimental) |
| Gateway UDPRoute | `--source=gateway-udproute` | ✅ (experimental) |
| Istio Gateway | `--source=istio-gateway` | ✅ |
| Istio VirtualService | `--source=istio-virtualservice` | ✅ |
| CRD | `--source=crd` | ✅ (externaldns.k8s.io/v1alpha1) |
| Node | `--source=node` | ✅ |
| Pod | `--source=pod` | ✅ |
| OpenShift Route | `--source=openshift-route` | ✅ |
| Contour HTTPProxy | `--source=contour-httpproxy` | ✅ |
| Traefik Proxy | `--source=traefik-proxy` | ✅ |

**Deployed sources:** `ingress`, `gateway-httproute`.

## Providers

ExternalDNS supports 25+ providers. Deployed: Cloudflare.

### Cloudflare

Auth via API token (env var `CF_API_TOKEN`). Example values:

```yaml
provider:
  name: cloudflare
env:
  - name: CF_API_TOKEN
    valueFrom:
      secretKeyRef:
        name: cloudflare-credentials
        key: api-token
extraArgs:
  - --cloudflare-proxied
```

Cloudflare-specific flags:

| Flag | Description |
|------|-------------|
| `--cloudflare-proxied` | Enable Cloudflare proxy (orange cloud) — CDN, DDoS protection, SSL |
| `--cloudflare-dns-records-per-page=N` | Records per page (default 100, max 5000) |
| `--cloudflare-custom-hostnames` | Enable Cloudflare for SaaS Custom Hostnames |
| `--cloudflare-regional-services` | Restrict HTTPS decryption to specific regions |
| `--cloudflare-region-key` | Region key for regional services |
| `--cloudflare-record-comment` | Add comment to provisioned records (≤100/≤500 chars) |

## Registry & Policy

Controls how ExternalDNS tracks ownership of records:

```yaml
registry: txt          # Use TXT records to track ownership
txtOwnerId: my-cluster # Owner identifier in TXT record
policy: upsert-only    # Only create/update, never delete
```

| Registry | Description |
|----------|-------------|
| `txt` | TXT records with owner ID (prevents overwriting records from other sources) |
| `aws` | AWS Route53 tag-based (provider-specific) |
| `noop` | No ownership tracking |

| Policy | Description |
|--------|-------------|
| `upsert-only` | Create and update only (safe for shared zones) |
| `sync` | Full sync — create, update, delete (can delete external records) |
| `create-only` | Only create, never update or delete |

## Domain & Ownership Filtering

```yaml
domainFilters:
  - example.com        # Only manage records in this zone
excludeDomains: []     # Exclude specific domains
zoneIdFilters: []      # Limit to specific zone IDs
annotationFilter: ""   # Filter resources by annotation
labelFilter: ""        # Filter resources by label
```

## Gateway API Integration

ExternalDNS reads hostnames from Gateway API HTTPRoute/GRPCRoute resources:

```yaml
sources:
  - gateway-httproute
  - gateway-grpcroute
```

RBAC for Gateway sources requires additional permissions:

```yaml
rbac:
  extraRules:
    - apiGroups: ["gateway.networking.k8s.io"]
      resources: ["httproutes", "gateways"]
      verbs: ["get", "watch", "list"]
```

HTTPRoute annotations for per-route overrides:

```yaml
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/cloudflare-proxied: "true"
    external-dns.alpha.kubernetes.io/ttl: "300"
```

## Deployment (Flux HelmRelease)

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
spec:
  chart:
    spec:
      chart: external-dns
      sourceRef:
        kind: HelmRepository
        name: external-dns
      version: 1.21.1
```

### Helm Values

| Value | Default | Description |
|-------|---------|-------------|
| `provider.name` | `aws` | DNS provider (cloudflare, google, aws, azure, etc.) |
| `sources` | `[service, ingress]` | Resources to watch (ingress, gateway-httproute, service, etc.) |
| `domainFilters` | `[]` | Limit to specific DNS zones |
| `policy` | `upsert-only` | Sync policy (upsert-only, sync, create-only) |
| `registry` | `txt` | Ownership registry (txt, aws, noop) |
| `txtOwnerId` | — | Owner identifier for TXT registry |
| `interval` | `1m` | Sync interval |
| `logLevel` | `info` | Log verbosity |
| `rbac.create` | `true` | Create ClusterRole |
| `rbac.extraRules` | `[]` | Additional RBAC rules (e.g. for Gateway API) |
| `nodeSelector` | `{}` | Node selector |
| `tolerations` | `[]` | Pod tolerations |
| `extraArgs` | `[]` | Additional CLI args |

## Common Mistakes

- **Missing RBAC for Gateway sources.** Without `rbac.extraRules`, gateway-httproute source returns no endpoints. Both `httproutes` and `gateways` resources must be listed.
- **`upsert-only` doesn't clean up stale records.** When an Ingress/HTTPRoute is deleted, its DNS record persists. Use `sync` policy or manual cleanup.
- **Cloudflare API token needs specific permissions.** Requires `Zone:DNS:Edit` for the target zone. A token with only `Zone:Read` will fail silently.
- **`--cloudflare-proxied` is a global flag.** To proxy only specific records, omit the global flag and use the `external-dns.alpha.kubernetes.io/cloudflare-proxied: "true"` annotation per resource.
- **Domain filter is a suffix match.** `kubexa.tech` matches `app.kubexa.tech` but NOT `kubexa.tech.app.com`. Add trailing dot if needed.
- **NodePort services not supported** with `source=service`. Only LoadBalancer services are detected.
- **Multiple TXT owner IDs on same zone.** If two ExternalDNS instances manage the same zone with different `txtOwnerId`, they can coexist. Records with unknown owner ID are left untouched by `upsert-only`.
