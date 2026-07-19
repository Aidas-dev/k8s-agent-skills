---
name: helm-chart
description: Writing Helm charts — chart anatomy, templates, Go template syntax, helpers, values design, schema validation, library charts, testing, and common chart-writing mistakes.
---

# Helm — Writing Charts

Covers chart structure, Go template syntax, values design, and testing. For operations (deploy, inspect, GitOps) see `helm-ops`.

## Chart Anatomy

```
mychart/
  Chart.yaml          # name, version, appVersion, dependencies
  values.yaml         # Default config (API contract)
  values.schema.json  # JSON Schema validation (strongly recommended)
  .helmignore         # Glob patterns for files to exclude when packaging
  charts/             # Packed subchart deps (helm dependency build)
  crds/               # CRD manifests — installed once, never updated by Helm
  templates/          # Go templates → K8s manifests
    _helpers.tpl      # Named template helpers (underscore = not rendered as manifest)
    deployment.yaml   # One resource per file, name reflects kind
    service.yaml
    ingress.yaml
    hpa.yaml
    NOTES.txt         # Post-install message (template, shown on helm install/status)
    tests/
      test-connection.yaml
  Chart.lock          # Dep lock file (generated)
  ci/                 # CI test values (optional, strongly recommended)
    ci-values.yaml    # Enables all feature toggles for CI rendering
```

**File rules:**
- `.yaml` extension for manifest templates. `.tpl` for helpers that produce no output.
- Dashed notation for file names (`my-app-deployment.yaml`), never camelCase.
- One K8s resource per file. Filename reflects the resource kind.
- Files starting with `_` are not rendered as manifests — only `_helpers.tpl` is conventional.

### Chart.yaml

```yaml
apiVersion: v2           # v1 is legacy, v2 for all Helm 3+ charts
name: myapp
version: 0.1.0           # Chart version — semver, bumps independently of appVersion
appVersion: "1.16.0"     # App image version (informational, not used by Helm)
kubeVersion: ">=1.28.0"  # Optional: minimum K8s version
type: application        # application or library
dependencies:
  - name: redis
    version: ">=17.0.0"
    repository: oci://registry-1.docker.io/bitnamicharts
    condition: redis.enabled
    alias: cache
    tags:
      - cache
```

**Rules:**
- Chart name: DNS-1123 — lowercase, hyphens, no underscores/dots, ≤63 chars, start/end with letter or number.
- `version` tracks chart changes (templates, defaults, structure). `appVersion` tracks the app. They evolve independently. Never couple them.
- `kubeVersion` catches clusters too old for your API version requirements.

### NOTES.txt

Post-install message. Brief, point to README for detail.

```gotmpl
Thank you for installing {{ .Chart.Name }} v{{ .Chart.Version }}.

1. Get the application URL:
   {{- if .Values.ingress.enabled }}
   http://{{ (index .Values.ingress.hosts 0).host }}
   {{- else }}
   kubectl port-forward svc/{{ include "myapp.fullname" . }} 8080:{{ .Values.service.port }}
   {{- end }}

2. Watch pods:
   kubectl get pods -n {{ .Release.Namespace }} -l {{ include "myapp.selectorLabels" . }}
```

## values.yaml — API Contract

Every field gets a `--` comment for `helm-docs` auto-generation. Defaults must be **safe for production**.

```yaml
# -- Number of replicas
replicaCount: 1

image:
  # -- Container image repository
  repository: nginx
  # -- Image tag. Defaults to chart appVersion.
  tag: ""
  # -- Image pull policy
  pullPolicy: IfNotPresent

# -- Image pull secrets for private registries
imagePullSecrets: []
# -- Override the chart name used in resource names
nameOverride: ""
# -- Fully override the resource name
fullnameOverride: ""

serviceAccount:
  # -- Create a service account
  create: true
  # -- Annotations for the service account (e.g. IRSA)
  annotations: {}
  # -- Service account name. Auto-generated if not set.
  name: ""

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 2000
  seccompProfile:
    type: RuntimeDefault

containerSecurityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: [ALL]

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts: []

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

# -- Extra environment variables as key-value pairs
extraEnv: {}

nodeSelector: {}
tolerations: []
affinity: {}
```

### Design Principles

