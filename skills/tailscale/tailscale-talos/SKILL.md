---
name: tailscale-talos
description: Tailscale integration with Talos Linux — system extension, machine config, node IP binding, subnet routing, cross-cluster connectivity. Use when deploying or troubleshooting Tailscale on Talos nodes.
---

# Tailscale on Talos Linux

Talos has no SSH/shell — Tailscale runs as a **system extension** at the OS level, not as a user-space daemon. This gives the node a persistent Tailscale interface independent of Kubernetes.

## Architecture

```
Talos Node
  └── OS Kernel with Tailscale extension (siderolabs/tailscale)
        └── ExtensionServiceConfig (env vars: TS_AUTHKEY, TS_ROUTES, etc.)
              └── tailscale0 interface (100.x.y.z)
                    ├── talosctl management via tailscale IP
                    ├── Kubernetes API via tailnet
                    └── Subnet routes (pod/service CIDRs) advertised to tailnet
```

## Installation

### 1. Add Tailscale System Extension

Built via Talos Factory (https://factory.talos.dev). The schematic must include the `siderolabs/tailscale` extension.

```yaml
# machine config (controlplane.yaml / worker.yaml)
machine:
  install:
    image: factory.talos.dev/installer/<schematic-id>:v1.13.0
    extensions:
      - image: ghcr.io/siderolabs/tailscale:v1.62.0
```

The extension provides the `tailscale` binary at the OS level. No `tailscale up` needed — configuration is via `ExtensionServiceConfig`.

### 2. Configure Tailscale Auth

```yaml
apiVersion: v1alpha1
kind: ExtensionServiceConfig
metadata:
  name: tailscale
spec:
  environment:
    - TS_AUTHKEY=tskey-auth-xxxxxxxxxxxxxxxxxxxx
    - TS_EXTRA_ARGS=--advertise-tags=tag:talos,tag:k8s --accept-dns=false
    - TS_HOSTNAME=talos-cp-1
```

Or via `machine.files` (alternative, works the same):

```yaml
machine:
  files:
    - content: |
        TS_AUTHKEY=tskey-auth-xxxxxxxxxxxxxxxxxxxx
        TS_EXTRA_ARGS=--advertise-tags=tag:talos,tag:k8s --accept-dns=false
        TS_HOSTNAME=talos-cp-1
      path: /var/etc/tailscale/auth.env
      permissions: 0o600
      op: create
```

### 3. Critical: Bind Node IP to Physical Network (not Tailscale)

**Without this, kubelet binds to the Tailscale IP**, etcd peers over tailnet, and the cluster breaks on node reboot.

```yaml
machine:
  kubelet:
    nodeIP:
      validSubnets:
        - 192.168.1.0/24    # Your physical/private subnet — NOT tailscale (100.x.y.z)
  cluster:
    etcd:
      advertisedSubnets:
        - 192.168.1.0/24    # Same physical subnet
```

### 4. Add Tailscale IPs to cert SANs

Required for `talosctl` to connect via Tailscale IP.

```yaml
machine:
  certSANs:
    - 100.64.0.0/10          # Tailscale IP range
    - talos-cp-1.tailnet-id.ts.net  # MagicDNS name
```

## Subnet Routing

Advertise Kubernetes pod/service CIDRs so tailnet devices can reach cluster resources directly.

```yaml
machine:
  files:
    - content: |
        TS_AUTHKEY=tskey-auth-xxxxxxxxxxxxxxxxxxxx
        TS_ROUTES=10.244.0.0/16,10.96.0.0/12    # Pod CIDR, Service CIDR
        TS_EXTRA_ARGS=--advertise-tags=tag:talos --snat-subnet-routes=false --accept-routes
      path: /var/etc/tailscale/auth.env
      permissions: 0o600
      op: create
```

- `--snat-subnet-routes=false` preserves original client IP (better for audit/logging)
- `--accept-routes` lets this node use routes advertised by other tailnet devices
- Routes must be **approved** in Tailscale admin console unless auto-approvers are configured

### Enable IP Forwarding (required for subnet router)

```yaml
machine:
  sysctls:
    net.ipv4.ip_forward: "1"
    net.ipv6.conf.all.forwarding: "1"
```

## Tailscale in Kubernetes (Operator)

Alternative to OS-level extension. Runs Tailscale as pods inside K8s, managed by Helm.

| Approach | Pros | Cons |
|---|---|---|
| **OS extension** (recommended for nodes) | Persistent across K8s outages, node reachable via talosctl, works before API server | Requires schematic rebuild per Talos version |
| **K8s operator** (recommended for services) | Native K8s integration, Ingress support, auto-cleanup, no OS changes | Dies if K8s control plane down, no node-level access |

### Subnet Router via K8s (operator)

```bash
# Install Tailscale operator
helm repo add tailscale https://pkgs.tailscale.com/helmcharts
helm install tailscale-operator tailscale/tailscale-operator \
  --namespace tailscale \
  --create-namespace \
  --set-string oauth.clientId=<client-id> \
  --set-string oauth.clientSecret=<client-secret>
```

Then create a proxy for cluster services or use `Connector` for subnet routing.

### Proxy Pod for Service Exposure

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tailscale-proxy
spec:
  hostNetwork: true
  containers:
    - name: tailscale
      image: tailscale/tailscale:latest
      env:
        - name: TS_AUTHKEY
          valueFrom:
            secretKeyRef:
              name: tailscale-auth
              key: TS_AUTHKEY
        - name: TS_ROUTES
          value: 10.244.0.0/16,10.96.0.0/12
        - name: TS_KUBE_SECRET
          value: tailscale-proxy-state
      securityContext:
        capabilities:
          add:
            - NET_ADMIN
            - SYS_MODULE
```

## Cross-Node Considerations

### Split-horizon DNS / API Server Endpoint

When nodes connect over tailnet, the API server endpoint must resolve to the Tailscale IP:

```yaml
cluster:
  controlPlane:
    endpoint: https://talos-cp.tailnet-id.ts.net:6443
```

### Minimum 3 Control Plane Nodes

Talos recommends 3 CP nodes for HA. With Tailscale, each CP node needs:
- Unique `TS_HOSTNAME` in ExtensionServiceConfig
- `advertisedSubnets` set to physical subnet (not tailscale)

### Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Node unreachable via talosctl after reboot | certSANs missing Tailscale IP | Add `machine.certSANs` with `100.64.0.0/10` |
| Kubelet binds to 100.x.y.z | `validSubnets` not set | Set `machine.kubelet.nodeIP.validSubnets` to physical subnet |
| etcd peers flapping | etcd advertising over tailnet | Set `cluster.etcd.advertisedSubnets` to physical subnet |
| Subnet routes not working | IP forwarding disabled | Set `net.ipv4.ip_forward=1` sysctl |
| Tailscale auth fails | Auth key expired | Generate new `tskey-auth-*` or use OAuth |
| `talosctl` can't connect via tailscale | Missing certSANs | Add tailscale IP range to `machine.certSANs` |
| Nodes can't reach each other's tailscale IPs | Firewall blocks UDP 41641 | Open UDP 41641 (WireGuard) between nodes |
| OAuth not working | Missing `--accept-dns=false` | Set `TS_EXTRA_ARGS` to include `--accept-dns=false` |
