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
