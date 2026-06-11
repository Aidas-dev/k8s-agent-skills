# KServe — Helm Deployment

**Repo:** `https://kserve.github.io/helm-charts`  
**Charts:** 10 (see table below)  
**Latest:** v0.18.0  

## Charts (10)

| Chart | Namespace | Purpose |
|-------|-----------|---------|
| `kserve-crd` | `kserve` | Core CRDs only |
| `kserve-resources` | `kserve` | Controller + webhook + storage init |
| `kserve-llmisvc-crd` | `kserve` | LLMInferenceService CRD only |
| `kserve-llmisvc-resources` | `kserve` | LLMInferenceService controller |
| `kserve-localmodel-crd` | `kserve` | LocalModelCache + LocalModelNode CRDs |
| `kserve-localmodel-resources` | `kserve` | LocalModel controller + node agent |
| `kserve-runtime-configs` | `kserve` | Default ServingRuntime configs |
| `kserve-crd-minimal` | `kserve` | Min CRDs (InferenceService only) |
| `kserve-resources-minimal` | `kserve` | Min controller (no Mesh/Knative) |
| `kserve-runtime-configs-minimal` | `kserve` | Minimal runtime configs |

## Prerequisites

| Dependency | Version | Required For |
|-----------|---------|-------------|
| Kubernetes | ≥ 1.26 | All |
| Cert-Manager | ≥ 1.12 | Controller webhook |
| Istio | ≥ 1.20 | Standard mode (with ingress gateway) |
| Knative Serving | ≥ 1.16 | Serverless mode (optional, auto-scaling to zero) |
| Gateway API | ≥ 1.2 | Gateway API ingress (alternative to Istio) |

## Quick Install

### Standard Mode (Knative + Istio)

```bash
# 1. Install Knative (if needed)
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.16.0/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.16.0/serving-core.yaml
kubectl apply -f https://github.com/knative/net-istio/releases/download/knative-v1.16.0/net-istio.yaml

# 2. Install KServe
helm repo add kserve https://kserve.github.io/helm-charts
helm repo update

# CRDs
helm install kserve-crd oci://ghcr.io/kserve/charts/kserve-crd \
  --version v0.18.0 --namespace kserve --create-namespace

# Controllers
helm install kserve-resources oci://ghcr.io/kserve/charts/kserve-resources \
  --version v0.18.0 --namespace kserve

# Runtime configs
helm install kserve-runtime-configs oci://ghcr.io/kserve/charts/kserve-runtime-configs \
  --version v0.18.0 --namespace kserve
```

### Raw Deployment Mode (No Mesh)

```bash
# Minimal CRDs (InferenceService only)
helm install kserve-crd oci://ghcr.io/kserve/charts/kserve-crd-minimal \
  --version v0.18.0 --namespace kserve --create-namespace

# Minimal controller
helm install kserve-resources oci://ghcr.io/kserve/charts/kserve-resources-minimal \
  --version v0.18.0 --namespace kserve

# Minimal runtime configs
helm install kserve-runtime-configs oci://ghcr.io/kserve/charts/kserve-runtime-configs-minimal \
  --version v0.18.0 --namespace kserve
```

### LLM Inference Service (Optional)

```bash
helm install kserve-llmisvc-crd oci://ghcr.io/kserve/charts/kserve-llmisvc-crd \
  --version v0.18.0 --namespace kserve

helm install kserve-llmisvc-resources oci://ghcr.io/kserve/charts/kserve-llmisvc-resources \
  --version v0.18.0 --namespace kserve
```

### LocalModel (Optional)

```bash
helm install kserve-localmodel-crd oci://ghcr.io/kserve/charts/kserve-localmodel-crd \
  --version v0.18.0 --namespace kserve

helm install kserve-localmodel-resources oci://ghcr.io/kserve/charts/kserve-localmodel-resources \
  --version v0.18.0 --namespace kserve
```

## Key Values (kserve-resources)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `kserve.controller.deploymentMode` | `Serverless` | `Serverless`, `RawDeployment`, `ModelMesh` |
| `kserve.controller.gateway.ingressGateway.enabled` | `true` | Enable ingress gateway config |
| `kserve.controller.gateway.ingressGateway.gatewayService` | `istio-ingressgateway.istio-system` | Ingress gateway service |
| `kserve.controller.gateway.ingressGateway.localGatewayService` | `knative-local-gateway.knative-serving` | Local gateway for mesh |
| `kserve.controller.gateway.clusterLocalGateway.enabled` | `true` | Enable cluster-local gateway |
| `kserve.controller.gateway.clusterLocalGateway.service` | `knative-local-gateway.knative-serving` | Local gateway service |

