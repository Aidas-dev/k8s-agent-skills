---
name: netbird-server
description: Self-hosted NetBird control plane on Kubernetes — the netbirdio/netbird-server combined management/signal/relay/STUN server + dashboard. Covers the helmforge/netbird chart, config.yaml, PostgreSQL store, embedded Dex IdP, gRPC/HTTP routing, STUN, and ESO secret delivery.
---

# netbird-server — Self-Hosted Control Plane

> **Scope:** This skill documents the **helmforge/netbird chart** (`repo.helmforge.dev`, `helmforge/netbird`) deployment of the self-hosted control plane — the pattern used by the production talos-stack deployment. NetBird also ships an official self-hosted setup (docker-compose with optional Traefik); if you're following that path, the components are the same but the chart-specific values (database subchart, existingSecret keys, GRPCRoute wiring) don't apply. For the docker-compose variant see docs.netbird.io/selfhosted.

For the client agent see `netbird-cli`. For Terraform provider see `netbird-terraform`. For the K8s operator see `netbird-operator` (the operator consumes an EXISTING control plane — it does not deploy one).

## What You're Deploying

The control plane is a **single combined server** (`netbirdio/netbird-server`) running management API, gRPC, signal, relay, metrics, health, and embedded STUN in one container, plus a separate dashboard (`netbirdio/dashboard`).

| Component | Image | Role |
|-----------|-------|------|
| server | `netbirdio/netbird-server` | management + signal + relay + STUN + embedded IdP (Dex) |
| dashboard | `netbirdio/dashboard` | web UI (embedded nginx) |

Configured by a single `config.yaml` (replaces old `management.json` + `relay.env`). All services listen on HTTP inside the cluster; **TLS terminates at your ingress/gateway** — the proxy MUST support HTTP/2 / gRPC (h2c).

## Ports to Expose

| Port | Proto | Purpose |
|------|-------|---------|
| 443 | TCP | dashboard, API, gRPC, signal, relay (via ingress/gateway) |
| 3478 | UDP | STUN (NAT traversal — CANNOT be proxied by HTTP reverse proxies) |

## Install (helmforge/netbird chart)

```bash
helm repo add helmforge https://repo.helmforge.dev
helm upgrade --install netbird helmforge/netbird \
  --namespace netbird --create-namespace
```

Minimal production values:

```yaml
server:
  publicUrl: https://netbird.example.com
  replicaCount: 1          # sqlite requires 1; postgres can scale
  auth:
    issuer: https://netbird.example.com/oauth2   # embedded IdP issuer
  config:
    existingSecret: netbird-config    # full config.yaml from secret
    existingSecretKey: config.yaml
  service:
    annotations:
      external-dns.alpha.kubernetes.io/hostname: netbird.example.com

dashboard:
  replicaCount: 1

database:
  mode: external            # auto | external | sqlite
  external:
    engine: postgres        # postgres | mysql
    host: netbird-db-rw.netbird.svc
    port: 5432
    name: netbird
    username: netbird
    existingSecret: netbird-db
    existingSecretPasswordKey: password   # CNPG uses `password`, chart default is `database-password`

postgresql:
  enabled: false            # disable bundled subchart when database.mode=external
```

### Database modes

| Mode | Use | Notes |
|------|-----|-------|
| `auto` (default) | bundled HelmForge PostgreSQL subchart | production default |
| `external` | your own DB (CNPG, managed PG) | `engine: postgres` or `mysql` |
| `sqlite` | lab/disposable | MUST keep `server.replicaCount: 1` |

Production = PostgreSQL (concurrent access, HA). The DSN lives in the config secret, not in values.

### config.yaml delivery (ESO)

Chart renders a default `config.yaml` from values, OR use `server.config.existingSecret` for the full upstream config. Sensitive parts (authSecret, sessionCookieEncryptionKey, store DSN + encryptionKey, owner email/password, IdP seed) belong in an external secret store:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: netbird-config
  namespace: netbird
spec:
  secretStoreRef: { name: vault, kind: ClusterSecretStore }
  target: { name: netbird-config, creationPolicy: Owner }
  data:
    - secretKey: config.yaml
      remoteRef: { key: kv/apps/netbird/config, property: config.yaml }
    - secretKey: idp-seed
      remoteRef: { key: kv/apps/netbird/config, property: idp-seed }
