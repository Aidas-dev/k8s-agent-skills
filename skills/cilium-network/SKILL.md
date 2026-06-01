---
name: cilium-network
description: Use when configuring Cilium CNI networking, writing network policies, debugging pod connectivity issues, setting up LB IPAM or L2 announcements for Service LoadBalancer IPs, enabling BGP route advertisement, configuring transparent encryption or Hubble observability, or managing Cilium security features (host firewall, Local Redirect Policy, CiliumCIDRGroup). Not for Gateway API (use cilium-gateway).
---

# Cilium Network

Cilium v1.19.4 — eBPF-based CNI, networking, and security for Kubernetes.

## Overview

Cilium provides pod networking via eBPF, replacing kube-proxy with socket-level load balancing. Security policies enforce L3-L7 rules based on identity (not IP), surviving pod churn. Hubble delivers observability. LB IPAM + L2/BGP announcements expose Services externally without a cloud LB.

## Cilium CRDs (Network Domain)

| CRD | API | Purpose |
|-----|-----|---------|
| `CiliumNetworkPolicy` | `cilium.io/v2` | Namespaced L3-L7 network policy (identity, HTTP/gRPC/Kafka, DNS) |
| `CiliumClusterwideNetworkPolicy` | `cilium.io/v2` | Cluster-scoped L3-L7 network policy |
| `CiliumEndpoint` | `cilium.io/v2` | Per-pod status: identity, labels, policy enforcement state |
| `CiliumEndpointSlice` | `cilium.io/v2` | Groups CiliumEndpoints for large-scale clusters |
| `CiliumIdentity` | `cilium.io/v2` | Security identity (labels → numeric ID) |
| `CiliumNode` | `cilium.io/v2` | Node-level Cilium config (allocated CIDRs, health endpoints) |
| `CiliumCIDRGroup` | `cilium.io/v2alpha1` | Named group of CIDRs for policy `fromCIDRSet`/`toCIDRSet` |
| `CiliumLoadBalancerIPPool` | `cilium.io/v2` | LB IPAM pool — allocate LoadBalancer IPs |
| `CiliumL2AnnouncementPolicy` | `cilium.io/v2alpha1` | L2 ARP/NDP announcement of LoadBalancer IPs (beta) |
| `CiliumLocalRedirectPolicy` | `cilium.io/v2` | Node-local traffic redirect (DNS cache, node-local proxy) |
| `CiliumBGPClusterConfig` | `cilium.io/v2alpha1` | BGP cluster-level config (ASN, listen port, node selector) |
| `CiliumBGPPeerConfig` | `cilium.io/v2alpha1` | BGP peer — transport, auth, timers, AFI/SAFI |
| `CiliumBGPAdvertisement` | `cilium.io/v2alpha1` | Advertise pod CIDRs / service LB IPs via BGP |
| `CiliumBGPNodeConfig` | `cilium.io/v2alpha1` | Per-node BGP status (read-only) |
| `CiliumEnvoyConfig` | `cilium.io/v2` | Envoy proxy config (L7 policy enforcement) |

## Key Concepts

### Routing Modes
- **Overlay (VXLAN/Geneve):** Encapsulation, works on any infra. Default.
- **Native routing:** Uses host routing table. Needs KubeSpan/IPsec/WireGuard for cross-node encryption. Faster.
- **kube-proxy replacement:** `kubeProxyReplacement=true` — eBPF replaces iptables for services. Required for many features.

### Identity-Based Security
Cilium assigns a security identity to every pod based on labels. Policies match on identity, not IP. This survives pod churn.

### Policy Enforcement Points
- L3/L4: eBPF datapath (fast path)
- L7: Envoy proxy (HTTP/gRPC/Kafka inspection)
- `reserved:host` identity for kubelet probes, `reserved:world` for external traffic

## CRD Usage Patterns

### Network Policies
```yaml
# Namespaced: allow ingress from app=frontend to app=backend on port 80
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-frontend
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP

# Cluster-wide: allow kubelet health probes from host
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: allow-kubelet-health-probes
spec:
  endpointSelector: {}
  ingress:
  - fromEntities:
    - host
    - remote-node
```
Key endpoints: `fromEndpoints`, `toEndpoints`, `fromEntities` (host/world/cluster/remote-node/all), `fromCIDR`, `toCIDR`, `toFQDNs` (DNS), `fromCIDRSet` with `cidrGroupRef`.

