---
name: kubeflow-trainer
description: Kubeflow Trainer v2.2 — TrainJob, TrainingRuntime, ClusterTrainingRuntime CRDs for framework-agnostic distributed training on Kubernetes. Covers PyTorch, TensorFlow, JAX, XGBoost, MPI, and Flux workloads.
---

# Kubeflow Trainer v2.2

**Repository:** `github.com/kubeflow/trainer`  
**Latest release:** v2.2.0 (March 20, 2026)  
**API version:** `trainer.kubeflow.org/v1alpha1`  
**Kubernetes:** v1.36  
**Underlying orchestration:** JobSet v0.10.1

## Architecture

Single TrainJob CRD replaces framework-specific CRDs. TrainingRuntime defines the execution template. Framework-agnostic: one abstraction for PyTorch, TensorFlow, JAX, XGBoost, MPI, Flux.

```
TrainJob (training intent)
  └─ runtimeRef ──> TrainingRuntime / ClusterTrainingRuntime (execution template)
                     └─ template ──> JobSet (underlying orchestration)
                                      └─ replicatedJobs ──> Jobs ──> Pods
```

## CRDs

### TrainJob (namespaced)

Defines the training workload.

```yaml
apiVersion: trainer.kubeflow.org/v1alpha1
kind: TrainJob
metadata:
  name: pytorch-mnist
spec:
  runtimeRef:
    name: pytorch-mnist-runtime
    apiGroup: trainer.kubeflow.org

  # Trainer definition
  trainer:
    image: pytorch/pytorch:2.5.0-cuda12.4-cudnn9-runtime
    command: ["python", "/workspace/train.py"]
    numNodes: 4
    env:
      - name: EPOCHS
        value: "10"
      - name: BATCH_SIZE
        value: "128"
    resources:
      requests:
        cpu: 4
        memory: 16Gi
      limits:
        nvidia.com/gpu: 2

  # Optional: dataset/model initializer (sidecar that runs before training)
  initializer:
    storageUri: s3://bucket/datasets/mnist/
    env:
      - name: AWS_ENDPOINT_URL
        value: s3.example.com
    secretRef:
      name: s3-credentials

  # Runtime patches (replaces deprecated PodTemplateOverrides)
  runtimePatches:
    - managerKey: user-overrides
      patch:
        - op: add
          path: /spec/replicatedJobs/0/template/spec/template/spec/containers/0/env/-
          value:
            name: LOG_LEVEL
            value: debug

  # Hard deadline
  activeDeadlineSeconds: 3600

  # Kueue integration
  managedBy: kueue.x-k8s.io/multikueue
```

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `runtimeRef` | `RuntimeRef` | ✅ | Reference to TrainingRuntime or ClusterTrainingRuntime |
| `trainer` | `Trainer` | ✅ | Image, command, numNodes, env, resources |
| `initializer` | `Initializer` | ❌ | Sidecar for dataset/model preloading |
| `runtimePatches` | `[]RuntimePatch` | ❌ | Multi-owner patches (manager-keyed with timestamps) |
| `activeDeadlineSeconds` | `int64` | ❌ | Hard timeout; expired job gets DeadlineExceeded |
| `managedBy` | `string` | ❌ | `trainjob-controller` (default) or `kueue.x-k8s.io/multikueue` |

#### Status

```yaml
status:
  conditions:
    - type: Accepted
      status: "True"
    - type: Started
      status: "True"
    - type: Completed
      status: "True"

  # Optional (requires TrainJobProgress feature gate)
  trainerStatus:
    progress: 75          # percentage
    eta: "2026-06-12T14:30:00Z"
    metrics:
      loss: 0.023
      accuracy: 0.992
```

### TrainingRuntime (namespaced) / ClusterTrainingRuntime (cluster-scoped)

Defines how the TrainJob executes.

