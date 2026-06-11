---
name: kubeflow-pipelines
description: Kubeflow Pipelines v2 — Python DSL, IR YAML compilation, @dsl.component, @dsl.pipeline, control flow, Kubernetes Native API, and artifact management.
---

# Kubeflow Pipelines v2

**Repository:** `github.com/kubeflow/pipelines`  
**KFP SDK:** latest ~2.16.0  
**API version:** `pipelines.kubeflow.org/v2beta1` (Kubernetes Native API)  
**Compile target:** IR YAML (not Argo Workflow YAML)

## Architecture

```
Python DSL (@dsl.component, @dsl.pipeline)
     │
     ▼ compile()
IR YAML (intermediate representation)
     │
     ▼ run/create
KFP Backend / Kubernetes Native API
```

## Python SDK

### Installation

```bash
pip install kfp
```

### Components

#### @dsl.component (Python function)

```python
from kfp import dsl

@dsl.component(base_image="python:3.11")
def train_model(
    dataset: dsl.Dataset,
    model: dsl.Output[dsl.Model],
    metrics: dsl.Output[dsl.Metrics],
    epochs: int = 10,
    lr: float = 0.001,
) -> str:
    """Train a model on the input dataset."""
    import json

    # Training logic here
    accuracy = 0.95

    # Output metrics
    metrics.log_metric("accuracy", accuracy)
    metrics.log_metric("loss", 0.023)

    # Model output path
    model.path = "/tmp/model.pkl"

    return f"Model trained with accuracy {accuracy}"
```

#### @dsl.container_component (pre-built container)

```python
@dsl.container_component
def preprocess_data(
    input_data: dsl.Input[dsl.Dataset],
    output_data: dsl.Output[dsl.Dataset],
):
    return dsl.ContainerSpec(
        image="my-registry/preprocessor:latest",
        command=["./preprocess.sh"],
        args=[
            "--input", input_data.uri,
            "--output", output_data.uri,
        ],
    )
```

#### Importer (external artifact)

```python
from kfp import dsl

@dsl.pipeline
def my_pipeline():
    # Import an existing artifact
    data = dsl.importer(
        artifact_uri="s3://bucket/datasets/mnist/",
        artifact_class=dsl.Dataset,
        reimport=False,
    )
```

### Pipelines

```python
@dsl.pipeline(
    name="training-pipeline",
    description="End-to-end model training pipeline",
    pipeline_root="s3://bucket/pipeline-runs/",
)
def training_pipeline(
    epochs: int = 10,
    lr: float = 0.001,
    model_name: str = "resnet50",
):
    # Component invocations
    preprocess_task = preprocess_data(
        input_data=dsl.importer(
            artifact_uri="s3://bucket/raw/",
            artifact_class=dsl.Dataset,
        ).output
    )

    train_task = train_model(
        dataset=preprocess_task.outputs["output_data"],
        epochs=epochs,
        lr=lr,
    )

    # Set resource limits per task
    train_task.set_cpu_limit("8")
    train_task.set_memory_limit("32Gi")
    train_task.set_accelerator_limit("nvidia.com/gpu", 4)
    train_task.set_caching_options(False)
```

### PipelineConfig

```python
from kfp import dsl

@dsl.pipeline
def my_pipeline():
    config = dsl.PipelineConfig(
        timeout_seconds=3600,
    )
    # ... components
```

### Control Flow

```python
from kfp.dsl import Condition, ParallelFor, OneOf, Collected, ExitHandler

@dsl.pipeline
def control_flow_pipeline():
    # Sequential (default)
    a = component_a()
    b = component_b(a.outputs["data"])

    # Conditional branching
    with Condition(b.outputs["score"] > 0.9):
        component_c(b.outputs["data"])

    with Condition(b.outputs["score"] <= 0.9):
        component_d(b.outputs["data"])

    # Parallel iteration
    with ParallelFor(items=["model_a", "model_b", "model_c"]) as model:
        component_e(model=model, data=b.outputs["data"])

    # OneOf (exclusive choice)
    with OneOf():
        component_f()
        component_g()

    # Exit handler (runs on completion or failure)
    with ExitHandler(component_cleanup()):
        main_workflow()

    # Collect results from parallel branches
    collected = Collected()
```

### Secrets and ConfigMaps

```python
from kfp import kubernetes as k8s

@dsl.pipeline
def secure_pipeline():
    task = train_model(...)

    # Add secret as env vars
    k8s.use_secret_as_env(
        task,
        secret_name="training-secrets",
        secret_key_to_env={
            "WANDB_API_KEY": "wandb-api-key",
            "S3_ACCESS_KEY": "aws-access-key",
        },
    )

    # Add ConfigMap as volume
    k8s.use_config_map_as_volume(
        task,
        config_map_name="training-config",
        mount_path="/etc/config",
    )

    # Node selector and tolerations
    task.set_display_name("GPU Training")
    task.add_node_selector("node-type", "gpu-node")
    task.add_toleration("nvidia.com/gpu", "Exists")
```

## Compilation

```bash
# Python DSL → IR YAML
kfp dsl compile --py pipeline.py --output pipeline.yaml
```

Or in Python:

```python
from kfp import compiler
compiler.Compiler().compile(
    pipeline_func=training_pipeline,
    package_path="pipeline.yaml",
)
```

## Execution

### Via KFP Backend

