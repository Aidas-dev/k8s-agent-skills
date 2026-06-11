# KServe — Operator CRDs & Configuration

**Repository:** `github.com/kserve/kserve`  
**Latest release:** v0.18.0 (Apr 29, 2026) — **v0.19.0-rc0** available  
**CNCF:** Incubating (Jan 2025)  
**License:** Apache 2.0  
**Stars:** 10k+

## Architecture

```
RawPredictOp → Preprocess → Predict → [Explain] → Postprocess → ExplainOp
                (Transformer)  (Predictor)           (Transformer)

InferenceService (CRD) ──► KServe Controller ──► Deployment + Service + HPA + Istio/Knative
                                │
                          Storage: S3/GCS/Azure/HF/PVC/HTTP/HDFS
                                │
                     Modelcar (OCI) or PVC or storage init container
```

### Deployment Modes

| Mode | Networking | Use Case |
|------|-----------|----------|
| **Standard** | Istio VirtualService | Enterprise, stable, full traffic control |
| **Knative** | Knative + Istio (Kourier optional) | Auto-scaling to zero, serverless |
| **Raw** | Direct Service + Ingress/Gateway | Simple, lightweight, no service mesh |

### Annotations Control

| Annotation | Effect |
|-----------|--------|
| `serving.kserve.io/deploymentMode` | `Serverless` (Knative, default), `RawDeployment`, `ModelMesh` |
| `serving.tempaltes.kserve.io/default` | Template rendering default |
| `sidecar.istio.io/inject` | `true` to inject Istio sidecar |

## CRDs (22 total — primary 9 covered here)

All under `serving.kserve.io`:

| CRD | API Version | Scope | Short Name | Description |
|-----|-------------|-------|-----------|-------------|
| **InferenceService** | `v1beta1` / `v1alpha1` | Namespaced | `isvc` | Core serving unit |
| **ServingRuntime** | `v1alpha1` | Namespaced | `sr` | Runtime template (ModelMesh) |
| **ClusterServingRuntime** | `v1alpha1` | Cluster | `csr` | Cluster-scoped runtime template |
| **InferenceGraph** | `v1alpha1` | Namespaced | `ig` | Router graph (ensemble/switch/splitter) |
| **TrainedModel** | `v1alpha1` | Namespaced | `tm` | Model-scoped config for ServingRuntime |
| **LLMInferenceService** | `v1alpha1` | Namespaced | — | LLM-specific (disaggregated prefill/decode) |
| **LocalModelCache** | `v1alpha1` | Namespaced | `lm` | Node-level model cache for LocalModelNode |
| **LocalModelNode** | `v1alpha1` | Cluster | `lmn` | Node DaemonSet for model loading |
| **ClusterStorageContainer** | `v1alpha1` | Cluster | `csc` | Default storage container template |

## InferenceService — `serving.kserve.io/v1beta1`

The primary CRD for model serving.

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: sklearn-iris
spec:
  predictor:
    # Built-in predictor
    sklearn:
      storageUri: s3://models/iris/model.joblib
      resources:
        requests:
          cpu: 100m
          memory: 256Mi
        limits:
          cpu: "1"
          memory: 1Gi
      readinessProbe:
        periodSeconds: 5
        successThreshold: 1
        timeoutSeconds: 1
      livenessProbe:
        periodSeconds: 5
        failureThreshold: 3
        timeoutSeconds: 1

    # ServiceOrchestration (multi-model composition)
    # serviceOrchestrationSpec:
    #   pipelining: router

    # Node selection
    # nodeSelector:
    #   node-type: gpu

    # Service account for storage access
    serviceAccountName: kserve-sa

    # Min/max replicas
    minReplicas: 1
    maxReplicas: 5

    # Scale target (Knative mode)
    scaleTarget: 1
    scaleMetric: concurrency
    containerConcurrency: 5

    # Canary rollout
    canaryTrafficPercent: 10
    canary:
      sklearn:
        storageUri: s3://models/iris-v2/model.joblib

  # Transformer (optional, pre/post-processing)
  transformer:
    container:
      image: myrepo/transformer:latest
      env:
        - name: MODEL_NAME
          value: iris
      resources:
        requests:
          cpu: 100m
          memory: 128Mi

  # Explainer (optional)
  explainer:
    alibi:
      storageUri: s3://models/iris-explainer/
      resources:
        requests:
          cpu: 100m
          memory: 256Mi

  # Storage via modelcar (OCI)
  # storage:
  #   modelcar:
  #     enabled: true
  #     pullAlways: true