| Principle | Why |
|-----------|-----|
| **camelCase first-lower** | `replicaCount`, not `replica_count` or `ReplicaCount`. Uppercase collides with Go built-ins. Hyphens break `--set`. |
| **Nested over flat** | `image.repository`, not `imageRepository`. Easier to override with `--set`, groups related config. |
| **Map over array** | `--set ingress.hosts[0].host=...` is fragile (array index dependent). Prefer map structures. |
| **Quote all strings** | Prevents YAML coercion (`yes` → boolean `true`, `012` → octal `10`). |
| **`--` comments** | `helm-docs` auto-generates README tables from `# --` prefixed comments. |
| **Safe defaults** | Non-root, resource limits, probes, no `latest`. Dev overrides loosen constraints. |
| **No secrets** | Reference ESO/Vault/Sealed Secrets. Never inline. |
| **No `latest`** | Default `image.tag` to `.Chart.AppVersion`. Pinned versions for reproducible deploys. |

### Values Schema (values.schema.json)

Catches type errors before template rendering. Underused but critical.

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "replicaCount": { "type": "integer", "minimum": 1 },
    "image": {
      "type": "object",
      "properties": {
        "repository": { "type": "string" },
        "tag": { "type": "string" },
        "pullPolicy": { "enum": ["Always", "IfNotPresent", "Never"] }
      },
      "required": ["repository"]
    },
    "ingress": {
      "type": "object",
      "properties": {
        "enabled": { "type": "boolean" }
      }
    }
  },
  "required": ["replicaCount", "image"]
}
```

Also validates subchart values. Skip with `--skip-schema-validation` (air-gapped).

## Writing Templates

### Syntax Basics

| Syntax | Purpose |
|--------|---------|
| `{{ .Values.replicaCount }}` | Render a value |
| `{{- .Values.name \| upper }}` | Strip whitespace left |
| `{{ .Values.name -}}` | Strip whitespace right |
| `{{ .Values.name \| default "fallback" }}` | Default when empty |
| `{{ .Values.name \| quote }}` | Quote string |
| `{{ "value" \| nindent 8 }}` | Newline + 8-space indent |

### include vs template

**Always use `include`, never `template`.** `template` inserts inline and cannot be piped. `include` returns a string so you can control indentation:

```gotmpl
{{- include "myapp.labels" . | nindent 4 }}    # ✅ correct
{{- template "myapp.labels" . }}               # ❌ wrong — no indentation control
```

### Flow Control

```gotmpl
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}

{{/* `with` scopes `.` to the value — avoids repeated `.Values.foo.bar` */}}
{{- with .Values.containerSecurityContext }}
securityContext:
  allowPrivilegeEscalation: {{ .allowPrivilegeEscalation }}
  readOnlyRootFilesystem: {{ .readOnlyRootFilesystem }}
{{- end }}

{{- range .Values.extraHosts }}
- host: {{ . }}
{{- end }}

{{/* Use `$` to access root scope inside range/with */}}
{{- range $key, $val := .Values.extraEnv }}
- name: {{ $key }}
  value: {{ $val | quote }}
{{- end }}
```

### Variables

```gotmpl
{{- $fullName := include "myapp.fullname" . }}
{{- $labels := include "myapp.labels" . }}
```

Good for caching repeated `include` calls and computed values. Use `$` to access root scope inside `range`/`with`.

### `required` Function

Fail with a clear message when mandatory values are missing:

```gotmpl
image: "{{ required "image.repository is required" .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

Better than silently producing empty fields that fail at apply time.

### `tpl` Function

When a value contains Go template syntax, render it with `tpl`:

```gotmpl
annotations:
  {{- tpl .Values.podAnnotations . | nindent 2 }}
```

### Built-in Objects

Available as `.` in every template:

| Object | Access |
|--------|--------|
| `.Values` | Merged values from all `-f` + `--set` sources |
| `.Release.Name` | Release name |
| `.Release.Namespace` | Target namespace |
| `.Release.Service` | Always `Helm` |
| `.Release.Revision` | Revision number (1, 2, 3...) |
| `.Chart` | All Chart.yaml fields |
| `.Files` | Non-template files in chart |
| `.Files.Get "path"` | Read file content |
| `.Capabilities.KubeVersion` | Cluster K8s version |
| `.Capabilities.APIVersions.Has "x/y"` | Check API availability |
| `.Template.Name` | Template file path |

### Important Sprig Functions

