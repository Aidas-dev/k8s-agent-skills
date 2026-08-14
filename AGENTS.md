# AGENTS.md — k8s-agent-skills

## Skills Must Be User-Agnostic

Every skill in this repo is published to npm and consumed by agents of **any** user, on **any** cluster. Skills must work standalone for anyone, anywhere.

### Never hardcode cluster-specific data

Forbidden in skills:

- Node names, IPs, tailnet addresses, hostnames (`controlplane-svt`, `100.x.x.x`, `talos-cluster`)
- Cluster names, endpoints, API URLs, domain names
- Secrets, tokens, certs, private keys
- Node counts, hardware specs, disk layouts of a specific deployment
- Provider account IDs, tenant IDs, region names specific to one infra

### What to write instead

- Placeholder patterns: `<node-ip>`, `<cluster-name>`, `https://<endpoint>:6443`
- Generic role labels: `cp1/cp2/cp3`, `controlplane`/`worker`
- "Source of truth" references when a real repo pattern is genuinely instructive — but abstract the specifics (e.g. "secrets come from the live cluster, stored gitignored" — not the actual JSON path of one deployment)

### If a real-world example adds value

Genericize it: describe the pattern, keep the structure, strip the specifics. The reader's cluster is different — the skill must survive that.

### Review gate

Before committing any skill, grep for: IP literals, tailnet `100.x` ranges, node hostnames, cluster names, and URLs pointing at personal infrastructure. If any present — rewrite before committing.
