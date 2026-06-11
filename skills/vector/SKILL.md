# Vector — Skill Router

Pick the right sub-skill.

## Which Sub-Skill?

| User wants to... | Load skill |
|---|---|
| Deploy Vector on K8s via Helm (Agent/Aggregator/Stateless-Aggregator) | `vector-helm` |
| Manage Vector with operator CRDs (Vector, VectorPipeline, VectorAggregator) | `vector-operator` |

## Quick Map

| Task | Skill |
|---|---|
| "Deploy Vector as DaemonSet on all nodes" | `vector-helm` |
| "Configure Vector pipeline with customConfig" | `vector-helm` |
| "Set up Vector aggregator as StatefulSet" | `vector-helm` |
| "Deploy kaasops/vector-operator with CRDs" | `vector-operator` |
| "Create a VectorPipeline for kubernetes_logs" | `vector-operator` |
| "Create a ClusterVectorPipeline for cluster-wide sources" | `vector-operator` |
| "Set up VectorAggregator for centralized processing" | `vector-operator` |
| "Configure Prometheus PodMonitor for Vector" | `vector-operator` |
| "Use VRL transforms in a VectorPipeline" | `vector-operator` |
