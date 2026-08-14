---
name: netbird-cli
description: NetBird client CLI — install, connect (up/down), status, networks, reverse proxy (expose), SSH, Kubernetes access, and debugging. Commands verified against v0.76.x.
---

# netbird-cli — Client

For Terraform provider see `netbird-terraform`. For K8s operator see `netbird-operator`.

## Install

```bash
curl -fsSL https://pkgs.netbird.io/install.sh | sudo bash   # Linux
# or: apt install netbird / dnf install netbird / pacman -S netbird
# GUI on desktop: netbird-ui (Wails v3 app since v0.75.0)
```

Client runs as a daemon (`netbird service run`); the CLI talks to it over a Unix socket. Default config: `/var/lib/netbird/default.json`.

## Register & Connect

```bash
# Interactive SSO (opens browser)
netbird up

# Headless / server / routing peer — setup key from Management dashboard or API
netbird up --setup-key <key>
netbird up --setup-key-file <file>

# SSO without browser (print URL + QR for phone)
netbird login --no-browser
netbird login --qr

# Re-auth without tearing down the tunnel
netbird login --extend
```

**Setup-key peers don't expire** (not subject to login session expiration) — always use setup keys for routing peers/gateways.

## Connect / Disconnect

| Command | Purpose |
|---------|---------|
| `netbird up` | Connect (WireGuard up, connect to management, establish P2P) |
| `netbird up --disable-dns` | Connect without DNS config |
| `netbird up --disable-firewall` | Without firewall rule changes |
| `netbird up --disable-ipv6` | Without IPv6 overlay |
| `netbird up --block-inbound` | No inbound connections (overrides policies) |
| `netbird up --enable-rosenpass` | [Experimental] Post-quantum security |
| `netbird down` | Disconnect everything |
| `netbird deregister` / `netbird logout` | Deregister peer from management (REMOVES from network) |

## Status

```bash
netbird status                  # connection status + peer info
netbird status -d               # detailed human-readable
netbird status --json           # JSON (jq-able)
netbird status -y               # YAML
netbird status -4               # only overlay IPv4 (e.g. 100.64.0.33)
netbird status -6               # only overlay IPv6
netbird status --check live     # health check, exit 0/1 (live|ready|startup)
netbird status -d --filter-by-status connected
netbird status -d --filter-by-connection-type P2P
```

## Networks & Resources

`netbird networks` (alias `routes`) replaces the old routes command — for clientless Networks (subnets/domains behind routing peers).

```bash
netbird networks list                    # list available networks
netbird networks select all              # accept all (incl. new)
netbird networks select <net1> <net2>    # select specific (replace mode)
netbird networks select -a <net3>        # append to selection
netbird networks deselect <net>          # deselect
```

## Reverse Proxy (expose a local service)

```bash
netbird expose 8080                                  # HTTP, public URL auto-created
netbird expose --protocol tcp 5432                   # L4 TCP
netbird expose --protocol udp 53                     # L4 UDP
netbird expose --protocol tls --with-custom-domain tls.example.com 4443
netbird expose --with-password safe-pass 8080        # password-protected
netbird expose --with-pin 123456 8080                # 6-digit PIN
netbird expose --with-user-groups devops,Backend 8080 # restrict to SSO groups
netbird expose --with-name-prefix my-app 8080
netbird expose --protocol tcp --with-external-port 5433 5432
```

Protocols: `http`, `https`, `tcp`, `udp`, `tls`. `--with-custom-domain` requires the domain configured on your account. Managed via dashboard/API afterwards (`netbird_route`-independent — it's the reverse proxy subsystem, not routes).

## SSH

Central SSH access control (policies in dashboard, no manual key distribution):

```bash
netbird ssh <peer-name>            # SSH to peer with policy-based access
netbird ssh -u <user> <peer>       # specify username
netbird ssh -i <keyfile> <peer>    # explicit identity
```

## Kubernetes Access (API server proxy)

```bash
netbird kubernetes list             # list clusters exposed via operator API proxy
netbird kubernetes write-kubeconfig <cluster>   # write kubeconfig for remote cluster
```

Works with the operator's `ClusterProxy`/API-server-proxy pattern: your NetBird user is the identity.

## Debug & Diagnostics

| Command | Purpose |
|---------|---------|
| `netbird debug bundle` | Collect full debug bundle (logs, config, status) |
| `netbird debug capture` | Packet capture on the WireGuard interface |
| `netbird debug config` | Dump effective configuration |
| `netbird debug log` | Manage daemon logging |
| `netbird debug trace` | Trace a packet through the firewall |
| `netbird debug for <duration>` | Run debug logs for a duration, bundle |
| `netbird forwarding list` | List forwarding rules |

## Service Management

```bash
netbird service install --disable-networks        # harden: disable network selection
netbird service install --disable-profiles        # harden: disable profile switching
netbird service install --enable-json-socket      # enable HTTP/JSON API socket
netbird service install --enable-capture          # allow packet capture
netbird service status / start / stop / restart / uninstall
netbird service reset-params
```

## Profiles (multi-account)

```bash
netbird profile list                 # show profiles
netbird profile switch <name>        # switch account
```

## Global Flags

| Flag | Purpose |
|------|---------|
| `-k, --setup-key` / `--setup-key-file` | Setup key for registration |
| `-m, --management-url` | Management URL (default `https://api.netbird.io:443`) |
| `--admin-url` | Admin panel URL (default `https://app.netbird.io:443`) |
| `-n, --hostname` | Custom hostname for the device |
| `--preshared-key` | WireGuard PreSharedKey (only same-key peers can communicate) |
| `-l, --log-level` | Log level (default info) |
| `-A, --anonymize` | Anonymize IPs/non-netbird domains in logs/status |
| `-c, --config` | Override profile file location |

## Troubleshooting

- **Peer shows idle/connecting** → `netbird status -d`, check connection type (P2P vs Relayed). Relayed = NAT traversal failed, traffic via relay service (still encrypted).
- **Registration fails with setup key** → key expired or used up (one-off vs reusable). Check dashboard; `netbird up --setup-key` fresh key.
- **Can't reach a resource** → `netbird networks list` — is the network selected? Policy must allow your group. Resource reachable ≠ routing peer itself reachable (needs separate peer-to-peer policy).
- **DNS issues** → `netbird up --disable-dns` to isolate; check `netbird status -d` DNS state.
- **Need support bundle** → `netbird debug bundle` FIRST, then reproduce.
- **v0.75.1 security advisory** — versions 0.5.0–0.75.1 had a local privilege escalation via the unauthenticated daemon socket (GHSA-qcpp-8vwj-hhwr). Upgrade clients; don't expose the daemon socket to untrusted local users.

## Common Mistakes

- **`logout` vs `down`** — `down` just disconnects; `deregister`/`logout` removes the peer from the network (re-registration needed).
- **Interactive `netbird up` on a server** — SSO login expires. Use `--setup-key` for always-on gateways/routing peers.
- **`netbird expose` ≠ route** — expose is the reverse proxy (public internet); routes/networks are for private overlay. Don't mix them up in docs/automation.
- **Forgetting `-n` on networks select** — default is REPLACE mode; `-a` appends. `select all` is the way to accept everything.
- **Editing `/etc/netbird` config directly** — client is daemon-managed; use flags + `service reconfigure`, not hand-edited config.