```

### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `predictor` | PredictorSpec | ✅ | Model predictor configuration |
| `predictor.minReplicas` | int | ❌ | Min pods |
| `predictor.maxReplicas` | int | ❌ | Max pods |
| `predictor.scaleTarget` | int | ❌ | Scale to zero target |
| `predictor.scaleMetric` | string | ❌ | `concurrency`, `rps`, `cpu`, `memory` |
| `predictor.containerConcurrency` | int | ❌ | Max concurrent requests (Knative) |
| `predictor.canaryTrafficPercent` | int | ❌ | Canary traffic percentage |
| `predictor.canary` | object | ❌ | Canary predictor spec (same structure) |
| `predictor.serviceAccountName` | string | ❌ | KSA for storage access |
| `predictor.readinessProbe` | Probe | ❌ | Custom readiness probe |
| `predictor.livenessProbe` | Probe | ❌ | Custom liveness probe |
| `predictor.annotations` | map | ❌ | Pod annotations |
| `predictor.nodeSelector` | map | ❌ | Node selector |
| `predictor.affinity` | Affinity | ❌ | Pod affinity |
| `predictor.tolerations` | []Toleration | ❌ | Node tolerations |
| `predictor.topologySpreadConstraints` | []TSC | ❌ | Topology spread |
| `predictor.sidecarInject` | string | ❌ | `istio-proxy`, `envoy`, etc. |
| `predictor.serviceOrchestrationSpec` | object | ❌ | Multi-model routing |
| `predictor.storage` | StorageSpec | ❌ | Storage config |
| `transformer` | ComponentSpec | ❌ | Pre/post-processing |
| `explainer` | ComponentSpec | ❌ | Model explainability |
| `storage.modelcar.enabled` | bool | ❌ | Modelcar OCI mode |
| `storage.modelcar.pullAlways` | bool | ❌ | Always pull model image |

### Built-in Predictors

| Predictor | Spec Field | Image (default) | Protocol |
|-----------|-----------|-----------------|----------|
| **SKLearn** | `sklearn` | `kserve/sklearnserver` | REST/gRPC (v2) |
| **TensorFlow** | `tensorflow` | `kserve/tfserving` | REST/gRPC (TFServing) |
| **PyTorch** | `pytorch` | `kserve/torchserve` | REST/gRPC (TorchServe) |
| **Triton** | `triton` | `nvcr.io/nvidia/tritonserver` | REST/gRPC (KServe v2) |
| **ONNX** | `onnx` | `kserve/onnxserver` | REST/gRPC (v2) |
| **XGBoost** | `xgboost` | `kserve/xgbserver` | REST/gRPC (v2) |
| **LightGBM** | `lightgbm` | `kserve/lgbserver` | REST/gRPC (v2) |
| **PMML** | `pmml` | `kserve/pmmlserver` | REST (PMML) |
| **Paddle** | `paddle` | `kserve/paddleserver` | REST/gRPC |
| **HuggingFace** | `huggingface` | `kserve/huggingfaceserver` | REST (TGI/transformers) |

```yaml
# SKLearn
predictor:
  sklearn:
    storageUri: s3://models/iris/
    protocolVersion: v2

# Triton
predictor:
  triton:
    storageUri: s3://models/triton-repo/
    runtimeVersion: 24.12
    protocolVersion: v2  # KServe v2 protocol or grpc-v2

# HuggingFace
predictor:
  huggingface:
    storageUri: s3://models/llama/
    resources:
      limits:
        nvidia.com/gpu: 1
    env:
      - name: HF_HUB_DISABLE_TELEMETRY
        value: "true"
```

### Custom Predictor (Container)

```yaml
predictor:
  containers:
    - name: kserve-container
      image: myrepo/custom-model:latest
      ports:
        - containerPort: 8080
          protocol: TCP
      env:
        - name: MODEL_DIR
          value: /models
      resources:
        requests:
          cpu: 500m
          memory: 1Gi
        limits:
          nvidia.com/gpu: 1
```

### Multi-Model Composition

```yaml
predictor:
  serviceOrchestrationSpec:
    pipelining: router  # OR tf (TensorFlow DAG)
  sklearn:
    storageUri: s3://models/step1/
  pytorch:
    storageUri: s3://models/step2/
```

### Examples

**S3 model with canary rollout:**
```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: mymodel
spec:
  predictor:
    minReplicas: 2
    canaryTrafficPercent: 10
    sklearn:
      storageUri: s3://models/v1/
    canary:
      sklearn:
        storageUri: s3://models/v2/
