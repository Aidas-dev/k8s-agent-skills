---
name: helm-ops
description: Helm production operations — CLI usage for development/debug/emergency, release inspection, troubleshooting, OCI registry, hooks, values management, and GitOps integration (Flux HelmRelease, ArgoCD). **For production deploys, prefer GitOps over manual helm CLI.**
---

# Helm — Operations

For chart writing (templates, values, testing) see `helm-chart`.

## ⚠️ Production Reminder

`helm install` / `helm upgrade` is for dev, debug, and emergency only. **Day-2 operations go through GitOps:**

| Tool | Resource |
|------|----------|
| Flux | `HelmRelease` (helm.toolkit.fluxcd.io/v2) |
| ArgoCD | `Application` with Helm source |

See GitOps Integration below.

## Release Inspection & Troubleshooting

```bash
# List releases
helm list -n production                           # namespace
helm list -A                                      # all namespaces
helm list -n production -a                        # include failed/uninstalled
helm list -n production --filter "myapp"          # filter by name
helm list -n production --date --reverse          # newest first

# Inspect release state
helm status myapp -n production                   # resources + notes
helm history myapp -n production                  # revision history
helm get values myapp -n production               # current values
helm get values myapp -n production --revision 2  # specific revision
helm get manifest myapp -n production             # rendered K8s manifests
helm get manifest myapp -n production --revision 2
helm get notes myapp -n production                # install-time output
helm get hooks myapp -n production                # hooks in release
helm get all myapp -n production                  # everything

# Compare revisions
diff <(helm get manifest myapp -n production --revision 2) \
     <(helm get manifest myapp -n production --revision 3)

# Dry-run / debug
helm install myapp ./mychart --dry-run -f values-prod.yaml
helm upgrade myapp ./mychart --dry-run --server-side -f values-prod.yaml  # Helm 4 SSA
helm template myapp ./mychart -f values.yaml > rendered.yaml               # no cluster
helm template myapp ./mychart --show-only templates/deployment.yaml
helm install myapp ./mychart --debug -f values.yaml  # full template debug output

# Inspect remote charts
helm show chart oci://ghcr.io/myorg/mychart --version 0.1.0
helm show values oci://ghcr.io/myorg/mychart --version 0.1.0
helm show readme oci://ghcr.io/myorg/mychart --version 0.1.0
helm show crds oci://ghcr.io/myorg/mychart --version 0.1.0

# Troubleshoot failed release
helm status myapp -n production                      # what's the status
helm history myapp -n production                     # find last good revision
helm get values myapp -n production --revision 5     # what values caused it
helm get manifest myapp -n production --revision 5   # what rendered
helm rollback myapp 4 -n production --cleanup-on-fail
helm lint ./mychart -f values-prod.yaml              # local validation
```

Common statuses: `deployed`, `failed`, `pending-install`, `pending-upgrade`, `pending-rollback`, `superseded`, `uninstalled`.

### Flux Equivalents

```bash
kubectl describe helmrelease myapp -n production
kubectl get helmrelease myapp -n production -o jsonpath='{.status.conditions}'
kubectl get secret -n production -l "name=myapp,owner=helm"  # raw release data
helm status myapp -n production  # works because Flux creates Helm releases
```

## OCI Registry

Charts as OCI artifacts — no chart repo server. GA since Helm v3.12.

```bash
helm registry login ghcr.io -u <user> --password-stdin
helm package ./mychart && helm push mychart-0.1.0.tgz oci://ghcr.io/myorg
helm install myapp oci://ghcr.io/myorg/mychart --version 0.1.0
helm pull oci://ghcr.io/myorg/mychart --version 0.1.0 --untar
```

Supported registries: GHCR, ECR, Docker Hub, Harbor, Gitea, GitLab.

### Chart.yaml with OCI deps

```yaml
dependencies:
  - name: redis
    version: "~17.0.0"
    repository: oci://registry-1.docker.io/bitnamicharts
    condition: redis.enabled
```

Run `helm dependency update` to resolve.

## Values Management

```bash
# Layered values — later files override earlier
helm template release . -f values.yaml -f values-prod.yaml
```

Layered values merge/override. Array override replaces entirely (use `--set` for array element override).

### Global Values

Flow down to subcharts. Parent globals override subchart globals.

```yaml
global:
  environment: production
  imageRegistry: ghcr.io/myorg
# Accessed in subcharts: .Values.global.environment
```

## Hooks

**Only during Helm-managed installs** — does not work with `helm template | kubectl apply` or GitOps tools.

### Hook Types
`pre-install`, `post-install`, `pre-upgrade`, `post-upgrade`, `pre-rollback`, `post-rollback`, `pre-delete`, `post-delete`, `test`.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-migrate"
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  ttlSecondsAfterFinished: 600
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["/app/bin/migrate"]
```

**Rules:** `restartPolicy: Never`. Always set `hook-delete-policy` (prevents name collision on retry). Weight is string annotation, lower runs first.

**GitOps alternative:** ArgoCD sync-waves, init containers, PostRenderers.

## GitOps Integration (Preferred)

### Flux HelmRelease

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: myapp
  namespace: production
spec:
  interval: 10m
  chart:
    spec:
      chart: mychart
      version: ">=0.1.0 <1.0.0"
      sourceRef:
        kind: HelmRepository
        name: my-charts
        namespace: flux-system
  values:
    replicaCount: 3
    image:
      tag: v2.1.0
    ingress:
      enabled: true
      hosts:
        - host: app.example.com
          paths:
            - path: /
              pathType: Prefix
  upgrade:
    remediation:
      retries: 3
      remediateLastFailure: true
  rollback:
    cleanupOnFail: true
  driftDetection:
    mode: enabled
```

**Key fields:** `chart.spec.version` (SemVer range), `values` (inline, equivalent to `-f`), `upgrade.remediation.retries`, `rollback.cleanupOnFail`, `driftDetection.mode`.

### ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  sources:
    - repoURL: oci://ghcr.io/myorg/charts
      chart: myapp
      targetRevision: 0.1.0
      helm:
        valueFiles:
          - $values/values-prod.yaml
    - repoURL: https://github.com/myorg/config
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### PostRenderer (Last-Mile Patching)

For third-party charts missing values knobs:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
spec:
  chart:
    spec:
      chart: bitnami/nginx
  postRenderers:
    - kustomize:
        patches:
          - target:
              kind: Deployment
            patch:
              - op: add
                path: /spec/template/spec/containers/0/env/-
                value:
                  name: EXTRA_VAR
                  value: "some-value"
```

## Common Ops Mistakes

- **Manual Helm for production.** Use Flux/ArgoCD. CLI is dev/debug/emergency only.
- **`helm template | kubectl apply`** — Hooks don't run. No release tracking. No rollback.
- **`--set` with secrets in CI.** Leaks in logs. Use `-f secrets.yaml` (encrypted via SOPS).
- **Helm 4 flags.** `--atomic` → `--rollback-on-failure`. Check CI scripts after upgrade.
- **Hook Jobs without `hook-delete-policy`.** Previous Job blocks retry with same name.
- **Array override via `--set`.** `--set` replaces entire arrays, use `-f` with YAML for precision.
- **Uninstalling without `--keep-history`.** No way to recover release metadata. Use `-n <ns> --keep-history` if you might need to inspect later.
- **Rollback skips hooks by default.** `helm rollback --recreate-pods` doesn't re-run `pre/post-upgrade` hooks. Use `--recreate-pods` carefully.
