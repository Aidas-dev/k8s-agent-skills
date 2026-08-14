---
name: netbird-operator
description: NetBird Kubernetes operator (netbirdio/kubernetes-operator) — CRDs for exposing cluster services on the NetBird mesh, managing peers/routes/groups from inside K8s. Covers SetupKey, Group, NetworkRouter, NetworkResource, NetworkEgress, SidecarProfile, ClusterProxy.
---

# netbird-operator — Kubernetes Operator

For client CLI see `netbird-cli`. For Terraform provider see `netbird-terraform`.

**Repo:** github.com/netbirdio/kubernetes-operator  
**Chart:** `ghcr.io/netbirdio/helm-charts/netbird-operator` (latest 0.8.0)  
**API group:** `netbird.io/v1alpha1`

Automates NetBird provisioning for services in your cluster: peers, routes, groups declared as CRs. Setup keys and credentials auto-stored/rotated as Kubernetes Secrets. Works with NetBird Cloud and self-hosted.

## Install

```bash
# 1. cert-manager (for admission webhooks) — recommended
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.0/cert-manager.yaml

# 2. Namespace + API secret (NetBird PAT for the Management API)
kubectl create namespace netbird
kubectl -n netbird create secret generic netbird-mgmt-api-key \
  --from-literal=NB_API_KEY=${NB_API_KEY}

# 3. Install operator
helm upgrade --install --create-namespace -n netbird \
  netbird-operator oci://ghcr.io/netbirdio/helm-charts/netbird-operator

# 4. Verify
kubectl get pods -n netbird   # all Running before creating CRs
```

PAT: create a **service user** in the NetBird dashboard and generate a Personal Access Token.

## CRDs (`netbird.io/v1alpha1`)

| Kind | Purpose |
|------|---------|
| `SetupKey` | Declarative setup key (auto-groups, expiry, ephemeral, extra DNS labels) |
| `Group` | Group (by name, id, or localRef to another Group CR) |
| `NetworkRouter` | Expose a ClusterIP Service via a shared routing peer + DNS zone |
| `NetworkResource` | Expose an address/domain as a Network resource (networkRouterRef) |
| `NetworkEgress` | Give a pod's network egress onto the NetBird overlay |
| `SidecarProfile` | Per-pod identity: inject netbird sidecar so the pod becomes its own peer |
| `ClusterProxy` | Proxy the Kubernetes API server for `netbird kubernetes` + `kubectl` |

## Patterns (choose by what you're exposing)

| Pattern | Exposes | Identity | Reach it via | Best for |
|---------|---------|----------|--------------|----------|
| `NetworkRouter` | a ClusterIP Service | shared routing peer | `service.namespace.<zone>` DNS | stable internal services (DBs, APIs) many peers reach |
| `SidecarProfile` (client sidecar) | the pod itself | pod = own peer | pod's overlay IP | workloads needing own identity / outbound overlay access |
| `ClusterProxy` (API server proxy) | the Kubernetes API | your NetBird user | `netbird kubernetes` + `kubectl` | operating remote clusters |
| Gateway API (Beta) | Services via Gateway CRDs | gateway routing peer | route hostname / overlay | teams standardizing on Gateway API |

**Rule of thumb:** routing peer (NetworkRouter) for "peers reach my service" — one peer fronts many services. Sidecar for "this pod initiates traffic onto the overlay / needs per-pod identity" (cost: one peer per pod). A routing peer exposes services TO your peers; it does NOT give other pods a path OUT onto the overlay — that needs a sidecar.

## SetupKey Example

```yaml
apiVersion: netbird.io/v1alpha1
kind: SetupKey
metadata:
  name: my-setup-key
  namespace: netbird
spec:
  name: my-setup-key          # key name in NetBird
  ephemeral: false
  allowExtraDnsLabels: false
  duration: 24h               # validity (pattern: 1h, 30m, 24h…)
  autoGroups:                 # auto-assign peers using this key
    - name: my-group          # localRef to a Group CR in same namespace
```

The operator stores the generated setup key in a **Secret** (status tracks it) for consumption by workloads.

## NetworkRouter Example (expose a Service)

```yaml
apiVersion: netbird.io/v1alpha1
kind: NetworkRouter
metadata:
  name: my-db-router
  namespace: netbird
spec:
  networkRouterRef:            # the NetBird network + router to attach to
    name: my-network
    namespace: netbird
  serviceRef:                  # ClusterIP Service to expose
    name: my-db
  groups:                      # who can reach it (NetBird groups)
    - name: my-group
  # zone: <DNSZoneReference>   # optional DNS zone by domain name
```

Result: peers in `my-group` reach the service at `my-db.<namespace>.<zone>` DNS name through the shared routing peer.

## NetworkResource Example

```yaml
apiVersion: netbird.io/v1alpha1
kind: NetworkResource
metadata:
  name: my-resource
  namespace: netbird
spec:
  networkRouterRef:
    name: my-network
    namespace: netbird
  serviceRef:                  # or a raw address
    name: my-service
  groups:
    - name: my-group
  # address: 10.0.0.5          # single IP alternative
```

## NetworkEgress (pod egress onto overlay)

Gives the pod's network a path out onto the NetBird network (opposite direction of NetworkRouter). Uses `NetworkResourceSpec`-style address/destination config.

## ClusterProxy (kubectl access to remote clusters)

```yaml
apiVersion: netbird.io/v1alpha1
kind: ClusterProxy
metadata:
  name: my-cluster
  namespace: netbird
spec:
  clusterName: my-cluster                      # required
  apiServer: https://kubernetes.default.svc.cluster.local/  # required
  serviceAccountName: my-impersonation-sa      # required (impersonation)
  groups:                                      # NetBird groups allowed to access
    - name: admins
```

User side:

```bash
netbird kubernetes list
netbird kubernetes write-kubeconfig my-cluster
kubectl get nodes   # through the NetBird tunnel
```

## Operations

```bash
kubectl get setupkeys,groups,networkrouters,networkresources,networkegresses,sidecarprofiles,clusterproxies -A
kubectl get secret -n netbird          # setup keys + API key secrets
kubectl describe networkrouter my-db-router   # status + conditions
```

Namespace-scoped or cluster-wide deployment; per-namespace for multi-tenant clusters.

## Common Mistakes

- **Two sources of truth for the same key** — `SetupKey` CRD vs Terraform `netbird_setup_key`: pick ONE per key or they drift.
- **Routing peer ≠ pod egress** — NetworkRouter exposes services to peers; it does NOT let pods reach the overlay. Pods need `SidecarProfile`/`NetworkEgress`.
- **NetworkRouter reach vs routing-peer-host reach** — the resource policy reaches services BEHIND the routing peer; reaching services ON the routing peer host needs a separate peer-to-peer policy (input chain).
- **Forgetting cert-manager** — webhook-backed CRDs fail without it; `helm install` looks fine but CR validation/admission breaks.
- **`netbird-mgmt-api-key` secret name mismatch** — the chart expects this exact secret name + `NB_API_KEY` key. Change either and the operator can't authenticate.
- **Service user PAT, not personal PAT** — operator auth uses service-user tokens; personal tokens may lack account-level permissions.
- **Gateway API still Beta** — prefer NetworkRouter/SidecarProfile for anything you depend on today.