```yaml
apiVersion: trainer.kubeflow.org/v1alpha1
kind: TrainingRuntime
metadata:
  name: pytorch-mnist-runtime
spec:
  # ML framework policy (exactly one)
  mlPolicy:
    torch:
      numGPUsPerNode: 2
      elastic:
        minReplicas: 2
        maxReplicas: 8
        nprocPerNode: 1

  # Gang scheduling policy
  podGroupPolicy:
    cosmoscheduling:
      minMember: 4
    # OR
    # volcano:
    #   minMember: 4
    #   queue: default

  # JobSet template
  template:
    spec:
      replicatedJobs:
        - name: worker
          replicas: 4
          template:
            spec:
              containers:
                - name: trainer
                  image: pytorch/pytorch:2.5.0-cuda12.4-cudnn9-runtime
              restartPolicy: OnFailure
      failurePolicy:
        maxRetries: 3
```

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mlPolicy` | `MLPolicy` | ✅ | Exactly one: `torch`, `mpi`, `jax`, `xgboost`, or `flux` |
| `podGroupPolicy` | `PodGroupPolicy` | ❌ | `coscheduling` or `volcano` for gang-scheduling |
| `template` | `JobSetTemplate` | ✅ | JobSet spec template (replicatedJobs, failurePolicy, network, coordinator) |

#### ML Policy Types

| Framework | Fields | Notes |
|-----------|--------|-------|
| **torch** | `numGPUsPerNode`, `elastic` (minReplicas, maxReplicas, nprocPerNode) | Sets MASTER_ADDR/MASTER_PORT, WORLD_SIZE, RANK env vars |
| **mpi** | `numCPUsPerNode`, `numMPIWorkersPerNode`, `sshdPort` | MPI launcher manages distributed setup |
| **jax** | `numCPUsPerNode` | JAX uses its own SPMD/FSDP coordination |
| **xgboost** | `numCPUsPerNode`, `numXGBoostWorkersPerNode` | XGBoost Rabit tracker managed by runtime |
| **flux** | `numCPUsPerNode`, `fluxMinReplicas`, `fluxMaxReplicas` | Flux HPC framework (e.g., LAMMPS workloads) |

### RuntimePatches API

Multi-owner patching system replacing the old PodTemplateOverrides. Each patch is keyed by `managerKey` with automatic timestamp ordering.

```yaml
runtimePatches:
  - managerKey: user-overrides
    patch:
      - op: add
        path: /spec/replicatedJobs/0/template/spec/template/spec/containers/0/env/-
        value:
          name: EXPERIMENT_ID
          value: "42"
  - managerKey: infra
    patch:
      - op: replace
        path: /spec/replicatedJobs/0/template/spec/template/spec/containers/0/resources/limits/nvidia\.com/gpu
        value: 4
```

Patch targets: JobSet → ReplicatedJob → Pod → container levels.

**Immutable fields** (cannot be patched after TrainJob creation): `runtimeRef`, `initializer`, `trainer.numNodes`, `trainer.image`.

## Scheduling Integration

### Kueue (recommended)

```yaml
spec:
  managedBy: kueue.x-k8s.io/multikueue
```

Requires matching labels on TrainJob:

```yaml
metadata:
  labels:
    kueue.x-k8s.io/queue-name: training-queue
```

Kueue handles admission, gang-scheduling, topology-aware scheduling (`kubernetes-sigs/kueue#7249`), and preemption.

### Volcano

```yaml
spec:
  podGroupPolicy:
    volcano:
      minMember: 4
      queue: default
```

Volcano provides gang-scheduling, resource fairness, and job priorities.

### DRA (Dynamic Resource Allocation)

Coming post-K8s 1.36 for on-demand accelerator allocation (GPU partitioning, NIC bonding, etc.).

## TrainJobProgress Feature Gate

Optional feature for real-time training observability. Enable via:

```yaml
spec:
  trainer:
    image: ...
```

Requires the TrainJobProgress feature gate enabled in the controller manager. Runs an HTTPS progress server that reports: percentage complete, estimated time of arrival, loss/metrics, and node-level status.

## Examples

### Multi-node PyTorch TrainJob

```yaml
apiVersion: trainer.kubeflow.org/v1alpha1
kind: TrainJob
metadata:
  name: pytorch-distributed
spec:
  runtimeRef:
    name: pytorch-runtime
  trainer:
    image: pytorch/pytorch:2.5.0-cuda12.4-cudnn9-runtime
    command:
      - torchrun
      - --nnodes=4
      - --nproc-per-node=2
      - /workspace/train.py
    numNodes: 4
    resources:
      limits:
        nvidia.com/gpu: 2
---
apiVersion: trainer.kubeflow.org/v1alpha1
kind: TrainingRuntime
metadata:
  name: pytorch-runtime
spec:
  mlPolicy:
    torch:
      numGPUsPerNode: 2
  podGroupPolicy:
    cosmoscheduling:
      minMember: 4
  template:
    spec:
      replicatedJobs:
        - name: worker
          replicas: 4
          template:
            spec:
              containers:
                - name: trainer
                  image: pytorch/pytorch:2.5.0-cuda12.4-cudnn9-runtime
              restartPolicy: OnFailure
```

