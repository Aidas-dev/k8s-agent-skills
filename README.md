# k8s-agent-skills

Agent skills for Kubernetes cluster operations tooling. Each skill is a self-contained `SKILL.md` designed for agentic AI tools (Claude Code, OpenCode, Codex) that load skills for task-specific expertise.

## Usage

### Via npm/bun (recommended)

```bash
# Install as dev dependency
bun add -d k8s-agent-skills
# or
npm install --save-dev k8s-agent-skills

# Symlink skills to agent config dir
npx skills-npm
# → symlinks skills/ into ~/.agents/skills/
```

### Via git clone

```bash
git clone https://git.kubexa.tech/Aidas/k8s-agent-skills.git
ln -sf $(pwd)/k8s-agent-skills/skills/* ~/.agents/skills/
```

### Via Gitea npm registry

```bash
bun add -d @aidmantas/k8s-agent-skills@https://git.kubexa.tech/Aidas/k8s-agent-skills.git
npx skills-npm
```

### Manual copy

```bash
# OpenCode / Codex
cp -r skills/external-dns ~/.agents/skills/

# Claude Code Desktop
cp -r skills/flagger ~/.claude/skills/
```

## Skills

| Skill | What it covers | CRDs |
|-------|---------------|------|
| [external-dns](skills/external-dns/SKILL.md) | DNS sync — Cloudflare, Route53, Gateway API, sources, registry | None |
| [nvidia-device-plugin](skills/nvidia-device-plugin/SKILL.md) | GPU discovery, GFD, NFD, CDI, MIG, time-slicing | None (ConfigMap) |
| [sealed-secrets](skills/sealed-secrets/SKILL.md) | Encrypted Secrets for GitOps, kubeseal, key rotation | SealedSecret |
| [flagger](skills/flagger/SKILL.md) | Progressive delivery, canary, A/B, blue/green | Canary, MetricTemplate, AlertProvider |
| [cilium-network](skills/cilium-network/SKILL.md) | Cilium CNI, network policies, LB IPAM, encryption, Hubble | CiliumNetworkPolicy, CiliumCIDRGroup, etc. |
| [cilium-gateway](skills/cilium-gateway/SKILL.md) | Gateway API, TLS, traffic splitting, oauth2-proxy, hostNetwork | GatewayClass, Gateway, HTTPRoute, etc. |
| [talos](skills/talos/SKILL.md) | Talos Linux — cluster deploy, machine config, upgrades, talosctl | None |
| [flux](skills/flux/SKILL.md) | Flux CD router — debugging, CRDs, repo audit | Router → sub-skills |
| [tekton](skills/tekton/SKILL.md) | Tekton pipelines — resolver refs, matrix, CEL, TTL | Task, Pipeline, etc. |
| [cnpg](skills/cnpg/SKILL.md) | CloudNativePG — PostgreSQL clusters, backups, poolers | Cluster, Backup, ScheduledBackup, Pooler |
| [atlas](skills/atlas/SKILL.md) | Atlas Operator — DB schema migrations, lint, policies | AtlasSchema, AtlasMigration |
| [dragonfly](skills/dragonfly/SKILL.md) | DragonflyDB — Redis-compatible operator, replication, TLS | Dragonfly |
| [cert-manager](skills/cert-manager/SKILL.md) | TLS cert provisioning, ACME, Issuers, Gateway integration | Certificate, Issuer, ClusterIssuer |
| [rook-ceph-operator](skills/rook-ceph-operator/SKILL.md) | Ceph cluster, block pools, object store, NFS, CSI | CephCluster, CephBlockPool, CephObjectStore, etc. |
| [rook-ceph-toolbox](skills/rook-ceph-toolbox/SKILL.md) | Ceph CLI — health, OSD mgmt, RBD, RGW, CRUSH | None (toolbox ops) |
| [victoriametrics-operator](skills/victoriametrics-operator/SKILL.md) | VM Operator CRDs — VMAgent, VMAlert, VMServiceScrape, VLogs | 19 CRDs |
| [victoria-metrics](skills/victoria-metrics/SKILL.md) | VM skill router → operator, queries, cardinality, logs, traces | Router |
| [stakater-reloader](skills/stakater-reloader/SKILL.md) | ConfigMap/Secret reload, annotations, Helm values | None (annotation-based) |
| [gitea](skills/gitea/SKILL.md) | Gitea router → API, runner, registry, webhooks | Router |
| [gitea-api](skills/gitea-api/SKILL.md) | Gitea REST API — auth, repos, issues, PRs, packages | None |
| [gitea-runner](skills/gitea-runner/SKILL.md) | Gitea Actions runners — registration, host-mode, ephemeral | None |
| [gitea-registry](skills/gitea-registry/SKILL.md) | Gitea container registry — OCI, multi-arch, push/pull | None |
| [gitea-webhooks](skills/gitea-webhooks/SKILL.md) | Gitea webhooks — events, HMAC, org vs repo hooks | None |
| [gitea-tea](skills/gitea-tea/SKILL.md) | tea CLI — commands, auth, actions, webhooks, admin | None |
| [zitadel](skills/zitadel/SKILL.md) | ZITADEL router → API + Helm deployment | Router |
| [zitadel-api](skills/zitadel-api/SKILL.md) | ZITADEL API — OIDC/SAML/API apps, users, orgs, roles | None |
| [zitadel-helm](skills/zitadel-helm/SKILL.md) | ZITADEL Helm — CNPG, Gateway API, caches, masterkey | None |

## License

MIT — free to use, modify, and distribute.
