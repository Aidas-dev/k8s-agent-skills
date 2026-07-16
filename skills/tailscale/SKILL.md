---
name: tailscale
description: Use when working with Tailscale — mesh VPN, tailnet device discovery, connectivity troubleshooting, subnet routing, or running Tailscale on Talos/Kubernetes. Triggers: tailscale, tailnet, 100.x.y.z IP, ts.net, tailscale status, tailscale ping, MagicDNS.
---

# Tailscale

**HARD RULE: Always run `tailscale status --json` before assuming any IP addresses.** The tailnet is the source of truth — more authoritative than files, config, or code. Device IPs change, nodes join/leave, routes shift. Status tells you what's actually reachable right now.

## Sub-Skills

| Task | Load skill | What it does |
|---|---|---|
| Talos-specific Tailscale integration (system extension, machine config, node IP config, subnet routes) | `tailscale-talos` | ExtensionServiceConfig, certSANs, kubelet.nodeIP, IP forwarding, static pods, cluster cross-connect |
| Tailscale CLI — status, ping, up, down, ip | (inline, no sub-skill needed) | Reference for common diagnostic commands |

## Decision Flow

- Need to find a device's current IP? → run `tailscale status --json` (source of truth, don't guess)
- Need to check if a peer is online / direct vs relay? → run `tailscale status --json`
- Integrating Tailscale into Talos Linux machine config? → `tailscale-talos`
- Configuring tailscale in Kubernetes (operator, subnet router, proxy)? → `tailscale-talos` (K8s section)
- Debugging connectivity between two devices? → `tailscale ping <ip-or-hostname>` then `tailscale status`
- Managing ACLs, tags, keys via API? → use `curl` against Tailscale API directly

## CLI Reference

### `tailscale status` — Source of Truth

```bash
# Human-readable table (default)
tailscale status

# Machine-readable JSON — preferred for automation
tailscale status --json

# Filter to only active peers
tailscale status --active
```

JSON output structure (`tailscale status --json`):

| Field | Description |
|---|---|
| `Self.TailscaleIPs` | IPs of the current machine |
| `Self.DNSName` | MagicDNS FQDN of current machine |
| `Peer` | Object keyed by node key, each entry is a `PeerStatus` |
| `Peer[HOST].TailscaleIPs` | Array of IPs for peer |
| `Peer[HOST].DNSName` | FQDN like `host.tailnet-id.ts.net.` |
| `Peer[HOST].OS` | Operating system string |
| `Peer[HOST].Online` | Boolean — connected to control plane |
| `Peer[HOST].Active` | Boolean — recent traffic (last ~2 min) |
| `Peer[HOST].ExitNode` | Boolean — currently selected exit node |
| `Peer[HOST].ExitNodeOption` | Boolean — can be exit node |
| `Peer[HOST].Relay` | DERP relay region if not direct |
| `Peer[HOST].CurAddr` | Current direct endpoint address |
| `Peer[HOST].Tags` | ACL tags applied to node |
| `Peer[HOST].PrimaryRoutes` | Subnet routes this node advertises |

```bash
# Extract specific device IP
tailscale status --json | jq -r '.Peer[] | select(.HostName == "myserver") | .TailscaleIPs[0]'

# Map all hostnames to IPs
tailscale status --json | jq -r '[.Self] + [.Peer[]] | sort_by(.DNSName) | map({(.DNSName | split(".")[0]): (.TailscaleIPs[0])}) | add'
```

### `tailscale ip` — Device IP

```bash
tailscale ip          # Both IPv4 + IPv6
tailscale ip -4       # Only Tailscale IPv4
tailscale ip -6       # Only Tailscale IPv6
tailscale ip <hostname>  # Get specific peer's IP
```

### `tailscale ping` — Connectivity Test

```bash
tailscale ping 100.x.y.z          # Ping by Tailscale IP
tailscale ping hostname           # Ping by MagicDNS name
tailscale ping --c 5 --until-direct 100.x.y.z  # Wait for direct connection
```

If ping returns relayed, run `tailscale status` to check if a direct path exists. `--until-direct` waits up to 5s for a direct path to establish.

### `tailscale up` — Join/Auth

```bash
tailscale up                               # Authenticate interactively
tailscale up --authkey tskey-auth-xxx      # Auth key for automation
tailscale up --accept-routes               # Accept subnet routes
tailscale up --advertise-routes 10.0.0.0/16  # Advertise subnet
tailscale up --advertise-exit-node         # Act as exit node
tailscale up --hostname mynode             # Custom tailnet hostname
tailscale up --reset                       # Reset state, re-auth
```

### Other Useful Commands

```bash
tailscale down                                     # Disconnect from tailnet
tailscale logout                                   # Log out, invalidate key
tailscale switch --list                            # List available profiles
tailscale switch <profile>                         # Switch Tailscale profile
tailscale serve                                    # Expose local service via tailnet
tailscale funnel                                   # Expose to internet via tailnet
tailscale netcheck                                 # NAT/direct connection diagnostics
tailscale version                                  # Show version
```
