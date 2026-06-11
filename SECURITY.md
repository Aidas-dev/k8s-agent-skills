# Security Policy

## Reporting a Vulnerability

This project provides agent skill definitions (SKILL.md files) — not production code.

If you find a security issue in a skill's recommended configuration (e.g., insecure defaults, exposed credentials, unsafe RBAC patterns), please open an issue or PR with the fix.

For critical vulnerabilities in the underlying projects documented by these skills, please report directly to the respective project maintainers.

## Supported Versions

| Version | Supported |
|---------|-----------|
| Latest npm release | ✅ |
| Main branch | ✅ |
| Older tags | ❌ |

## Safe Practices

Skills document Kubernetes operator configurations. Always:

- Review generated configurations before applying
- Use secrets management (external Secrets operators, not plaintext)
- Enable TLS where documented as optional
- Follow the principle of least privilege for RBAC
