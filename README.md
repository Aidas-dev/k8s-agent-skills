# k8s-agent-skills

[![npm version](https://img.shields.io/npm/v/k8s-agent-skills?logo=npm)](https://www.npmjs.com/package/k8s-agent-skills)
[![license](https://img.shields.io/github/license/Aidas-dev/k8s-agent-skills)](LICENSE)
[![Publish](https://github.com/Aidas-dev/k8s-agent-skills/actions/workflows/publish.yml/badge.svg)](https://github.com/Aidas-dev/k8s-agent-skills/actions/workflows/publish.yml)
[![GitHub stars](https://img.shields.io/github/stars/Aidas-dev/k8s-agent-skills?style=flat)](https://github.com/Aidas-dev/k8s-agent-skills)

Agent skills for Kubernetes cluster operations tooling. Each skill is a self-contained `SKILL.md` designed for agentic AI tools (OpenCode, Claude Code, Codex, Cursor) that load skills for task-specific expertise.

Designed for: **Cilium, Talos, Flux, Rook-Ceph, CNPG, ZITADEL, Gitea, Tekton, Cert-Manager, VictoriaMetrics, Harbor, Higress, KServe, Kubeflow, Vault, ExternalDNS, External Secrets, Dragonfly, Flagger, MariaDB, Vector, Sealed Secrets, Stakater Reloader, NVIDIA GPU, Atlas, Nextcloud** and more.

## Quick Start

```bash
# Install
npm install --save-dev k8s-agent-skills

# Skills auto-link to ~/.agents/skills/ + ~/.config/opencode/skills/ + ~/.claude/skills/
# + ~/.codex/skills/ + ~/.cursor/skills/ on install.
# Just restart your AI agent — skills are ready to use.

# To manually re-link (e.g. after npm update):
npx skills-link --global

# Check all skills are linked:
npx skills-link --global --check
```

Skills use **symlinks**, so content always reflects the latest package version. `npm update` + automatic `postinstall` = skills stay current.

### Per-agent linking (manual)

```bash
npx skills-link               # ~/.agents/skills/ only (default)
npx skills-link --opencode    # ~/.config/opencode/skills/
npx skills-link --claude      # ~/.claude/skills/
npx skills-link --codex       # ~/.codex/skills/
npx skills-link --cursor      # ~/.cursor/skills/
npx skills-link --dry-run     # preview without linking
```

## Skills

| Skill | What it covers | CRDs |
|-------|---------------|------|
| [atlas](skills/atlas/SKILL.md) | Atlas Operator — DB schema migrations, lint, policies | AtlasSchema, AtlasMigration |
| [cert-manager](skills/cert-manager/SKILL.md) | TLS cert provisioning, ACME, Issuers, Gateway integration | Certificate, Issuer, ClusterIssuer |
| [cilium-gateway](skills/cilium-gateway/SKILL.md) | Gateway API, TLS, traffic splitting, oauth2-proxy, hostNetwork | GatewayClass, Gateway, HTTPRoute, etc. |
| [cilium-network](skills/cilium-network/SKILL.md) | Cilium CNI, network policies, LB IPAM, encryption, Hubble | CiliumNetworkPolicy, CiliumCIDRGroup, etc. |
| [cnpg](skills/cnpg/SKILL.md) | CloudNativePG — PostgreSQL clusters, backups, poolers | Cluster, Backup, ScheduledBackup, Pooler |
| [dragonfly](skills/dragonfly/SKILL.md) | DragonflyDB — Redis-compatible operator, replication, TLS | Dragonfly |
| [external-dns](skills/external-dns/SKILL.md) | DNS sync — Cloudflare, Route53, Gateway API, sources, registry | None |
| [external-secrets](skills/external-secrets/SKILL.md) | External Secrets Operator — sync K8s Secrets from Vault, AWS, GCP, Azure | SecretStore, ClusterSecretStore, ExternalSecret, PushSecret |
| [flagger](skills/flagger/SKILL.md) | Progressive delivery, canary, A/B, blue/green | Canary, MetricTemplate, AlertProvider |
| [flux](skills/flux/SKILL.md) | Flux CD router — debugging, CRDs, repo audit | Router → sub-skills |
| [gitea](skills/gitea/SKILL.md) | Gitea router — API, runner, registry, webhooks, tea CLI | Router → sub-skills |
| [gitea-api](skills/gitea-api/SKILL.md) | Gitea REST API — auth, repos, issues, PRs, packages | None |
| [gitea-registry](skills/gitea-registry/SKILL.md) | Gitea container registry — OCI, multi-arch, push/pull | None |
| [gitea-runner](skills/gitea-runner/SKILL.md) | Gitea Actions runners — registration, host-mode, ephemeral | None |
| [gitea-tea](skills/gitea-tea/SKILL.md) | tea CLI — commands, auth, actions, webhooks, admin | None |
| [gitea-webhooks](skills/gitea-webhooks/SKILL.md) | Gitea webhooks — events, HMAC, org vs repo hooks | None |
| [harbor](skills/harbor/SKILL.md) | Harbor router — API, Helm, Terraform | Router → sub-skills |
| [harbor-api](skills/harbor-api/SKILL.md) | Harbor REST API v2 — projects, artifacts, robots, replication, GC, OIDC | None |
| [harbor-helm](skills/harbor-helm/SKILL.md) | Harbor Helm chart — production deploy, external DB/Redis/S3, Trivy | None |
| [harbor-terraform](skills/harbor-terraform/SKILL.md) | Harbor Terraform provider — 20 resources, 8 data sources | None |
| [higress](skills/higress/SKILL.md) | Higress router — CRDs, Wasm plugins, AI Gateway, Helm | Router → sub-skills |
| [higress-helm](skills/higress-helm/SKILL.md) | Higress Helm chart — core/console/redis/plugin-server/o11y | None |
| [higress-operator](skills/higress-operator/SKILL.md) | Higress CRDs — WasmPlugin, Http2Rpc, McpBridge, 41 Wasm plugins, 16 AI providers | WasmPlugin, Http2Rpc, McpBridge |
| [kserve](skills/kserve/SKILL.md) | KServe router — CRDs, Helm, deployment modes | Router → sub-skills |
| [kserve-helm](skills/kserve-helm/SKILL.md) | KServe Helm — 10 charts, Serverless/Raw/ModelMesh modes | None |
| [kserve-operator](skills/kserve-operator/SKILL.md) | KServe CRDs — InferenceService, ServingRuntime, InferenceGraph, LLMInferenceService, LocalModel | 22 CRDs under serving.kserve.io |
| [kubeflow](skills/kubeflow/SKILL.md) | Kubeflow router — Trainer v2, Pipelines v2, Training Operator v1 | Router → sub-skills |
| [kubeflow-pipelines](skills/kubeflow-pipelines/SKILL.md) | KFP v2 SDK — DSL, IR YAML, control flow, Kubernetes Native API | PipelineRun (v2beta1) |
| [kubeflow-trainer](skills/kubeflow-trainer/SKILL.md) | Kubeflow Trainer v2.2 — TrainJob, TrainingRuntime, 5 ML policies | TrainJob, TrainingRuntime, ClusterTrainingRuntime |
| [kubeflow-training-operator](skills/kubeflow-training-operator/SKILL.md) | Legacy v1 — PyTorchJob, TFJob, MPIJob, XGBoostJob | PyTorchJob, TFJob, MPIJob, XGBoostJob |
| [mariadb](skills/mariadb/SKILL.md) | MariaDB router — operator CRDs, Helm | Router → sub-skills |
| [mariadb-helm](skills/mariadb-helm/SKILL.md) | MariaDB operator Helm — 3 charts, production HA values | None |
| [mariadb-operator](skills/mariadb-operator/SKILL.md) | MariaDB operator — 12 CRDs, Galera HA, MaxScale, backups, PITR | 12 CRDs under k8s.mariadb.com |
| [nextcloud](skills/nextcloud/SKILL.md) | Nextcloud router — Helm deployment, Provisioning API | Router → sub-skills |
| [nextcloud-helm](skills/nextcloud/nextcloud-helm/SKILL.md) | Nextcloud Helm chart — NC34, S3, Imaginary, Collabora, cron | None |
| [nextcloud-api](skills/nextcloud/nextcloud-api/SKILL.md) | Nextcloud Provisioning API — users, groups, apps | None |
| [nvidia-device-plugin](skills/nvidia-device-plugin/SKILL.md) | GPU discovery, GFD, NFD, CDI, MIG, time-slicing | None (ConfigMap) |
| [rook-ceph](skills/rook-ceph/SKILL.md) | Rook-Ceph router — operator CRDs, toolbox CLI | Router → sub-skills |
| [rook-ceph-operator](skills/rook-ceph-operator/SKILL.md) | Ceph cluster, block pools, object store, NFS, CSI | CephCluster, CephBlockPool, CephObjectStore, etc. |
| [rook-ceph-toolbox](skills/rook-ceph-toolbox/SKILL.md) | Ceph CLI — health, OSD mgmt, RBD, RGW, CRUSH | None (toolbox ops) |
| [sealed-secrets](skills/sealed-secrets/SKILL.md) | Encrypted Secrets for GitOps, kubeseal, key rotation | SealedSecret |
| [stakater-reloader](skills/stakater-reloader/SKILL.md) | ConfigMap/Secret reload, annotations, Helm values | None (annotation-based) |
| [talos](skills/talos/SKILL.md) | Talos Linux — cluster deploy, machine config, upgrades, talosctl | None |
| [tekton](skills/tekton/SKILL.md) | Tekton pipelines — resolver refs, matrix, CEL, TTL | Task, Pipeline, etc. |
| [vault](skills/vault/SKILL.md) | Vault router — Helm, API, Terraform | Router → sub-skills |
| [vault-api](skills/vault-api/SKILL.md) | Vault REST API — health, init, unseal, auth, KV, policies | None |
| [vault-helm](skills/vault-helm/SKILL.md) | Vault Helm chart — HA+Raft, injector, storage, TLS, telemetry | None |
| [vault-terraform](skills/vault-terraform/SKILL.md) | Vault Terraform provider — 188+ resources, auth backends, KV, policies, identity | None |
| [vector](skills/vector/SKILL.md) | Vector router — Helm, operator CRDs | Router → sub-skills |
| [vector-helm](skills/vector-helm/SKILL.md) | Vector Helm chart — 3 roles (Agent/Aggregator/Stateless), customConfig | None |
| [vector-operator](skills/vector-operator/SKILL.md) | Vector operator — 5 CRDs, auto-routing by source type | Vector, VectorPipeline, ClusterVectorPipeline, VectorAggregator, ClusterVectorAggregator |
| [victoria-metrics](skills/victoria-metrics/SKILL.md) | VM skill router — operator, queries, cardinality, logs, traces | Router |
| [victoriametrics-operator](skills/victoriametrics-operator/SKILL.md) | VM Operator CRDs — VMAgent, VMAlert, VMServiceScrape, VLogs | 19 CRDs |
| [zitadel](skills/zitadel/SKILL.md) | ZITADEL router — API, Helm, Terraform | Router → sub-skills |
| [zitadel-api](skills/zitadel-api/SKILL.md) | ZITADEL API — OIDC/SAML/API apps, users, orgs, roles | None |
| [zitadel-helm](skills/zitadel-helm/SKILL.md) | ZITADEL Helm — CNPG, Gateway API, caches, masterkey | None |
| [zitadel-terraform](skills/zitadel-terraform/SKILL.md) | ZITADEL Terraform provider — 80+ resources, 40+ data sources | None |

## How It Works

Skills are **symlinked** from the npm package to your agent's skill directory:

```
node_modules/k8s-agent-skills/skills/cilium-network/SKILL.md
  → ~/.agents/skills/cilium-network/SKILL.md  (OpenCode)
  → ~/.config/opencode/skills/cilium-network/  (OpenCode config)
  → ~/.claude/skills/cilium-network/            (Claude Code)
  → ~/.codex/skills/cilium-network/             (Codex CLI)
  → ~/.cursor/skills/cilium-network/            (Cursor)
```

This means:
- **Skill content updates instantly** on `npm update` — symlinks are live
- **New skills auto-link** via `postinstall` script on package install
- **Stale symlinks** (skills removed from a release) are cleaned automatically
- Run `npx skills-link --global --check` any time to verify

## Manual Installation

```bash
# git clone (if not using npm)
git clone https://github.com/Aidas-dev/k8s-agent-skills.git
cd k8s-agent-skills
./bin/skills-link --global

# Or symlink manually
ln -sf $(pwd)/skills/* ~/.agents/skills/
```

## npm Publishing

Auto-publishes on `v*` tag push via GitHub Actions with OIDC Trusted Publisher — no tokens needed.

```bash
npm version patch  # or minor / major
git push origin main --tags
git push github main --tags
```

Or manually: `git tag vX.Y.Z && git push origin main --tags && git push github main --tags`.

## License

[MIT](LICENSE) — free to use, modify, and distribute.
