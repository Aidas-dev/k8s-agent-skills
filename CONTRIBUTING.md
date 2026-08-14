# Contributing

## Skill Format

Each skill is a directory with a single `SKILL.md` file following this structure:

```
skills/<name>/
  SKILL.md        # Required: the skill content
```

For router skills with sub-skills:

```
skills/<name>/
  SKILL.md        # Router: table of sub-skills + quick map
skills/<name>-<sub>/
  SKILL.md        # Sub-skill: domain-specific content
```

## Guidelines

- **One concern per skill** — A skill should cover one operator, CRD set, or tool.
- **Router pattern for complex projects** — If a project has both CRDs and Helm charts, split into `-operator` and `-helm` sub-skills.
- **Research before writing** — Check the latest release, CRD API versions, and current best practices.
- **Keep it current** — Include version numbers, release dates, and known issues. Update as projects evolve.
- **Examples are mandatory** — Every operation type should have a real YAML or CLI example.
- **Common mistakes section** — Document pitfalls you discovered during research.

## PR Process

1. Fork or branch
2. Add/update skills following the existing pattern
3. Verify the skill loads with your agent tool: `skill(name="<skill-name>")`
4. Open a PR with a clear description of what the skill covers

## npm Package

This repo is published as `@aidas-dev/k8s-agent-skills` on npm. The `bin/skills-link` script handles symlinking skills to agent directories. See [README](README.md#usage) for details.

## Release Process

**Two paths — both end in the tag push, which is the ONLY thing that publishes.**

### Path A: GitHub UI (recommended, no local git)

Go to Actions → "Publish to npm" → Run workflow → pick `patch`/`minor`/`major`.
The workflow bumps the version, creates the `vX.Y.Z` tag, pushes both — the tag push then triggers publish + Release automatically.

### Path B: local CLI

```bash
# 1. Bump version and create v-prefixed tag (use npm version, NOT git tag directly)
npm version patch -m "chore: bump version to %s"
#    ^ Creates commit + tag vX.Y.Z automatically
#    ^ WARNING: Do NOT add [skip ci] — that suppresses the publish workflow

# 2. Push to both remotes
git push origin main --tags
git push github main --tags
#    ^ GitHub Actions publishes to npm + creates Release automatically
```

### How the publish workflow protects you

- **Tag/version mismatch guard** — if the tag says `v1.9.0` but package.json says `1.8.5`, the workflow fails with a clear error instead of publishing the wrong version.
- **No double-publish** — dispatch runs only bump+tag; only the tag push event actually publishes.
- **No `[skip ci]`** in `npm version` messages — the tag push must trigger the workflows.

### Pitfalls to avoid

| Mistake | Consequence |
|---------|-------------|
| `git tag 1.8.3` (no `v` prefix) | Workflow filter is `tags: 'v*'` — won't trigger |
| `npm version patch -m "msg [skip ci]"` | `[skip ci]` skips ALL workflows including tag-triggered ones |
| `git tag v1.8.5` from wrong commit | Release points to wrong commit. Always use `npm version` which tags HEAD |
| `git tag -d` + `git tag vX.Y.Z` (manual) | Easy to tag wrong commit. Use `npm version` instead.