```

**GPU serving with token auth:**
```yaml
predictor:
  minReplicas: 1
  maxReplicas: 3
  pytorch:
    storageUri: s3://models/resnet/
    resources:
      limits:
        nvidia.com/gpu: 1
```

## ServingRuntime / ClusterServingRuntime — `v1alpha1`

Used with ModelMesh deployment mode. Defines runtime templates for model serving.

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: ServingRuntime
metadata:
  name: sklearn-runtime
spec:
  # Supported model formats
  supportedModelFormats:
    - name: sklearn
      version: "1"
      autoSelect: true
    - name: sklearn
      version: "2"

  # Runtime container
  containers:
    - name: kserve-container
      image: kserve/sklearnserver:v0.18.0
      args:
        - --model-dir=/models
      resources:
        requests:
          cpu: 500m
          memory: 1Gi
        limits:
          cpu: "1"
          memory: 2Gi

  # Node-level multi-model pooling (ModelMesh)
  multiModel: true
  modelSize: Medium  # Small | Medium | Large | XLarge
  replicas: 2

  # Pool-scoped (ModelMesh)
  pooled: true

  # Serverless scaling behavior
  scaleToZero:
    enabled: false
    gracePeriod: 300

  # Protocol
  protocolVersions:
    - v2  # KServe v2 protocol

  # Built-in adapter (REST/gRPC inference adapter)
  # builtInAdapter: {}  # optional
```

### ClusterServingRuntime

Same spec but cluster-scoped. Available across all namespaces.

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: ClusterServingRuntime
metadata:
  name: triton-runtime
spec:
  supportedModelFormats:
    - name: triton
      autoSelect: true
  containers:
    - name: kserve-container
      image: nvcr.io/nvidia/tritonserver:24.12-py3
      args:
        - tritonserver
        - --model-repository=/models
  multiModel: true
  protocolVersions:
    - grpc-v2
    - v2
```

### Spec Fields

| Field | Type | Description |
|-------|------|-------------|
| `supportedModelFormats[]` | []ModelFormat | Model formats this runtime supports |
| `supportedModelFormats[].name` | string | Format name |
| `supportedModelFormats[].version` | string | Format version |
| `supportedModelFormats[].autoSelect` | bool | Auto-select for this format |
| `containers[]` | []Container | Runtime containers |
| `multiModel` | bool | Serve multiple models per pod |
| `modelSize` | string | `Small`, `Medium`, `Large`, `XLarge` |
| `replicas` | int | Min replicas |
| `pooled` | bool | Pool-scoped runtime |
| `scaleToZero.enabled` | bool | Allow scale to zero |
| `scaleToZero.gracePeriod` | int | Grace period before scale to zero |
| `protocolVersions[]` | []string | Inference protocol versions |
| `builtInAdapter` | object | Built-in adapter config |
| `disabled` | bool | Disable this runtime |

## InferenceGraph — `v1alpha1`

Router graph for multi-model routing.

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: InferenceGraph
metadata:
  name: model-router
spec:
  # Route nodes
  nodes:
    # Entry point
    root:
      routerType: Sequence  # Sequence | Switch | Ensemble | Splitter
      routes:
        - serviceUrl: predictor1.default.svc.cluster.local
          weight: 80
        - serviceUrl: predictor2.default.svc.cluster.local
          weight: 20

    # Conditional routing
    classifier:
      routerType: Switch
      condition: "input.category == 'a'"
      routes:
        - serviceUrl: model-a.default.svc.cluster.local
          data: "$request"
        - serviceUrl: model-b.default.svc.cluster.local
          data: "$request"

    # Ensemble (merge results)
    ensemble:
      routerType: Ensemble
      routes:
        - serviceUrl: model-a.default.svc
        - serviceUrl: model-b.default.svc

    # Splitter (fan-out)
    splitter:
      routerType: Splitter
      routes:
        - serviceUrl: shard-0.default.svc
        - serviceUrl: shard-1.default.svc

  # Entry point router
  entryPoint: root

  # Auth
  auth:
    enabled: true
    token: my-auth-token

  # Autoscaling
  autoscaler:
    initialScale: 1
    maxScale: 10
    scaleDownDelay: 300
```

### Router Types

| Type | Behavior |
|------|----------|
| `Sequence` | Step-by-step pipeline |
| `Switch` | Conditional routing (first match) |
| `Ensemble` | Fan-out, merge results |
| `Splitter` | Fan-out by weight |