[Docs](http://masterminds.github.io/sprig/)

| Function | Use |
|----------|-----|
| `default "fb" .Val` | Default when empty |
| `required "msg" .Val` | Fail if missing |
| `quote`, `upper`, `lower` | String transforms |
| `trimSuffix "-"`, `trunc 63` | DNS label safety |
| `nindent N` | Newline + N spaces |
| `toYaml .` | Convert to YAML (passthrough blocks) |
| `fromYaml`, `toJson`, `fromJson` | Format conversion |
| `sha256sum` | Hash (config checksum) |
| `contains`, `hasPrefix` | String checks |
| `merge`, `mergeOverwrite` | Dict merge |
| `keys`, `values` | Dict iteration helpers |
| `uniq`, `sortAlpha` | List operations |
| `printf` | Format string |
| `now`, `date` | Time functions |

## _helpers.tpl Patterns

**Always prefix template names with chart name.** Template names are globally scoped — `{{ define "labels" }}` collides silently with any subchart or library.

```gotmpl
{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/* Use app.kubernetes.io/* labels — ArgoCD, Kiali, kubectl use these */}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{ include "myapp.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/* selectorLabels: only name + instance. Never chart/version — K8s rejects selector changes. */}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/* Document each helper with {{/* */}} comment */}}
{{- define "myapp.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "myapp.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- "default" }}
{{- end }}
{{- end }}
```

### Document Helpers

Every `{{ define }}` needs a `{{/* */}}` comment explaining purpose:

```gotmpl
{{/* myapp.name: resource name, truncated to 63 chars for DNS-1123 compliance */}}
```

## Common Template Patterns

### Full Deployment Template

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "myapp.labels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "myapp.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          securityContext:
            {{- toYaml .Values.containerSecurityContext | nindent 12 }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          livenessProbe:
            {{- toYaml .Values.livenessProbe | nindent 12 }}
          readinessProbe:
            {{- toYaml .Values.readinessProbe | nindent 12 }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            {{- range $key, $val := .Values.extraEnv }}
            - name: {{ $key }}
              value: {{ $val | quote }}
            {{- end }}
          envFrom:
            - configMapRef:
                name: {{ include "myapp.fullname" . }}
            - secretRef:
                name: {{ include "myapp.fullname" . }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### Env Vars from Values

```gotmpl
env:
  - name: LOG_LEVEL
    value: {{ .Values.logLevel | default "info" | quote }}
  {{- range $key, $val := .Values.extraEnv }}
  - name: {{ $key }}
    value: {{ $val | quote }}
  {{- end }}
envFrom:
  - configMapRef:
      name: {{ include "myapp.fullname" . }}
  - secretRef:
      name: {{ include "myapp.fullname" . }}  # managed externally by ESO
```

### Volumes

```gotmpl
volumes:
  - name: config
    configMap:
      name: {{ include "myapp.fullname" . }}
  - name: tmp
    emptyDir: {}
  {{- with .Values.extraVolumes }}
  {{- toYaml . | nindent 2 }}
  {{- end }}
```

### ConfigMap Checksum Annotation

```yaml
template:
  metadata:
    annotations:
      checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

Forces pod restart when ConfigMap changes. **Without this, updated config doesn't trigger rollout — a top production incident cause.**

### Image Pull Secrets

```gotmpl
{{- with .Values.imagePullSecrets }}
imagePullSecrets:
  {{- toYaml . | nindent 2 }}
{{- end }}
```

### Deployment Strategy

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

`maxUnavailable: 0` ensures zero-downtime (requires PDB + pod anti-affinity).

### Conditional Resources

```gotmpl
{{- if .Values.serviceAccount.create }}
apiVersion: v1
kind: ServiceAccount
...
{{- end }}
```

### toYaml Passthrough

For user-configurable blocks (resources, nodeSelector, affinity, tolerations, probes):

```gotmpl
resources:
  {{- toYaml .Values.resources | nindent 2 }}
```

### Security Context (container and pod)

```gotmpl
# Pod level
securityContext:
  runAsNonRoot: true
  fsGroup: 65534
  seccompProfile:
    type: RuntimeDefault

# Container level
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: [ALL]
```

## Library Charts

A `type: library` chart provides reusable named templates without installing anything.

```yaml
# Chart.yaml
apiVersion: v2
name: mylib
type: library
version: 0.1.0
```

```gotmpl
{{- define "mylib.deployment" -}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mylib.fullname" . }}
  labels: {{- include "mylib.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels: {{- include "mylib.selectorLabels" . | nindent 6 }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          securityContext: {{- toYaml .Values.containerSecurityContext | nindent 12 }}
          resources: {{- toYaml .Values.resources | nindent 12 }}
{{- end }}
```

App chart: `{{ include "mylib.deployment" . }}`

Require in `Chart.yaml`:

```yaml
dependencies:
  - name: mylib
    version: ">=0.1.0"
    repository: file://../mylib
```

Library charts DRY up boilerplate across all services. Update one place, all benefit.

## Testing

```bash
# Lint structure (does NOT validate K8s schema!)
helm lint ./mychart --values ./mychart/ci/ci-values.yaml

# Render + validate against K8s API (no cluster)
helm template test-release ./mychart --values ci/ci-values.yaml > rendered.yaml
kubeconform rendered.yaml --kubernetes-version 1.31.0 --strict

# Alternative: kubectl dry-run (catches K8s API schema mismatches)
helm template test-release ./mychart | kubectl apply --dry-run=client -f -

# Unit test templates
helm-unittest ./mychart

# CI: official chart-testing
ct lint --charts charts/* --validate-maintainers=false
ct install --charts charts/*
```

**`helm lint` only checks Helm syntax, not K8s API validity.** Always pipe through kubeconform or kubectl dry-run.

### helm test (in-cluster)

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-test"
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  containers:
    - name: curl
      image: curlimages/curl
      command: ["curl", "http://{{ include "myapp.fullname" . }}:{{ .Values.service.port }}"]
  restartPolicy: Never
```

```bash
helm test myapp -n default
```

### Unit test example

```yaml
# tests/deployment_test.yaml
suite: test deployment
templates:
  - deployment.yaml
tests:
  - it: should enforce security context
    asserts:
      - equal:
          path: spec.template.spec.securityContext.runAsNonRoot
          value: true
      - equal:
          path: spec.template.spec.containers[0].securityContext.readOnlyRootFilesystem
          value: true
  - it: should set resource limits
    asserts:
      - isNotNull:
          path: spec.template.spec.containers[0].resources.requests
```

### CI Values File

`ci/ci-values.yaml` — enable every feature toggle for full template coverage:

```yaml
ingress:
  enabled: true
  hosts:
    - host: ci-test.example.com
      paths:
        - path: /
          pathType: Prefix
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
serviceAccount:
  create: true
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456:role/ci
extraEnv:
  FEATURE_FLAG: "true"
```

## Chart Versioning

| Bump | When |
|------|------|
| **Major** | Breaking values schema, removed keys, changed defaults |
| **Minor** | New features, optional values added |
| **Patch** | Bug fixes, template improvements |

Breaking changes need migration notes. Deprecated keys work for ≥1 minor version.

## Common Chart Mistakes

- **`latest` as default tag.** Pin versions. Default to `.Chart.AppVersion`.
- **Secrets in values.yaml.** Base64 is not encryption. Use ESO, Vault, or Sealed Secrets.
- **No values.schema.json.** Catches type errors before template rendering. Underused.
- **ConfigMap change doesn't trigger rollout.** Missing `checksum/config` annotation = stale config. Top cause of production confusion.
- **Hardcoded namespaces in templates.** Use `.Release.Namespace` or let deploy tool inject it.
- **Template names not prefixed.** Globally scoped — `{{ define "labels" }}` collides with subcharts silently.
- **`selectorLabels` contain `helm.sh/chart`.** K8s rejects selector changes. Only `name` + `instance`.
- **Using `template` not `include`.** `template` can't be piped — no indentation control.
- **No resource limits.** Unbounded pods starve other workloads. Default sensible requests.
- **No PDB.** Node drains kill all replicas. Default `minAvailable: 1`.
- **`lookup` in templates.** Returns empty on first install, different on upgrade. Test both paths.
- **Chart version not bumped.** Consumers can't distinguish versions. Automate in CI.
- **Oversized templates.** >150 lines → split into `_helpers.tpl`.
- **Relying on `helm lint` alone.** It doesn't validate K8s API schema. Use kubeconform or `kubectl apply --dry-run`.
- **`-` whitespace stripping errors.** `{{- if ... }}` without proper whitespace mangling creates blank lines in YAML. Test with `helm template --debug`.
