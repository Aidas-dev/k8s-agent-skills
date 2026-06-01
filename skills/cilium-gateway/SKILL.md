---
name: cilium-gateway
description: Use when creating Gateway API resources for ingress, configuring TLS termination or passthrough, setting up HTTP-to-HTTPS redirect or traffic splitting, integrating oauth2-proxy or ExternalDNS with Cilium, or debugging Cilium Gateway controller issues including the Programmed=False cosmetic bug.
---

# Cilium Gateway

Cilium v1.19.4 — Gateway API implementation for ingress traffic via per-node Envoy.

## Overview

Cilium implements Gateway API v1.4.1 using per-node Envoy proxies with eBPF TPROXY interception. Supports GatewayClass, Gateway, HTTPRoute, GRPCRoute, TLSRoute, and ReferenceGrant. Host network mode exposes listeners directly on node IPs without a LoadBalancer Service. TLS termination, traffic splitting, and header modification all handled in Envoy.

## CRDs Used

### Gateway API (standard)
| CRD | Version | Purpose |
|-----|---------|---------|
| `GatewayClass` | `gateway.networking.k8s.io/v1` | Class reference (parametersRef → `CiliumGatewayClassConfig`) |
| `Gateway` | `gateway.networking.k8s.io/v1` | Shared LB listener — hostname, TLS, ports |
| `HTTPRoute` | `gateway.networking.k8s.io/v1` | HTTP route rules — matches, filters, backends |
| `GRPCRoute` | `gateway.networking.k8s.io/v1` | gRPC route rules |
| `TLSRoute` | `gateway.networking.k8s.io/v1alpha2` | TLS passthrough routing by SNI (experimental) |
| `ReferenceGrant` | `gateway.networking.k8s.io/v1beta1` | Allow cross-namespace references (Secret, Service) |

### Cilium-specific
| CRD | Version | Purpose |
|-----|---------|---------|
| `CiliumGatewayClassConfig` | `cilium.io/v2alpha1` | Cilium-specific GatewayClass parameters (envoy config, LB type, etc.) |
| `CiliumEnvoyConfig` | `cilium.io/v2` | Low-level Envoy config (used internally by Gateway controller) |

## Architecture

```
Internet → LB IP → any node → eBPF TPROXY → per-node Envoy → identity "ingress" → backend pod
```

- Traffic arrives at any node, eBPF intercepts via TPROXY using `ingress` identity
- Per-node Envoy (DaemonSet or cilium-agent embedded) handles L7
- Two policy enforcement points: `world → ingress` and `ingress → backend`
- Source IP preserved in `X-Forwarded-For` and `X-Envoy-External-Address` headers

## Prerequisites

```yaml
# Helm values needed
kubeProxyReplacement: true
gatewayAPI:
  enabled: true
```

Gateway API v1.4.1 CRDs must be pre-installed:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_*.yaml
```

## Host Network Mode

Cilium 1.16+ — expose Gateway directly on host network (no LoadBalancer Service). **Requires Envoy hostNetwork.**

```yaml
gatewayAPI:
  enabled: true
  hostNetwork:
    enabled: true
    nodes:
      matchLabels:
        role: infra
envoy:
  enabled: true
  hostNetwork: true
  securityContext:
    capabilities:
      keepCapNetBindService: true
      envoy:
        - NET_BIND_SERVICE
```

**Privileged ports (≤1023):** Add `NET_BIND_SERVICE` capability to Envoy.

## Common Patterns

### Gateway with TLS Termination
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cilium-gateway
  namespace: cilium-gateway
  annotations:
    external-dns.alpha.kubernetes.io/target: 79.76.124.104
spec:
  gatewayClassName: cilium
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: "*.kubexa.tech"
    allowedRoutes:
      namespaces:
        from: All
  - name: https
    protocol: HTTPS
    port: 443
    hostname: "*.kubexa.tech"
    tls:
      mode: Terminate
      certificateRefs:
      - name: kubexa-tech-tls
        kind: Secret
  allowedRoutes:
    namespaces:
      from: All
```

### HTTP → HTTPS Redirect
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: http-to-https
  namespace: cilium-gateway
spec:
  parentRefs:
  - name: cilium-gateway
    sectionName: http
  hostnames:
  - "*.kubexa.tech"
  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        statusCode: 301
```

### HTTPRoute with Backend
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
  namespace: myapp
spec:
  parentRefs:
  - name: cilium-gateway
    namespace: cilium-gateway
    sectionName: https
  hostnames:
  - app.kubexa.tech
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: app-service
      port: 8080
```

### Cross-Namespace Reference (ReferenceGrant)
```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-httproutes
  namespace: cilium-gateway
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: kube-system
  to:
  - group: ""
    kind: Secret
    name: kubexa-tech-tls
```
Needed when HTTPRoute is in a different namespace than the Gateway or the Secret.

### oauth2-proxy Integration
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: hubble-ui
  namespace: kube-system
spec:
  parentRefs:
  - name: cilium-gateway
    namespace: cilium-gateway
    sectionName: https
  hostnames:
  - hubble.kubexa.tech
  rules:
  - backendRefs:
    - name: oauth2-proxy-hubble
      port: 4180
```

### Traffic Splitting
```yaml
spec:
  rules:
  - backendRefs:
    - name: app-v1
      port: 80
      weight: 90
    - name: app-v2
      port: 80
      weight: 10
```

### Header Modification
```yaml
spec:
  rules:
  - filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        set:
        - name: X-Custom-Header
          value: my-value
        add:
        - name: X-Trace-Id
          value: "abc123"
```

### LB IPAM Integration with Gateway Addresses
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  addresses:
  - type: IPAddress
    value: 172.18.0.140
  gatewayClassName: cilium
  listeners:
  - ...
```
Or via annotation (deprecated): `io.cilium/lb-ipam-ips: "172.18.0.141"`

## Known Issues

### Programmed=False (Cosmetic)
Cilium Gateway may show `PROGRAMMED=False` status even though routes work fine.
- Gateway status: "Address not ready yet" / Programmed: False
- HTTPRoute status: Accepted: True, ResolvedRefs: True
- Traffic flows despite Programmed=False
- Do not treat this as a failure — routes are functional

### TLS Passthrough Source IP
When using TLS passthrough, backends see Envoy IP (node IP) as source, not the client IP. This is inherent to TCP proxy mode.

## Troubleshooting

```bash
# Check gateway status
kubectl get gateway -A
kubectl describe gateway <name>

# Check HTTPRoute
kubectl describe httproute <name>

# Check operator logs for Gateway API errors
kubectl logs -n kube-system deployments/cilium-operator | grep gateway

# Check Envoy config
kubectl get ciliumenvoyconfigs -A

# Verify Gateway API CRDs installed
kubectl get crd | grep gateway.networking.k8s.io
```

### Common Errors
| Symptom | Cause | Fix |
|---------|-------|-----|
| GatewayClass not found | Gateway API CRDs not installed | Install v1.4.1 CRDs |
| Secret "X" not found | Missing ReferenceGrant for cross-namespace Secret | Add ReferenceGrant in Secret's namespace |
| BackendNotFound | Service doesn't exist or wrong namespace | Check `backendRefs` names |
| Programmed=False | Cosmetic bug (see above) | Verify HTTPRoute shows Accepted/ResolvedRefs |
| HostNetwork port clash | Port already in use | Use unique ports per Gateway resource |