### Spec Fields

| Field | Type | Description |
|-------|------|-------------|
| `nodes` | map[string]Node | Router node definitions |
| `entryPoint` | string | Entry node name |
| `auth.enabled` | bool | Enable auth on graph |
| `auth.token` | string | Auth token |
| `auth.tokenSecretKeyRef` | object | Token from secret |
| `autoscaler` | Autoscaler | Autoscaling config |

**Node fields:**

| Field | Type | Description |
|-------|------|-------------|
| `routerType` | string | `Sequence`, `Switch`, `Ensemble`, `Splitter` |
| `routes[]` | []Route | Route definitions |
| `routes[].serviceUrl` | string | InferenceService URL |
| `routes[].weight` | int | Traffic weight |
| `routes[].data` | string | Request data template |
| `condition` | string | Condition expression (Switch only) |

## LLMInferenceService — `v1alpha1`

LLM-optimized InferenceService for disaggregated prefill/decode, vLLM integration.

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: llama-llm
spec:
  # LLM model
  modelProvider: vllm
  modelName: llama-3-70b
  modelStorage: s3://models/llama-3-70b/

  # Disaggregated serving (separate prefill/decode)
  disaggregated: true
  prefill:
    replicas: 2
    resources:
      limits:
        nvidia.com/gpu: 4
    env:
      - name: MAX_BATCH_SIZE
        value: "256"
  decode:
    replicas: 4
    resources:
      limits:
        nvidia.com/gpu: 1

  # vLLM specific
  vllm:
    maxModelLen: 8192
    tensorParallelSize: 4
    gpuMemoryUtilization: 0.9

  # Autoscaling
  autoscaler:
    initialScale: 1
    maxScale: 10
    downscaleDelay: 300

  # Storage
  storage:
    modelcar:
      enabled: true
```

### Spec Fields

| Field | Type | Description |
|-------|------|-------------|
| `modelProvider` | string | `vllm`, `triton`, `tgi`, `transformers` |
| `modelName` | string | Model identifier |
| `modelStorage` | string | Model URI (`s3://...`, `pvc://...`, `hf://...`) |
| `disaggregated` | bool | Separate prefill/decode pods |
| `prefill.replicas` | int | Prefill pod count |
| `prefill.resources` | ResourceRequirements | Prefill GPU resources |
| `decode.replicas` | int | Decode pod count |
| `decode.resources` | ResourceRequirements | Decode GPU resources |
| `vllm.maxModelLen` | int | Max sequence length |
| `vllm.tensorParallelSize` | int | TP degree |
| `vllm.gpuMemoryUtilization` | float | GPU mem utilization (0-1) |
| `autoscaler` | Autoscaler | Scaling config |
| `storage` | StorageSpec | Model storage config |

## Storage Configuration

### Storage URI Formats

| Scheme | Format | Credentials |
|--------|--------|-------------|
| **S3** | `s3://bucket/key` | SecretRef or IRSA (IAM roles for service accounts) |
| **GCS** | `gs://bucket/key` | Workload Identity or SecretRef |
| **Azure** | `az://container/key` | SecretRef (account key) |
| **HuggingFace** | `hf://model-id` | `HF_TOKEN` env var |
| **PVC** | `pvc://claim/path` | PVC must exist |
| **HTTP** | `http(s)://url` | No auth needed |
| **HDFS** | `hdfs://nn:port/path` | SecretRef |

### Storage Credentials via Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: storage-creds
  annotations:
    serving.kserve.io/s3-endpoint: s3.us-east-1.amazonaws.com
    serving.kserve.io/s3-usehttps: "1"
    serving.kserve.io/s3-region: us-east-1
    serving.kserve.io/s3-verify-ssl: "1"
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: AKIA...
  AWS_SECRET_ACCESS_KEY: ...
```

Reference via service account:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kserve-sa
secrets:
  - name: storage-creds
---
spec:
  predictor:
    serviceAccountName: kserve-sa
```

### IRSA (IAM Roles for Service Accounts) — AWS

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kserve-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/kserve-s3-access
```

### ClusterStorageContainer — `v1alpha1`

Default storage container template applied across namespaces.

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: ClusterStorageContainer
metadata:
  name: s3-storage
spec:
  container:
    name: storage-initializer
    image: kserve/storage-initializer:v0.18.0
    env:
      - name: AWS_ENDPOINT_URL
        value: s3.us-east-1.amazonaws.com
      - name: AWS_REGION
        value: us-east-1
  supportedUriFormats:
    - prefix: s3://
```