### L7 HTTP Policy
```yaml
spec:
  endpointSelector:
    matchLabels:
      app: my-api
  ingress:
  - toPorts:
    - ports:
      - port: "8080"
      rules:
        http:
        - method: "GET"
          path: "/public/.*"
```

### LB IPAM + L2 Announcements
```yaml
# 1. IP pool
apiVersion: cilium.io/v2
kind: CiliumLoadBalancerIPPool
metadata:
  name: prod-pool
spec:
  blocks:
  - cidr: "10.0.10.0/24"
  serviceSelector:
    matchLabels:
      lb-site: prod

# 2. L2 announcement policy
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: prod-policy
spec:
  loadBalancerIPs: true
  serviceSelector:
    matchLabels:
      lb-site: prod
  nodeSelector:
    matchLabels:
      role: infra
  interfaces:
  - ^eth[0-9]+
```
**Key:** L2 announcements require `loadBalancerIPs: true` or `externalIPs: true`. Without an explicit service selector, all services match. Without node selector, all nodes are candidates.

### L2 Announcement Lease Tuning
```yaml
l2announcements:
  leaseDuration: 15s    # time before failover
  leaseRenewDeadline: 5s
  leaseRetryPeriod: 2s
```
Client rate limit must be sized: `QPS = #services * (1 / leaseRenewDeadline)`.

### CiliumCIDRGroup
```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumCIDRGroup
metadata:
  name: vpn-cidrs
spec:
  externalCIDRs:
  - "10.48.0.0/24"
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: from-vpn
spec:
  endpointSelector: {}
  ingress:
  - fromCIDRSet:
    - cidrGroupRef: vpn-cidrs
```

### BGP Control Plane
Enable via `bgpControlPlane.enabled=true` in Helm.

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPClusterConfig
metadata:
  name: cilium-bgp
spec:
  nodeSelector:
    matchLabels:
      bgp: enabled
  bgpInstances:
  - name: "65000"
    localASN: 65000
    peers:
    - name: "65001"
      peerAddress: "10.0.0.1"
      peerASN: 65001
      peerConfigRef:
        name: cilium-peer
---
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeerConfig
metadata:
  name: cilium-peer
spec:
  transport:
    peerPort: 179
  timers:
    holdTimeSeconds: 90
    keepAliveTimeSeconds: 30
  families:
  - afi: ipv4
    safi: unicast
---
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPAdvertisement
metadata:
  name: bgp-adverts
spec:
  advertisements:
  - advertisementType: "PodCIDR"
    selector:
      matchLabels:
        bgp: enabled
  - advertisementType: "Service"
    selector:
      matchExpressions:
      - {key: "color", operator: In, values: [blue]}
    service:
      addressType: LoadBalancerIP
```

### Encryption
- **IPsec:** `encryption.type=ipsec` — tunnel or wireguard encryption mode
- **WireGuard:** `encryption.type=wireguard` — per-tunnel, simpler key mgmt
- **Transparent Encryption:** node-to-node encryption without app changes

### Hubble Observability
Enable with `hubble.enabled=true`, `hubble.relay.enabled=true`, `hubble.ui.enabled=true`.
Metrics: `hubble.metrics.enabled: [dns, drop, tcp, flow, http, node:true]`

### Host Firewall
```yaml
hostFirewall:
  enabled: true
```
Protects host network namespace with CiliumClusterwideNetworkPolicy (use `nodeSelector` to target).

## Common Mistakes

- **Forgot `loadBalancerIPs: true` in L2 policy** → nothing announced
- **No client rate limit sizing for L2 announcements** → lease renewal fails at scale
- **CNP `endpointSelector` empty in wrong namespace** → policy applies to nothing
- **Using `protocol: UDP` instead of `protocol: UDP` in port rules** — Cilium uses uppercase `TCP`/`UDP`/`SCTP`
- **`externalTrafficPolicy: Local` with L2 announcements** → traffic drops on nodes without local pods. Use `Cluster` instead.
- **BGP: peer IP unreachable from all nodes** — BGP runs in host network, ensure L2 connectivity to peer
- **BGP `nodeSelector` across all CiliumBGP* CRDs must match** — a node must be selected by cluster config, peer config, AND advertisement
- **Hubble metrics missing desired protocols** — list explicit metrics `[dns, drop, tcp, flow, http]`
