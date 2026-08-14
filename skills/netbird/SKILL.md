---
name: netbird
description: Use when working with NetBird — route to the correct sub-skill based on what you need: client CLI, Terraform provider, or the Kubernetes operator.
---

# NetBird — Skill Router

**Latest client:** v0.76.x (verified against local binary)  
**Latest Terraform provider:** v0.0.9 (netbirdio/netbird)  
**Latest operator chart:** v0.8.0 (ghcr.io/netbirdio/helm-charts/netbird-operator)  
**Latest server chart:** v1.0.x (helmforge/netbird — repo.helmforge.dev)

## Overview

NetBird is an open-source, WireGuard-based zero-trust mesh VPN. Peers connect peer-to-peer (with ICE/STUN NAT traversal and a relay fallback) into an encrypted overlay network. A central Management service holds network state, distributes peer keys, and enforces access policies. Four surfaces for working with it:

- **Server (control plane)** — self-hosted `netbird-server` (management+signal+relay+STUN+embedded Dex IdP) + dashboard, deployed via the helmforge/netbird chart
- **CLI** (`netbird`) — the client agent: connect/disconnect, status, networks, expose, ssh, k8s
- **Terraform provider** (`netbirdio/netbird`) — declarative management of groups, peers, routes, networks, policies, setup keys, DNS, posture checks via the public API
- **Kubernetes operator** (`netbirdio/kubernetes-operator`) — CRDs that expose in-cluster services and manage peers/routes/groups from inside K8s (consumes an existing control plane)

## Which Sub-Skill?

**Match the user's request against the trigger keywords below, then call `skill(name="<matched-skill>")` to load the sub-skill's full content.**

### Routing Table

| Trigger keywords | User wants to... | Skill to load |
|---|---|---|
| server, control plane, self-hosted, helmforge, netbird-server, management, signal, relay, STUN, dashboard, config.yaml, embedded IdP, Dex, deploy netbird, install netbird | Deploy/operate the NetBird control plane itself (management + signal + relay + dashboard) | `netbird-server` |
| netbird up, netbird down, netbird status, netbird login, setup-key, setup key, peer, expose, reverse proxy, ssh, kubernetes, networks select, FQDN, overlay, WireGuard client, client install | Run the netbird client on a machine (connect/register/troubleshoot a peer) | `netbird-cli` |
| terraform, provider, netbird_group, netbird_peer, netbird_route, netbird_network, netbird_policy, netbird_setup_key, netbird_dns, netbird_posture_check, IaC, infra as code, plan, apply | Manage NetBird account resources as infrastructure code | `netbird-terraform` |
| operator, CRD, SetupKey, NetworkRouter, NetworkResource, NetworkEgress, SidecarProfile, ClusterProxy, Helm chart, in-cluster, expose service, kubeconfig, admission webhook | Run NetBird inside Kubernetes / expose cluster services via CRDs | `netbird-operator` |

### Decision Tree

If routing is ambiguous:

1. **User is deploying/operating the control plane** (chart, config.yaml, DB, IdP, gRPC routing) → `netbird-server`
2. **User is running client commands** (up, status, networks, expose) → `netbird-cli`
3. **User is writing Terraform** (`.tf` files, `netbird_*` resources) → `netbird-terraform`
4. **User is writing Kubernetes manifests** (CRDs, Helm) → `netbird-operator`
5. **Combination?** → Load the one matching the primary action. Common pairs: `netbird-server` deploys the API → `netbird-terraform` manages it (management_url at the in-cluster API) → `netbird-cli` registers peers with setup keys; the `netbird-operator` connects to the server via `managementURL` + PAT.

### When NOT to load this router

- User is working on **Tailscale** — that's a different mesh VPN (`tailscale` skill)
- User asks about **WireGuard config syntax** — NetBird manages WireGuard automatically; raw WireGuard belongs in a general networking skill
- User is setting up **netbird on Talos nodes** — combine `netbird-cli` (agent install) with `tailscale-talos`-style machine config knowledge, but the mesh itself is NetBird
- User just needs a **quick version/overview** — the Overview above is sufficient