```

Key config.yaml fields:

```yaml
server:
  listenAddress: ":443"
  exposedAddress: "https://netbird.example.com:443"   # public address peers use
  stunPorts: [3478]                                    # or `stuns:` for external
  authSecret: "..."                                    # required for local relay
  dataDir: "/var/lib/netbird/"
  auth:
    issuer: "https://netbird.example.com/oauth2"
    sessionCookieEncryptionKey: ""      # 16/24/32 bytes or base64
    dashboardRedirectURIs: ["https://app.example.com/nb-auth"]
    cliRedirectURIs: ["http://localhost:53000/"]
    owner: { email: "...", password: "..." }   # initial admin
  store:
    engine: "postgres"                   # sqlite | postgres | mysql
    dsn: "..."                           # from secret
    encryptionKey: ""                    # from secret
```

## Routing (Cilium Gateway / any Gateway API)

**gRPC must be routed via GRPCRoute** (or an h2c backend). Plain HTTPRoute forwarding 404s `GetServerKey` — match by RPC service name (see netbirdio/netbird#5542):

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: netbird-rpc
  namespace: netbird
spec:
  parentRefs:
    - name: cilium-gateway
      kind: Gateway
      namespace: cilium-gateway
      sectionName: https
  hostnames: [netbird.example.com]
  rules:
    - matches:
        - { method: { service: management.ManagementService } }
        - { method: { service: signalexchange.SignalExchange } }
      backendRefs: [{ name: netbird-server, port: 80 }]
```

HTTP paths on the server: `/api`, `/oauth2`, `/relay`, `/ws-proxy`. Dashboard UI is everything else:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: netbird
  namespace: netbird
spec:
  parentRefs:
    - name: cilium-gateway
      kind: Gateway
      namespace: cilium-gateway
      sectionName: https
  hostnames: [netbird.example.com]
  rules:
    - matches:
        - { path: { type: PathPrefix, value: /api } }
        - { path: { type: PathPrefix, value: /oauth2 } }
        - { path: { type: PathPrefix, value: /relay } }
        - { path: { type: PathPrefix, value: /ws-proxy } }
      backendRefs: [{ name: netbird-server, port: 80 }]
    - backendRefs: [{ name: netbird-dashboard, port: 80 }]
```

## Identity (embedded Dex + external IdP)

- Embedded IdP (Dex) is **always enabled** and serves `/oauth2/*` (OIDC discovery at `{issuer}/.well-known/openid-configuration`).
- Default dashboard auth matches the embedded issuer: `clientId: netbird-dashboard`, `audience: netbird-dashboard`, scopes `openid profile email groups`, redirects `/nb-auth` + `/nb-silent-auth`.
- **External IdP** (ZITADEL, Keycloak, Okta): register a web app with PKCE (public client, `auth_method_type NONE`), redirect URI = the Dex connector callback `https://<public-url>/oauth2/callback`. Seed the Dex connector into the embedded IdP at startup via `IDP_SEED_INFO` env (base64 dex.Connector JSON from a secret).
- **Dashboard + server must share one public hostname** when using the embedded provider. With an external OIDC provider, override ALL dashboard auth values (`AUTH_AUTHORITY`, `AUTH_CLIENT_ID`, `AUTH_AUDIENCE`, `AUTH_SUPPORTED_SCOPES`).

## Terraform + Operator Integration

- Manage account topology (groups, peers, routes, networks, policies, setup keys, DNS) via the `netbird-terraform` skill, pointing `management_url` at the in-cluster API (`https://netbird.example.com` or `netbird-server.netbird.svc:80` from inside the cluster).
- The `netbird-operator` (separate chart `netbirdio/helm-charts`) connects to this control plane via `managementURL` + a service-user PAT (`NB_API_KEY`). It does NOT deploy the server.

## Common Mistakes

- **Plain HTTPRoute for gRPC** — `GetServerKey` 404s. Use GRPCRoute matched by RPC service name (management.ManagementService, signalexchange.SignalExchange) or an h2c backend.
- **Missing UDP 3478** — STUN can't be HTTP-proxied. P2P NAT traversal breaks → all peers relay.
- **`existingSecretPasswordKey` mismatch** — CNPG initdb secret uses key `password`; chart default expects `database-password`. Set the override or the server can't reach the DB.
- **Forgetting `postgresql.enabled: false`** — bundled subchart conflicts with `database.mode: external` (chart validate).
- **sqlite with replicaCount > 1** — SQLite store is single-instance; scale only on PostgreSQL.
- **IdP connector type immutability** — after creation the NetBird API rejects type/schema changes on the identity provider. Delete+recreate (import afterwards in Terraform) rather than edit.
- **Dashboard external OIDC partial override** — mixing embedded defaults with an external IdP breaks login. Override all `AUTH_*` values.
- **`disableGeoliteUpdate: false` + air-gapped** — server pod downloads GeoLite DBs at boot; health endpoint can be slow to open. Leave disabled unless the cluster allows egress.
- **STUN deferred on Cilium** — Cilium UDP listener support for the STUN Service may be unproven; verify UDP Service exposure before relying on P2P.
