# KServe — Skill Router

Pick the right sub-skill.

## Which Sub-Skill?

| User wants to... | Load skill |
|---|---|
| Manage CRDs (InferenceService, ServingRuntime, InferenceGraph, LLMInferenceService, LocalModelNode), configure predictors, storage, transformers, explainers | `kserve-operator` |
| Deploy, configure, upgrade KServe via Helm (10 charts, deployment modes) | `kserve-helm` |

## Quick Map

| Task | Skill |
|---|---|
| "Deploy a sklearn InferenceService with S3 model" | `kserve-operator` |
| "Configure a multi-node LLM serving with vLLM" | `kserve-operator` |
| "Create a ServingRuntime for custom Triton setup" | `kserve-operator` |
| "Set up an InferenceGraph ensemble pipeline" | `kserve-operator` |
| "Configure S3/GCS/HF storage credentials" | `kserve-operator` |
| "Enable LocalModelCache for NVMe model caching" | `kserve-operator` |
| "Deploy KServe on Kubernetes with Helm" | `kserve-helm` |
| "Configure Standard vs Knative deployment mode" | `kserve-helm` |
| "Install KServe CRDs only" | `kserve-helm` |
| "Set up LLMInferenceService with disaggregated prefill/decode" | `kserve-operator` |
| "Add a transformer for pre/post-processing" | `kserve-operator` |
| "Create a canary rollout for model update" | `kserve-operator` |
| "Configure Gateway API ingress" | `kserve-helm` |