### Controller

| Parameter | Default | Description |
|-----------|---------|-------------|
| `kserve.controller.image` | `kserve/kserve-controller` | Controller image |
| `kserve.controller.tag` | `v0.18.0` | Image tag |
| `kserve.controller.replicas` | `1` | Controller replicas |
| `kserve.controller.resources` | `{cpu: 100m/500m, mem: 256Mi/1Gi}` | Container resources |
| `kserve.controller.nodeSelector` | `{}` | Node selector |
| `kserve.controller.tolerations` | `[]` | Tolerations |
| `kserve.controller.affinity` | `{}` | Pod affinity |
| `kserve.controller.topologySpreadConstraints` | `[]` | Topology spread |
| `kserve.controller.pdb.enabled` | `false` | PDB |
| `kserve.controller.pdb.maxUnavailable` | `1` | PDB max unavailable |
| `kserve.controller.env` | `[]` | Extra env vars |
| `kserve.controller.args` | `[]` | Extra args |

### Webhook

| Parameter | Default | Description |
|-----------|---------|-------------|
| `kserve.webhook.image` | `kserve/kserve-controller` | Webhook image |
| `kserve.webhook.tag` | `v0.18.0` | Image tag |
| `kserve.webhook.replicas` | `1` | Webhook replicas |
| `kserve.webhook.resources` | `{cpu: 100m/500m, mem: 256Mi/1Gi}` | Container resources |
| `kserve.webhook.nodeSelector` | `{}` | Node selector |
| `kserve.webhook.tolerations` | `[]` | Tolerations |
| `kserve.webhook.affinity` | `{}` | Pod affinity |

### Storage Initializer

| Parameter | Default | Description |
|-----------|---------|-------------|
| `kserve.storageInitializer.image` | `kserve/storage-initializer` | Init container image |
| `kserve.storageInitializer.tag` | `v0.18.0` | Image tag |
| `kserve.storageInitializer.resources` | `{cpu: 100m/1, mem: 200Mi/1Gi}` | Init container resources |
| `kserve.storageInitializer.env` | `[]` | Extra env vars |

### Agent

| Parameter | Default | Description |
|-----------|---------|-------------|
| `kserve.agent.image` | `kserve/agent` | Agent image |
| `kserve.agent.tag` | `v0.18.0` | Image tag |

### Router

| Parameter | Default | Description |
|-----------|---------|-------------|
| `kserve.router.image` | `kserve/router` | Router image |
| `kserve.router.tag` | `v0.18.0` | Image tag |

### Global

| Parameter | Default | Description |
|-----------|---------|-------------|
| `global.imagePullSecrets` | `[]` | Image pull secrets |
| `global.customMetrics.enabled` | `false` | Enable custom metrics adapter |

## Deployment Modes

### Standard Mode (Serverless - Knative)

Default mode. Auto-scaling to zero via Knative. Requires Knative Serving + Istio.

```bash
helm install kserve-resources oci://ghcr.io/kserve/charts/kserve-resources \
  --version v0.18.0 --namespace kserve \
  --set kserve.controller.deploymentMode=Serverless
```

**Characteristics:**
- Auto-scaling to zero (cold start)
- Traffic splitting (canary)
- Revision-based model versioning
- Requires Knative + Istio

### Raw Deployment Mode

Direct Deployments without Knative. No auto-scaling to zero. Simpler stack.

```bash
helm install kserve-resources oci://ghcr.io/kserve/charts/kserve-resources \
  --version v0.18.0 --namespace kserve \
  --set kserve.controller.deploymentMode=RawDeployment
```

**Characteristics:**
- No Knative dependency
- Standard K8s Deployments + Services
- HPA for autoscaling (minReplicas ≥ 1)
- Lighter resource usage

### ModelMesh Mode

Multi-model serving on shared runtimes. Requires ServingRuntime CRDs.

