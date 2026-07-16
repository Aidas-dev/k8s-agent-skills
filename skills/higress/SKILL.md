---
name: higress
description: Use when working with Higress — AI Gateway, Wasm plugins, Ingress, Helm deployment, or operator CRDs. Triggers: higress, AI Gateway, WasmPlugin, McpBridge.
---

# Higress — Skill Router

Pick the right sub-skill.

## Which Sub-Skill?

| User wants to... | Load skill |
|---|---|
| Manage CRDs (WasmPlugin, Http2Rpc, McpBridge), configure Wasm plugins, AI Gateway, service discovery | `higress-operator` |
| Deploy, configure, upgrade Higress via Helm | `higress-helm` |

## Quick Map

| Task | Skill |
|---|---|
| "Deploy a WasmPlugin for AI proxy" | `higress-operator` |
| "Configure Http2Rpc for Dubbo service" | `higress-operator` |
| "Set up Nacos service discovery with McpBridge" | `higress-operator` |
| "Deploy Higress on Kubernetes with Helm" | `higress-helm` |
| "Configure AI Gateway with OpenAI provider" | `higress-operator` |
| "Set up Gateway API support" | `higress-helm` |
| "Enable Redis for AI caching" | `higress-helm` |
| "Configure OIDC/OAuth via Wasm plugin" | `higress-operator` |
| "Set up Prometheus monitoring" | `higress-helm` |
| "Create a rate-limiting WasmPlugin" | `higress-operator` |
| "Connect to ZooKeeper registry" | `higress-operator` |
| "Enable Wasm plugin server" | `higress-helm` |