### Kueue-integrated TrainJob

```yaml
apiVersion: trainer.kubeflow.org/v1alpha1
kind: TrainJob
metadata:
  name: kueue-job
  labels:
    kueue.x-k8s.io/queue-name: training-queue
spec:
  runtimeRef:
    name: gpu-runtime
  trainer:
    image: tensorflow/tensorflow:2.17.0-gpu
    command: ["python", "/workspace/train.py"]
    numNodes: 2
    resources:
      limits:
        nvidia.com/gpu: 4
  managedBy: kueue.x-k8s.io/multikueue
```

### Flux HPC (LAMMPS)

```yaml
apiVersion: trainer.kubeflow.org/v1alpha1
kind: TrainingRuntime
metadata:
  name: flux-runtime
spec:
  mlPolicy:
    flux:
      numCPUsPerNode: 32
      fluxMinReplicas: 2
      fluxMaxReplicas: 16
  template:
    spec:
      replicatedJobs:
        - name: worker
          replicas: 4
          template:
            spec:
              containers:
                - name: trainer
                  image: fluxrm/flux-sched:latest
              restartPolicy: OnFailure
```

## Kubeflow Pipelines v2 Integration

TrainJobs can be orchestrated within KFP v2 pipelines as a component:

```python
from kfp import dsl
from kfp import kubernetes as k8s

@dsl.component(base_image="python:3.11")
def create_train_job_manifest(
    project: str,
    runtime: str,
    nodes: int,
    gpus_per_node: int,
) -> str:
    import yaml
    trainjob = {
        "apiVersion": "trainer.kubeflow.org/v1alpha1",
        "kind": "TrainJob",
        "metadata": {"name": f"train-{project}"},
        "spec": {
            "runtimeRef": {"name": runtime},
            "trainer": {
                "image": f"{project}/trainer:latest",
                "numNodes": nodes,
                "resources": {"limits": {"nvidia.com/gpu": gpus_per_node}},
            },
        },
    }
    return yaml.dump(trainjob)
```

## Katib Hyperparameter Tuning

TrainJobs can be used as trial templates in Katib:

```yaml
apiVersion: kubeflow.org/v1beta1
kind: Experiment
metadata:
  name: pytorch-hpo
spec:
  objective:
    type: maximize
    goal: 0.99
    objectiveMetricName: accuracy
  algorithm:
    algorithmName: random
  trialTemplate:
    trainJobSpec:
      runtimeRef:
        name: pytorch-runtime
      trainer:
        image: pytorch/pytorch:2.5.0-cuda12.4-cudnn9-runtime
        numNodes: 2
        env:
          - name: LR
            value: {{.HyperParameters.lr}}
  parameters:
    - name: lr
      parameterType: double
      feasibleSpace:
        min: "0.001"
        max: "0.1"
```

## Migration from v1 Training Operator

See `kubeflow-training-operator` skill for legacy CRDs. Migration guide:

1. Replace PyTorchJob → TrainJob + TrainingRuntime with `mlPolicy.torch`
2. Replace TFJob → TrainJob + TrainingRuntime with `mlPolicy.mpi` (TF_CONFIG is set automatically)
3. Replace MPIJob → TrainJob + TrainingRuntime with `mlPolicy.mpi`
4. Replace XGBoostJob → TrainJob + TrainingRuntime with `mlPolicy.xgboost`
5. Update `training-operator.kubeflow.org` → `trainer.kubeflow.org/v1alpha1`

## Common Mistakes

- **RuntimeRef changes** — `runtimeRef` is immutable after TrainJob creation. Delete and recreate to change.
- **JobSet not installed** — Trainer v2.2 requires JobSet v0.10.1+. Install separately (`kubectl apply -f https://github.com/kubernetes-sigs/jobset/releases/download/v0.10.1/manifests.yaml`).
- **Missing gang-scheduling** — Without `podGroupPolicy`, multiple TrainJobs may over-provision nodes. Always configure coscheduling or Volcano for shared clusters.
- **GPU count mismatch** — `mlPolicy.torch.numGPUsPerNode` must match `trainer.resources.limits["nvidia.com/gpu"]`.
- **TrainJobProgress feature gate** — Not enabled by default. Must be explicitly enabled in the controller.
- **RuntimePatches ordering** — Patches are applied in timestamp order within each manager key. Concurrent patches from different managers may conflict.
- **Kueue managedBy** — Setting `managedBy: kueue.x-k8s.io/multikueue` requires Kueue v0.12+ with Multikueue feature.