```python
from kfp import client

client = client.Client(host="https://kfp.example.com")
run = client.create_run_from_pipeline_func(
    pipeline_func=training_pipeline,
    arguments={"epochs": 20, "lr": 0.0001},
)
```

### Via Kubernetes Native API (v2beta1)

```yaml
apiVersion: pipelines.kubeflow.org/v2beta1
kind: PipelineRun
metadata:
  name: training-run
spec:
  pipelineSpec:
    pipelineInfo:
      name: training-pipeline
    components:
      preprocess:
        componentRef:
          name: comp-preprocess
        inputDefinitions:
          artifacts:
            input_data:
              artifactType:
                schemaTitle: system.Dataset
      train:
        componentRef:
          name: comp-train-model
        dependentTasks:
          - preprocess
        inputDefinitions:
          parameters:
            epochs:
              parameterType: NUMBER_INTEGER
              defaultValue: 10
            lr:
              parameterType: NUMBER_DOUBLE
              defaultValue: 0.001
    # ... full IR YAML structure
  runtimeConfig:
    parameters:
      epochs: "20"
      lr: "0.0001"
    pipelineRoot: s3://bucket/pipeline-runs/
```

```bash
kubectl apply -f pipeline-run.yaml
```

## IR YAML Format

KFP v2 compiles to IR (Intermediate Representation) YAML, not Argo Workflow YAML. Key sections:

```yaml
# Pipeline definition
pipelineSpec:
  pipelineInfo:
    name: training-pipeline
    description: End-to-end model training
  components:
    comp-preprocess:
      componentRef:
        name: preprocess-data
      inputDefinitions:
        artifacts:
          input_data:
            artifactType:
              schemaTitle: system.Dataset
      outputDefinitions:
        artifacts:
          output_data:
            artifactType:
              schemaTitle: system.Dataset
    comp-train:
      componentRef:
        name: train-model
      dependentTasks:
        - comp-preprocess
      inputDefinitions:
        parameters:
          epochs:
            parameterType: NUMBER_INTEGER
        artifacts:
          dataset:
            taskOutputArtifact:
              producerTask: comp-preprocess
              outputArtifactKey: output_data
  deploymentSpec:
    executors:
      exec-train:
        container:
          image: python:3.11
          command: ["python", "train.py"]
          args: [...]
  root:
    dag:
      tasks:
        comp-preprocess: {}
        comp-train:
          dependentTasks:
            - comp-preprocess
```

## Platform Concepts

| Concept | Description |
|---------|-------------|
| `Artifact` | Immutable typed data artifact (Dataset, Model, Metrics, etc.) |
| `Execution` | Single component run producing artifacts |
| `PipelineRun` | Single execution of a pipeline |
| `Experiment` | Logical grouping of runs |
| `RecurringRun` | Scheduled pipeline execution |
| `PipelineVersion` | Versioned pipeline definition |
| `PipelineRoot` | Root path for artifact storage |
| `Workspace` | Namespace-scoped resource grouping |
| `Importer` | Import non-KFP artifacts into pipeline context |

## Vertex AI Compatibility

KFP v2 pipelines can run on Google Vertex AI:

```python
from google.cloud import aiplatform

aiplatform.init(project="my-project", location="us-central1")

# Run a compiled pipeline
aiplatform.PipelineJob(
    display_name="training-pipeline",
    template_path="pipeline.yaml",
    parameter_values={"epochs": 20, "lr": 0.001},
    enable_caching=True,
).submit()
```

## Best Practices

- **Base images** — Use production images with pinned versions (not `latest`). Prefer `python:3.11-slim` over `python:3.11` for smaller footprints.
- **Caching** — Enable caching with `task.set_caching_options(True)` for fast iteration during development. Disable for training tasks with side effects.
- **Artifact typing** — Always annotate inputs/outputs with `dsl.Dataset`, `dsl.Model`, `dsl.Metrics` for type-safe component wiring.
- **Resource requests** — Set CPU/memory/accelerator limits on each task. Missing GPU limits = scheduling failures.
- **Importer for existing data** — Use `dsl.importer` instead of re-copying data that already exists in object storage.
- **Exit handlers** — Always add cleanup tasks via `ExitHandler` for GPU/TPU resource teardown.
- **Caching with Importer** — Set `reimport=False` on importers to reuse previously imported artifact metadata.

## Common Mistakes

- **Using Argo YAML patterns** — KFP v2 compiles to IR YAML, not Argo Workflow YAML. Don't write Argo templates directly.
- **Missing dsl.Output annotations** — If a component produces an artifact, the parameter must be typed `dsl.Output[dsl.Model]`, not a plain return value.
- **Lightweight components at import** — Python function imports happen at compile time, not runtime. All imports must be inside the function body.
- **Container component args** — `@dsl.container_component` arguments are only available as `dsl.ContainerSpec` args, not in Python function body.
- **Pipeline root permissions** — The pipeline root (S3/GCS/MinIO) must be accessible by the KFP backend, not just the training pods.
- **KFP SDK version mismatch** — The SDK compiling the pipeline and the KFP backend version must be compatible. SDK 2.16.0 requires backend ≥ 2.16.0.
- **Kubernetes Native API lack of v1beta1** — Only `pipelines.kubeflow.org/v2beta1` is available. Don't look for v1.
- **ParallelFor limits** — Deeply nested `ParallelFor` can produce thousands of tasks. Set reasonable iteration bounds.
