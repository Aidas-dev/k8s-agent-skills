---
name: gitea-registry
description: Use when working with the Gitea container registry — OCI/Docker v2 API, push/pull images, multi-arch builds, package management via API, and configuration for supported package types.
---

# Gitea Container Registry (v1.26)

OCI-compatible registry using Docker Registry API v2. Supports Docker images, Helm charts, and all OCI artifacts.

## Auth

```bash
docker login git.example.com
# Username: gitea username
# Password: personal access token (NOT account password for 2FA users)
```

## Image Naming

```
{registry}/{owner}/{image}:{tag}
```

Examples: `git.example.com/testuser/myapp:latest`, `git.example.com/myorg/backend:v1.2.3`

## Push / Pull

```bash
docker tag myapp:latest git.example.com/testuser/myapp:latest
docker push git.example.com/testuser/myapp:latest

docker pull git.example.com/testuser/myapp:latest
```

## Multi-Arch

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  -t git.example.com/testuser/myapp:latest --push .
```

## API

```bash
# List packages for owner
curl -H "Authorization: token <token>" \
  https://git.example.com/api/v1/packages/testuser

# Delete package version
curl -X DELETE -H "Authorization: token <token>" \
  https://git.example.com/api/v1/packages/testuser/container/myapp/1.0.0
```

## Supported Package Types

alpine, cargo, chef, composer, conan, conda, container, cran, debian, generic, go, helm, maven, npm, nuget, pub, pypi, rpm, rubygems, swift, vagrant, **terraform** (v1.26+).

## Config

```ini
[packages]
ENABLED = true
LIMIT_SIZE_CONTAINER = -1  # No size limit for container images
```

## Common Mistakes

- **Tag case** — Tags are case-insensitive. `image:Tag` and `image:tag` are the same image.
- **Auth for 2FA users** — Always use a personal access token as password, not account password.
- **Delete needs all segments** — REST delete endpoint requires owner, type, name, and version.
- **Helm chart push** — Use `helm push` with OCI format: `helm push mychart-1.0.0.tgz oci://git.example.com/myorg`.
