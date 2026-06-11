---
name: kubeflow-training-operator
description: Legacy Kubeflow Training Operator v1 — PyTorchJob, TFJob, MPIJob, XGBoostJob, ModelMesh CRDs for distributed training on Kubernetes (deprecated in favor of Trainer v2).
---

# Kubeflow Training Operator v1 (Legacy)

**Repository:** `github.com/kubeflow/training-operator`  
**API version:** `kubeflow.org/v1`  
**Status:** **Deprecated** — maintained on release-1.9 branch. New deployments should use `kubeflow-trainer` skill (Trainer v2).

## Migration Status

| v1 CRD | v2 Equivalent | Migration Status |
|--------|--------------|------------------|
| PyTorchJob | TrainJob + Runtime (`mlPolicy.torch`) | ✅ Fully migrated |
| TFJob | TrainJob + Runtime (`mlPolicy.mpi`) | ✅ Fully migrated |
| MPIJob | TrainJob + Runtime (`mlPolicy.mpi`) | ✅ Fully migrated |
| XGBoostJob | TrainJob + Runtime (`mlPolicy.xgboost`) | ✅ Supported in v2.2 |
| PaddleJob | TrainJob + Runtime | ⚠️ Pending |
| ModelMesh | — | Separate project |

**Migration guide:** [kubeflow.org/docs/components/trainer/operator-guides/migration/](https://www.kubeflow.org/docs/components/trainer/operator-guides/migration/)

## CRDs

### PyTorchJob

```yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: pytorch-mnist
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
            - name: pytorch
              image: pytorch/pytorch:2.5.0-cuda12.4-cudnn9-runtime
              command:
                - python
                - /workspace/train.py
              resources:
                limits:
                  nvidia.com/gpu: 1
    Worker:
      replicas: 3
      restartPolicy: OnFailure
      template:
        spec:
          containers:
            - name: pytorch
              image: pytorch/pytorch:2.5.0-cuda12.4-cudnn9-runtime
              command:
                - python
                - /workspace/train.py
              resources:
                limits:
                  nvidia.com/gpu: 1
```

### TFJob

```yaml
apiVersion: kubeflow.org/v1
kind: TFJob
metadata:
  name: tf-mnist
spec:
  tfReplicaSpecs:
    Chief:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
            - name: tensorflow
              image: tensorflow/tensorflow:2.17.0-gpu
              command: ["python", "/workspace/train.py"]
    Worker:
      replicas: 2
      restartPolicy: OnFailure
      template:
        spec:
          containers:
            - name: tensorflow
              image: tensorflow/tensorflow:2.17.0-gpu
              command: ["python", "/workspace/train.py"]
    PS:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
            - name: tensorflow
              image: tensorflow/tensorflow:2.17.0-gpu
              command: ["python", "/workspace/train.py"]
```

### MPIJob

```yaml
apiVersion: kubeflow.org/v1
kind: MPIJob
metadata:
  name: mpi-job
spec:
  slotsPerWorker: 4
  mpiReplicaSpecs:
    Launcher:
      replicas: 1
      template:
        spec:
          containers:
            - image: mpi-image:latest
              command: ["mpirun", "-np", "4", "/workspace/train"]
    Worker:
      replicas: 2
      template:
        spec:
          containers:
            - image: mpi-image:latest
              resources:
                limits:
                  nvidia.com/gpu: 2
```

### XGBoostJob

```yaml
apiVersion: kubeflow.org/v1
kind: XGBoostJob
metadata:
  name: xgboost-job
spec:
  xgbReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
            - name: xgboost
              image: xgboost/xgboost:latest
              command: ["python", "/workspace/train.py"]
    Worker:
      replicas: 3
      restartPolicy: OnFailure
      template:
        spec:
          containers:
            - name: xgboost
              image: xgboost/xgboost:latest
              command: ["python", "/workspace/train.py"]
```

## Common Fields (all v1 CRDs)

| Field | Description |
|-------|-------------|
| `spec.<framework>ReplicaSpecs` | Map of replica types (Master/Worker/Chief/PS/Launcher) |
| `replicas` | Number of pods for this replica type |
| `restartPolicy` | `Never`, `OnFailure`, `ExitCode` |
| `template` | Standard PodTemplateSpec |
| `runPolicy` | CleanPodPolicy, scheduling, backoff, ttlSecondsAfterFinished |
| `spec.slotsPerWorker` | (MPIJob) GPUs/accelerators per worker |

## Common Mistakes

- **Not migrating** — v1 Training Operator is on maintenance-only. New features, performance improvements, and JAX/XGBoost/Flux support are only in v2 Trainer.
- **Other frameworks** — PaddleJob is not yet migrated. Pin to specific `training-operator` versions if using PaddlePaddle.
- **Missing scheduling** — v1 jobs don't have native Kueue integration. Use `runPolicy.scheduling.podGroupPolicy` for basic gang-scheduling.