## TrainedModel — `v1alpha1`

Model-scoped configuration within a ServingRuntime.

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: TrainedModel
metadata:
  name: iris-v2
  namespace: default
spec:
  inferenceService: sklearn-iris
  model:
    storageUri: s3://models/iris-v2/
    framework: sklearn
    memory: 256Mi
  scaleTarget: 1
```

## LocalModelCache / LocalModelNode — `v1alpha1`

Node-level model caching for faster start times.

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LocalModelCache
metadata:
  name: llm-cache
spec:
  modelSize: 70Gi
  nodeGroup: gpu-nodes
  storage:
    local:
      nodeStateRoot: /mnt/local-models
    persistentVolumeClaim:
      name: model-cache-pvc
      size: 200Gi
---
apiVersion: serving.kserve.io/v1alpha1
kind: LocalModelNode
metadata:
  name: gpu-node-1
spec:
  cacheConfigRef: llm-cache
  nodeName: gpu-worker-1
  nodeAgentImage: kserve/local-model-node-agent:v0.18.0
```

## Modelcar OCI Mode

Model stored as an OCI image, loaded into the serving pod without init container.

```yaml
predictor:
  storage:
    modelcar:
      enabled: true
      pullAlways: true
  containers:
    - name: kserve-container
      image: myrepo/my-model:latest  # model AND runtime combined
```

## Common Patterns

**Minimal inference with S3:**
```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: quick-start
spec:
  predictor:
    sklearn:
      storageUri: s3://kserve-examples/sklearn/iris
```

**Multi-node Triton with GPU:**
```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: triton-gpu
spec:
  predictor:
    triton:
      storageUri: s3://models/triton-repo/
      runtimeVersion: 24.12
      protocolVersion: grpc-v2
      resources:
        limits:
          nvidia.com/gpu: 2
    minReplicas: 2
    maxReplicas: 8
```

**Canary rollout with traffic splitting:**
```yaml
spec:
  predictor:
    sklearn:
      storageUri: s3://models/v1/
    canaryTrafficPercent: 10
    canary:
      sklearn:
        storageUri: s3://models/v2/
```

**InferenceGraph with conditional routing:**
```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: InferenceGraph
metadata:
  name: smart-router
spec:
  nodes:
    root:
      routerType: Switch
      condition: "input.model == 'large'"
      routes:
        - serviceUrl: gpu-cluster.default.svc
    default:
      routerType: Sequence
      routes:
        - serviceUrl: cpu-cluster.default.svc
  entryPoint: root
```

**LLMInferenceService with vLLM:**
```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: llama-service
spec:
  modelProvider: vllm
  modelStorage: s3://models/llama-3-8b/
  disaggregated: false
  vllm:
    maxModelLen: 4096
    tensorParallelSize: 1
    gpuMemoryUtilization: 0.9
  autoscaler:
    initialScale: 1
```

## Common Mistakes

- **Storage endpoint annotation mismatch** — S3 storage requires `serving.kserve.io/s3-endpoint` annotation on the secret, not an env var in the predictor.
- **Missing service account** — Without a service account with storage credentials, predictions fail with "model not found" at the `/models` mount point.
- **Multi-model vs single-model** — ModelMesh (serving.kserve.io/deploymentMode=ModelMesh) uses ServingRuntime for multi-model. Standard mode uses InferenceService per model.
- **Protocol version confusion** — `v1` = KServe v1 protocol (legacy, sklearn/tensorflow). `v2` = KServe v2 protocol (Triton, MLServer, most built-in). `grpc-v2` = gRPC KServe v2.
- **Knative scaling to zero** — Default is `minReplicas: 0` in Knative mode. Set `minReplicas: 1` to avoid cold starts.
- **Canary traffic percent integer** — Must be 0–100 integer. Fractional values not supported.
- **Storage URI trailing slash** — Some storage backends are sensitive to trailing slashes in `storageUri`.
- **InferenceService vs LLMInferenceService** — LLMInferenceService is a separate CRD, not a field on InferenceService. Use the v1alpha1 API version.
- **Disaggregated prefill/decode** — Requires multi-node GPU topology. Prefill uses more GPU memory per token, decode needs lower latency.
- **Modelcar with non-OCI image** — `storage.modelcar.enabled: true` expects the predictor image to embed the model. Standard init container mode is default.
