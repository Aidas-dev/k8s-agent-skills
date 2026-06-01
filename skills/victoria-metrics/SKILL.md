---
name: victoria-metrics
description: Route to the correct VictoriaMetrics skill. Use when working with VictoriaMetrics — operator CRDs, queries, cardinality analysis, logs, traces, or trace analysis.
---

# VictoriaMetrics — Skill Router

Pick the right sub-skill based on what the user needs.

## Which Sub-Skill?

| User wants to... | Load skill |
|---|---|
| Create/configure VM Operator CRDs (VMAgent, VMSingle, VMAlert, VMServiceScrape, VLogs, etc.) | `victoriametrics-operator` |
| Query metrics with PromQL/MetricsQL via curl, explore labels/series | `victoriametrics-query` |
| Analyze high cardinality, find optimization opportunities | `victoriametrics-cardinality-analysis` |
| Find unused metrics wasting storage/ingestion | `victoriametrics-unused-metrics-analysis` |
| Search logs with LogsQL, explore log fields/streams | `victorialogs-query` |
| Query traces via Jaeger-compatible API | `victoriatraces-query` |
| Analyze a slow VM query trace JSON | `vm-trace-analyzer` |

## Quick Map

| Task | Skill |
|---|---|
| "Create a VMAgent for my cluster" | `victoriametrics-operator` |
| "Set up VMServiceScrape for my app" | `victoriametrics-operator` |
| "Write a PromQL query for CPU usage" | `victoriametrics-query` |
| "Find metrics with too many labels" | `victoriametrics-cardinality-analysis` |
| "Which metrics are never queried?" | `victoriametrics-unused-metrics-analysis` |
| "Search logs for error in namespace X" | `victorialogs-query` |
| "Find traces for service X" | `victoriatraces-query` |
| "This PromQL query is slow, why?" | `vm-trace-analyzer` |
| "Configure VMAlert with alerting rules" | `victoriametrics-operator` |
| "Set up VLSingle for log storage" | `victoriametrics-operator` |