```bash
helm install kserve-resources oci://ghcr.io/kserve/charts/kserve-resources \
  --version v0.18.0 --namespace kserve \
  --set kserve.controller.deploymentMode=ModelMesh
```

**Characteristics:**
- Multiple models per pod (multiModel)
- ServingRuntime templates
- Model pooling
- Efficient GPU utilization

## Production Values Example

### Standard Mode Production

```yaml
kserve:
  controller:
    deploymentMode: Serverless
    replicas: 3
    resources:
      requests:
        cpu: 500m
        memory: 1Gi
      limits:
        cpu: 2
        memory: 4Gi
    env:
      - name: ENABLE_GPU
        value: "true"
    affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values:
                    - kserve-controller
            topologyKey: kubernetes.io/hostname
    pdb:
      enabled: true
      maxUnavailable: 1

  webhook:
    replicas: 3
    resources:
      requests:
        cpu: 200m
        memory: 512Mi
      limits:
        cpu: 500m
        memory: 1Gi

  storageInitializer:
    env:
      - name: AWS_ENDPOINT_URL
        value: s3.us-east-1.amazonaws.com
      - name: AWS_REGION
        value: us-east-1
```

### Raw Deployment Mode

```yaml
kserve:
  controller:
    deploymentMode: RawDeployment
    gateway:
      ingressGateway:
        enabled: false
      clusterLocalGateway:
        enabled: false
```

## Upgrading

```bash
helm repo update

# Upgrade CRDs first
helm upgrade kserve-crd oci://ghcr.io/kserve/charts/kserve-crd \
  --version v0.18.0 --namespace kserve

# Upgrade controller
helm upgrade kserve-resources oci://ghcr.io/kserve/charts/kserve-resources \
  --version v0.18.0 --namespace kserve

# Upgrade runtime configs
helm upgrade kserve-runtime-configs oci://ghcr.io/kserve/charts/kserve-runtime-configs \
  --version v0.18.0 --namespace kserve
```

## Uninstalling

```bash
# Uninstall resources (reverses operation order)
helm uninstall kserve-runtime-configs --namespace kserve
helm uninstall kserve-localmodel-resources --namespace kserve
helm uninstall kserve-localmodel-crd --namespace kserve
helm uninstall kserve-llmisvc-resources --namespace kserve
helm uninstall kserve-llmisvc-crd --namespace kserve
helm uninstall kserve-resources --namespace kserve

# Uninstall CRDs last
helm uninstall kserve-crd --namespace kserve

# Delete namespace
kubectl delete namespace kserve
```

**⚠️ WARNING:** Uninstalling `kserve-crd` deletes ALL CRDs and cascades to delete ALL InferenceServices across ALL namespaces.

## Common Mistakes

- **CRDs not installed first** — The controller chart fails without CRDs. Always install `kserve-crd` first.
- **Missing cert-manager** — KServe webhook requires cert-manager ≥ 1.12. Without it, controller pods crash-loop with webhook TLS errors.
- **Deployment mode mismatch** — `Serverless` mode requires Knative installed. `RawDeployment` does not. Set correctly before install.
- **Knative + Istio version mismatch** — KServe v0.18.0 requires Knative ≥ 1.16 and Istio ≥ 1.20. Version mismatch causes networking errors.
- **Stale webhook certs** — After upgrade, webhook certificates may need re-issuance. Restarting the webhook pod usually resolves.
- **Gateway service exists** — If `istio-ingressgateway.istio-system` doesn't exist in Standard mode, InferenceServices get `Failed to get gateway` status.
- **Localmodel with non-NVMe storage** — LocalModel cache works best with NVMe. Regular disks may cause model loading performance issues.
- **ModelMesh with single model** — ModelMesh expects `multiModel: true` on ServingRuntime. For single-model, use Standard mode.
- **v0.19.0-rc0 caution** — Release candidate. Test for production before using.
- **OCI chart vs Helm repo** — Use `oci://ghcr.io/kserve/charts/` for v0.18.0+ charts. Older versions used the HTTPS helm repo.
- **Annotations not propagated** — Some predictor annotations (e.g., `serving.kserve.io/deploymentMode`) must be on the InferenceService, not the pod template.
- **Storage initializer timeout** — Large models (>10GB) may exceed the default init container timeout. Increase via `kserve.storageInitializer.env`.
