---
name: kubeflow
description: Use when working with Kubeflow — route to the correct sub-skill based on what the user needs: TrainJob CRDs, Kubeflow Pipelines v2, or legacy Training Operator v1.
---

# Kubeflow — Skill Router

Pick the right sub-skill.

## Which Sub-Skill?

| User wants to... | Load skill |
|---|---|
| Manage TrainJob/TrainingRuntime/ClusterTrainingRuntime CRDs (v2.2) | `kubeflow-trainer` |
| Write KFP v2 pipelines with DSL, compile to IR YAML, orchestrate runs | `kubeflow-pipelines` |
| Manage legacy PyTorchJob/TFJob/MPIJob/XGBoostJob CRDs (v1) | `kubeflow-training-operator` |

## Quick Map

| Task | Skill |
|---|---|
| "Create a multi-node PyTorch TrainJob" | `kubeflow-trainer` |
| "Deploy a TrainingRuntime for JAX" | `kubeflow-trainer` |
| "Integrate TrainJob with Kueue for gang-scheduling" | `kubeflow-trainer` |
| "Write a component with @dsl.component" | `kubeflow-pipelines` |
| "Compile a pipeline to IR YAML" | `kubeflow-pipelines` |
| "Run a KFP pipeline via Kubernetes Native API" | `kubeflow-pipelines` |
| "Use ParallelFor and Condition in a pipeline" | `kubeflow-pipelines` |
| "Create a PyTorchJob for distributed training" | `kubeflow-training-operator` |
| "Migrate from v1 Training Operator to v2 Trainer" | `kubeflow-training-operator` |
| "Set up TFJob with TFConfig environment" | `kubeflow-training-operator` |
| "Configure RuntimePatches for container overrides" | `kubeflow-trainer` |
| "Use Importer to pass artifacts between pipelines" | `kubeflow-pipelines` |
