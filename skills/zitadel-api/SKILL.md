---
name: zitadel-api
description: >
  ZITADEL identity management platform (v4 GA, v2 API). Covers resource-based API for
  managing organizations, projects, applications (OIDC/SAML/API), roles, users, and
  OIDC settings via Connect RPC (HTTP/1.1 JSON) and REST endpoints. PAT or
  JWT assertion auth.
triggers:
  - zitadel api
  - oidc app
  - zitadel app creation
  - zitadel user
  - auth api
  - zitadel oidc
  - zitadel saml
---

# ZITADEL API v2 (v4 GA)

Official docs: https://zitadel.com/docs/apis/v2

## Architecture

```
ZITADEL Instance
  └── Organizations (multi-tenant units)
        └── Projects (group apps + roles)
              ├── Applications (OIDC / SAML / API clients)
              │     └── Application Keys (for API apps)
              └── Roles (custom keys: admin/viewer/editor/etc.)
                    └── Grant Assignments (user-to-role mapping)
```

## Authentication

### Who Is the Client?

All instance-level operations (create org, create project, create app, create user, role grants) require the **ZITADEL IAM admin service account**. This is the system's own admin user, not a regular org user. In this cluster, the credentials are stored in the `zitadel` namespace:

- `iam-admin-pat` — Personal Access Token (simpler, use first)
- `iam-admin-jwt` — Private Key JWT (use when PAT doesn't have enough permissions)

A regular user's PAT (from Acme, Family, or any org) can only manage resources within their own org and project. It cannot create orgs, projects, or users across the instance.

### Personal Access Token (PAT)

Simplest auth. Get from the IAM admin secret:

```bash
export PAT=$(kubectl get secret iam-admin-pat -n zitadel -o jsonpath='{.data.pat}' | base64 -d)
```

**Headers for all API calls:**
```bash
-H "Authorization: Bearer ${PAT}" \
-H "Content-Type: application/json" \
-H "Connect-Protocol-Version: 1"
```

### Service Account JWT (Private Key)

Use when PAT returns 403 on instance-level operations. Generates a fresh access token via JWT assertion (RFC 7523).

```bash
# Get service account key
kubectl get secret -n zitadel iam-admin-jwt -o jsonpath='{.data.private-key\.json}' | base64 -d > /tmp/sa-jwt.json

# Exchange JWT assertion for access token
SA_TOKEN=$(python3 <<'PYEOF'
import json, time, jwt

with open('/tmp/sa-jwt.json') as f:
    sa = json.load(f)

now = int(time.time())
assertion = jwt.encode({
    'iss': sa['userId'],
    'sub': sa['userId'],
    'aud': 'https://auth.example.com/oauth/v2/token',
    'iat': now,
    'exp': now + 3600
}, sa['key'], algorithm='RS256')
print(assertion)
PYEOF
)

TOKEN=$(curl -s -X POST "https://auth.example.com/oauth/v2/token" \
  -d "grant_type=client_credentials" \
  -d "scope=urn:zitadel:iam:org:project:id:zitadel:aud" \
  -d "client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer" \
  -d "client_assertion=$SA_TOKEN" | jq -r '.access_token')

echo "$TOKEN"
```

Use the resulting `$TOKEN` with the same headers (Bearer auth, Content-Type JSON, Connect-Protocol-Version 1).

### Direct DB Access (Emergency Fallback)

When API access is unavailable (e.g., org was deleted, admin permissions lost), query ZITADEL's PostgreSQL database directly:

```bash
# Connect to ZITADEL CNPG database
kubectl exec -n zitadel deploy/zitadel -- psql -h localhost -d zitadel -U zitadel

# Get all OIDC app client IDs and secrets
SELECT 
  a.id AS app_id,
  c.client_id,
  c.client_secret
FROM projections.apps7 AS a
JOIN projections.apps7_oidc_configs AS c ON a.id = c.app_id;

# Get org details
SELECT id, name, state FROM projections.orgs;

# Get user details
SELECT id, username, email, state FROM projections.users7;

# Get active OIDC apps with full config
SELECT 
  a.id, 
  a.name,
  a.project_id, 
  c.client_id, 
  c.client_secret,
  c.redirect_uris,
  c.post_logout_redirect_uris
FROM projections.apps7 AS a
JOIN projections.apps7_oidc_configs AS c ON a.id = c.app_id
JOIN projections.apps7_projects AS p ON a.project_id = p.project_id
WHERE a.state = 1;
```

## Common Operations

> All operations below use `$PAT` which is the **IAM admin PAT** from `kubectl get secret iam-admin-pat -n zitadel`. See [Authentication](#authentication) for setup.

### Organizations

#### 1. List Organizations
```bash
curl -s -X POST "https://auth.example.com/zitadel.org.v2.OrganizationService/ListOrganizations" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{}' | jq
```

#### 2. Create Organization
```bash
curl -s -X POST "https://auth.example.com/zitadel.org.v2.OrganizationService/CreateOrganization" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{"name": "Family"}' | jq
```
Returns `orgId` in response.

### Projects

#### 3. List Projects
```bash
curl -s -X POST "https://auth.example.com/zitadel.project.v2.ProjectService/ListProjects" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{}' | jq
```

#### 4. Create Project
```bash
curl -s -X POST "https://auth.example.com/zitadel.project.v2.ProjectService/CreateProject" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{
    "projectName": "Family-Apps"
  }' | jq
```

Note: `organizationId` is NOT required in v2 body for org-scoped operations. The PAT's org scope is used automatically. For cross-org operations, include `organizationId` in body.

### Applications (OIDC / SAML / API)

**IMPORTANT: The service name changed!** In v4 GA, the endpoint is:
- `zitadel.application.v2.ApplicationService/CreateApplication` (NOT `zitadel.app.v2.AppService/AddApp`)
- All app methods consolidated under `ApplicationService`.
- App type determined by which config field you include: `oidcConfiguration`, `samlConfiguration`, or `apiConfiguration`.

#### 5. Create OIDC Application (Authorization Code + PKCE)
```bash
curl -s -X POST "https://auth.example.com/zitadel.application.v2.ApplicationService/CreateApplication" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{
    "projectId": "123456789012345678",
    "name": "Immich",
    "oidcConfiguration": {
      "redirectUris": [
        "https://immich.example.com/auth/login",
        "https://immich.example.com/user-settings",
        "app.immich:///oauth-callback"
      ],
      "responseTypes": ["OIDC_RESPONSE_TYPE_CODE"],
      "grantTypes": [
        "OIDC_GRANT_TYPE_AUTHORIZATION_CODE",
        "OIDC_GRANT_TYPE_REFRESH_TOKEN"
      ],
      "applicationType": "OIDC_APP_TYPE_USER_AGENT",
      "authMethodType": "OIDC_AUTH_METHOD_TYPE_NONE",
      "postLogoutRedirectUris": [
        "https://immich.example.com"
      ],
      "version": "OIDC_VERSION_1_0",
      "accessTokenType": "OIDC_TOKEN_TYPE_BEARER",
      "developmentMode": false
    }
  }' | jq
```

**Enum reference for OIDC config:**

| Field | Enum Values | Description |
|-------|------------|-------------|
| `responseTypes` | `OIDC_RESPONSE_TYPE_CODE=1`, `OIDC_RESPONSE_TYPE_ID_TOKEN=2`, `OIDC_RESPONSE_TYPE_ID_TOKEN_TOKEN=3` | What flows return |
| `grantTypes` | `OIDC_GRANT_TYPE_AUTHORIZATION_CODE=0`, `OIDC_GRANT_TYPE_IMPLICIT=1`, `OIDC_GRANT_TYPE_REFRESH_TOKEN=2`, `OIDC_GRANT_TYPE_DEVICE_CODE=3`, `OIDC_GRANT_TYPE_TOKEN_EXCHANGE=4` | Allowed flows |
| `applicationType` | `OIDC_APP_TYPE_WEB=0`, `OIDC_APP_TYPE_USER_AGENT=1`, `OIDC_APP_TYPE_NATIVE=2` | Client type |
| `authMethodType` | `OIDC_AUTH_METHOD_TYPE_BASIC=0` (client_secret_basic), `OIDC_AUTH_METHOD_TYPE_POST=1` (client_secret_post), `OIDC_AUTH_METHOD_TYPE_NONE=2` (PKCE), `OIDC_AUTH_METHOD_TYPE_PRIVATE_KEY_JWT=3` | Token endpoint auth |
| `accessTokenType` | `OIDC_TOKEN_TYPE_BEARER=0`, `OIDC_TOKEN_TYPE_JWT=1` | Token format |

#### 6. Create OIDC Application (Client Secret Basic — for oauth2-proxy)
```bash
curl -s -X POST "https://auth.example.com/zitadel.application.v2.ApplicationService/CreateApplication" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{
    "projectId": "123456789012345678",
    "name": "MLflow",
    "oidcConfiguration": {
      "redirectUris": [
        "https://mlflow.example.com/oauth2/callback"
      ],
      "responseTypes": ["OIDC_RESPONSE_TYPE_CODE"],
      "grantTypes": [
        "OIDC_GRANT_TYPE_AUTHORIZATION_CODE",
        "OIDC_GRANT_TYPE_REFRESH_TOKEN"
      ],
      "applicationType": "OIDC_APP_TYPE_WEB",
      "authMethodType": "OIDC_AUTH_METHOD_TYPE_BASIC",
      "postLogoutRedirectUris": [
        "https://mlflow.example.com"
      ],
      "version": "OIDC_VERSION_1_0",
      "accessTokenType": "OIDC_TOKEN_TYPE_BEARER"
    }
  }' | jq
```
Response includes `clientId` and `clientSecret`. **Save the client_secret — it cannot be retrieved again.**

#### 7. List Applications
```bash
curl -s -X POST "https://auth.example.com/zitadel.application.v2.ApplicationService/ListApplications" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{
    "filters": [
      {"projectIdFilter": {"projectId": "123456789012345678"}}
    ]
  }' | jq '.applications[] | {id: .applicationId, name: .name, type: (.oidcConfiguration != null) as $oidc | ($oidc | if . then "OIDC" else "") + ((.samlConfiguration != null) as $saml | if $saml then "SAML" else "") + ((.apiConfiguration != null) as $api | if $api then "API" else "")}'
```

Note: Each app response includes one of `oidcConfiguration`, `samlConfiguration`, or `apiConfiguration` depending on type.

#### 8. Get Application
```bash
curl -s -X POST "https://auth.example.com/zitadel.application.v2.ApplicationService/GetApplication" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{"applicationId": "123456789012345679"}' | jq
```

#### 9. Update Application (OIDC Config)
Replaces the old REST-style `PATCH /v2/applications/{id}` with RPC-style:

```bash
curl -s -X POST "https://auth.example.com/zitadel.application.v2.ApplicationService/UpdateApplication" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{
    "applicationId": "123456789012345680",
    "projectId": "123456789012345678",
    "oidcConfiguration": {
      "redirectUris": [
        "https://grafana.example.com/login/generic_oauth"
      ],
      "postLogoutRedirectUris": [
        "https://grafana.example.com"
      ]
    }
  }' | jq
```

**Note**: Arrays REPLACE all existing values. Include ALL desired URIs.

#### 9b. Update SAML Application Configuration

```bash
curl -s -X POST "https://auth.example.com/zitadel.application.v2.ApplicationService/UpdateApplication" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{
    "applicationId": "123456789012345678",
    "projectId": "123456789012345678",
    "samlConfiguration": {
      "metadataUrl": "https://sp.example.com/saml/metadata.xml"
    }
  }' | jq
```

#### 9c. Update API Application Configuration

```bash
curl -s -X POST "https://auth.example.com/zitadel.application.v2.ApplicationService/UpdateApplication" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{
    "applicationId": "123456789012345678",
    "projectId": "123456789012345678",
    "apiConfiguration": {
      "authMethodType": "API_AUTH_METHOD_TYPE_PRIVATE_KEY_JWT"
    }
  }' | jq
```

#### 10. Generate New Client Secret (OIDC/API only)
```bash
curl -s -X POST "https://auth.example.com/zitadel.application.v2.ApplicationService/GenerateClientSecret" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{
    "applicationId": "123456789012345680",
    "projectId": "123456789012345678"
  }' | jq -r '.clientSecret'
```

#### 11. Delete Application
```bash
curl -s -X POST "https://auth.example.com/zitadel.application.v2.ApplicationService/DeleteApplication" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{
    "applicationId": "123456789012345678",
    "projectId": "123456789012345678"
  }' | jq
```

### Users

#### 12. Create Human User
```bash
curl -s -X POST "https://auth.example.com/v2/users/human" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "jdoe",
    "profile": {
      "givenName": "Jane",
"familyName": "Doe"
    },
    "email": {
      "email": "admin@example.com",
      "sendCode": {}
    }
  }' | jq
```

Sets initial password via Management API:
```bash
curl -s -X POST "https://auth.example.com/management/v1/users/${USER_ID}/password" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "x-zitadel-orgid: ${ORG_ID}" \
  -d '{
    "password": "Jane123!",
    "changeRequired": false
  }' | jq
```

#### 13. Get Current User (verify PAT)
```bash
curl -s "https://auth.example.com/v2/users/me" \
  -H "Authorization: Bearer ${PAT}" | jq
```

### Project Roles & Grants

#### 14. Add Project Role
```bash
curl -s -X POST "https://auth.example.com/zitadel.project.v2.ProjectService/AddProjectRole" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{
    "projectId": "123456789012345678",
    "roleKey": "admin",
    "displayName": "Administrator",
    "group": "default"
  }' | jq
```

#### 15. List Project Roles
```bash
curl -s -X POST "https://auth.example.com/zitadel.project.v2.ProjectService/ListProjectRoles" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{"projectId": "123456789012345678"}' | jq '.result[] | {key: .key, displayName: .displayName}'
```

#### 16. Assign Project Role to User (v2)

Same-org role assignments (user and project in same org). Uses the v2 AuthorizationService.

```bash
curl -s -X POST "https://auth.example.com/zitadel.authorization.v2.AuthorizationService/CreateAuthorization" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{
    "userId": "${USER_ID}",
    "projectId": "${PROJECT_ID}",
    "organizationId": "${ORG_ID}",
    "roleKeys": ["viewer"]
  }' | jq
```
Response returns `id` and `creationDate`.

#### 17. List Role Assignments (Authorizations)

```bash
curl -s -X POST "https://auth.example.com/zitadel.authorization.v2.AuthorizationService/ListAuthorizations" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{"projectId": "${PROJECT_ID}"}' | jq '.authorizations[] | {user: .user.preferredLoginName, roles: [.roles[].key]}'
```

#### 18. Enable Project Role Check (Restrict Access)

By default, all users in the ZITADEL instance can authenticate to any project's apps. Enable `projectRoleCheck` to require role assignments.

```bash
curl -s -X PUT "https://auth.example.com/management/v1/projects/${PROJECT_ID}" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "x-zitadel-orgid: ${ORG_ID}" \
  -d '{
    "name": "Acme",
    "projectRoleAssertion": true,
    "projectRoleCheck": true,
    "hasProjectCheck": false
  }' | jq
```

| Setting | Effect |
|---------|--------|
| `projectRoleAssertion` | Include roles in tokens (userinfo endpoint) |
| `projectRoleCheck` | Deny login if user has no role in this project |
| `hasProjectCheck` | Deny login if user's org doesn't own/grant this project |

#### 19. Cross-Org Project Grant (v1 — deprecated)

For granting access to users from a DIFFERENT organization:
```bash
curl -s -X POST "https://auth.example.com/management/v1/projects/${PROJECT_ID}/grants" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "x-zitadel-orgid: ${ORG_ID}" \
  -d '{
    "roleKeys": ["viewer"],
    "userId": "${USER_ID}",
    "grantedOrgId": "${GRANTED_ORG_ID}"
  }' | jq
```

**Note**: Use CreateAuthorization v2 for same-org assignments. The v1 grants API is cross-org only.

## End-to-End Workflows

### Applications (SAML 2.0)

SAML apps use the same `CreateApplication` endpoint with `samlConfiguration`. No clientId/clientSecret — SAML uses metadata XML or URL.

**SAML config fields:**

| Field | Type | Description |
|-------|------|-------------|
| `metadataXml` | string (base64) | Full SAML metadata XML as string. Use for static metadata. One of `metadataXml` or `metadataUrl` required. |
| `metadataUrl` | string | URL where ZITADEL fetches SAML metadata. Use when IdP metadata URL is known. |

**SAML response:** Returns `applicationId` only. No `clientId` or `clientSecret`.

#### 5a. Create SAML Application (Metadata XML)

```bash
curl -s -X POST "https://auth.example.com/zitadel.application.v2.ApplicationService/CreateApplication" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{
    "projectId": "123456789012345678",
    "name": "Enterprise SAML App",
    "samlConfiguration": {
      "metadataXml": "<?xml version=\"1.0\"?>\n<md:EntityDescriptor xmlns:md=\"urn:oasis:names:tc:SAML:2.0:metadata\" entityID=\"https://sp.example.com/saml\">\n  <md:SPSSODescriptor protocolSupportEnumeration=\"urn:oasis:names:tc:SAML:2.0:protocol\">\n    <md:AssertionConsumerService Binding=\"urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST\" Location=\"https://sp.example.com/saml/acs\" index=\"0\"/>\n  </md:SPSSODescriptor>\n</md:EntityDescriptor>"
    }
  }' | jq
```

#### 5b. Create SAML Application (Metadata URL)

```bash
curl -s -X POST "https://auth.example.com/zitadel.application.v2.ApplicationService/CreateApplication" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{
    "projectId": "123456789012345678",
    "name": "Enterprise SAML App",
    "samlConfiguration": {
      "metadataUrl": "https://sp.example.com/saml/metadata.xml"
    }
  }' | jq
```

### Applications (API/Machine-to-Machine)

API apps authenticate at the introspection endpoint using client credentials. No user interaction.

**API config fields:**

| Field | Type | Description |
|-------|------|-------------|
| `authMethodType` | string | `"API_AUTH_METHOD_TYPE_BASIC"` (client_id+client_secret) or `"API_AUTH_METHOD_TYPE_PRIVATE_KEY_JWT"` (JWT assertion) |

**API response:** Returns `clientId` and (for BASIC auth) `clientSecret`. **Save the secret — cannot be retrieved later.**

#### 5c. Create API Application (Basic Auth)

```bash
curl -s -X POST "https://auth.example.com/zitadel.application.v2.ApplicationService/CreateApplication" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{
    "projectId": "123456789012345678",
    "name": "API Client",
    "apiConfiguration": {
      "authMethodType": "API_AUTH_METHOD_TYPE_BASIC"
    }
  }' | jq
```

Response:
```json
{
  "applicationId": "123456789012345678",
  "apiConfiguration": {
    "clientId": "123456789012345678@myproject",
    "clientSecret": "gjoq34589uasgh"
  }
}
```

#### 5d. Create API Application (Private Key JWT)

```bash
curl -s -X POST "https://auth.example.com/zitadel.application.v2.ApplicationService/CreateApplication" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{
    "projectId": "123456789012345678",
    "name": "API Client (JWT)",
    "apiConfiguration": {
      "authMethodType": "API_AUTH_METHOD_TYPE_PRIVATE_KEY_JWT"
    }
  }' | jq
```

Response has `clientId` but no `clientSecret` (auth uses JWT assertion signed with app key).

For Private Key JWT, create an application key to sign assertions:
```bash
curl -s -X POST "https://auth.example.com/zitadel.application.v2.ApplicationService/CreateApplicationKey" \
  -H "Authorization: Bearer ${PAT}" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{
    "applicationId": "123456789012345678",
    "projectId": "123456789012345678",
    "keyType": "APPLICATION_KEY_TYPE_JSON"
  }' | jq -r '.keyDetails' | base64 -d
```

Returns a JSON key pair (private key + public key). Use the private key to sign JWT assertions for token exchange.

#### App type summary table

| Type | Config field | Auth | Returns client secret | Use case |
|------|-------------|------|----------------------|----------|
| OIDC | `oidcConfiguration` | Authorization code, PKCE, implicit, device code, JWT | Yes | User-facing apps (web, SPA, mobile) |
| SAML | `samlConfiguration` | SAML 2.0 assertion | No (metadata-based) | Enterprise SSO |
| API | `apiConfiguration` | Basic (client_secret) or private_key_jwt | Yes (basic) / No (JWT) | Machine-to-machine, server-to-server |

### Workflow A: Bootstrap a New Org with User, Project, Roles, and App

Creates a complete org from scratch with a human user, project, roles, OIDC app, and role grant.

```bash
export PAT=$(kubectl get secret iam-admin-pat -n zitadel -o jsonpath='{.data.pat}' | base64 -d)
export BASE="https://auth.example.com"

# 1. Create organization
ORG_RESP=$(curl -s -X POST "$BASE/zitadel.org.v2.OrganizationService/CreateOrganization" \
  -H "Authorization: Bearer $PAT" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{"name": "Family"}')
ORG_ID=$(echo "$ORG_RESP" | jq -r '.orgId')
echo "Org: $ORG_ID"

# 2. Create project
PROJ_RESP=$(curl -s -X POST "$BASE/zitadel.project.v2.ProjectService/CreateProject" \
  -H "Authorization: Bearer $PAT" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{"projectName": "Family-Apps"}')
PROJ_ID=$(echo "$PROJ_RESP" | jq -r '.id')
echo "Project: $PROJ_ID"

# 3. Add roles (admin, viewer, editor)
for role in admin viewer editor; do
  curl -s -X POST "$BASE/zitadel.project.v2.ProjectService/AddProjectRole" \
    -H "Authorization: Bearer $PAT" \
    -H "Content-Type: application/json" \
    -H "Connect-Protocol-Version: 1" \
    -d "{\"projectId\": \"$PROJ_ID\", \"roleKey\": \"$role\", \"displayName\": \"${role^}\", \"group\": \"default\"}" > /dev/null
done
echo "Roles created: admin, viewer, editor"

# 4. Create human user
USER_RESP=$(curl -s -X POST "$BASE/v2/users/human" \
  -H "Authorization: Bearer $PAT" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "jdoe",
    "profile": {"givenName": "Jane", "familyName": "Acme"},
    "email": {"email": "admin@example.com", "sendCode": {}}
  }')
USER_ID=$(echo "$USER_RESP" | jq -r '.userId')
echo "User: $USER_ID"

# 5. Set initial password (no email verification)
curl -s -X POST "$BASE/management/v1/users/$USER_ID/password" \
  -H "Authorization: Bearer $PAT" \
  -H "Content-Type: application/json" \
  -H "x-zitadel-orgid: $ORG_ID" \
  -d '{"password": "Welcome123!", "changeRequired": true}' > /dev/null
echo "Password set"

# 6. Grant user viewer role (v2 AuthorizationService)
curl -s -X POST "$BASE/zitadel.authorization.v2.AuthorizationService/CreateAuthorization" \
  -H "Authorization: Bearer $PAT" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{"userId": "'$USER_ID'", "projectId": "'$PROJ_ID'", "organizationId": "'$ORG_ID'", "roleKeys": ["viewer"]}' > /dev/null
echo "Role viewer granted to user"

# 7. Create OIDC PKCE app
APP_RESP=$(curl -s -X POST "$BASE/zitadel.application.v2.ApplicationService/CreateApplication" \
  -H "Authorization: Bearer $PAT" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{
    "projectId": "'$PROJ_ID'",
    "name": "Immich",
    "oidcConfiguration": {
      "redirectUris": ["https://immich.example.com/auth/login", "app.immich:///oauth-callback"],
      "responseTypes": ["OIDC_RESPONSE_TYPE_CODE"],
      "grantTypes": ["OIDC_GRANT_TYPE_AUTHORIZATION_CODE"],
      "applicationType": "OIDC_APP_TYPE_USER_AGENT",
      "authMethodType": "OIDC_AUTH_METHOD_TYPE_NONE",
      "postLogoutRedirectUris": ["https://immich.example.com"],
      "version": "OIDC_VERSION_1_0",
      "accessTokenType": "OIDC_TOKEN_TYPE_BEARER"
    }
  }')
APP_ID=$(echo "$APP_RESP" | jq -r '.applicationId')
CLIENT_ID=$(echo "$APP_RESP" | jq -r '.oidcConfiguration.clientId')
echo "App: $APP_ID | ClientID: $CLIENT_ID"
```

### Workflow B: Register OIDC App and Store in K8s Secret

Full flow for creating an oauth2-proxy OIDC app and immediately saving credentials to cluster.

```bash
#!/usr/bin/env bash
set -euo pipefail

PAT=$(kubectl get secret iam-admin-pat -n zitadel -o jsonpath='{.data.pat}' | base64 -d)
PROJECT_ID="123456789012345678"  # Acme project
APP_NAME="MyApp"
REDIRECT_URI="https://myapp.example.com/oauth2/callback"
NAMESPACE="myapp"

# Create app with client_secret_basic auth
APP_RESP=$(curl -s -X POST "https://auth.example.com/zitadel.application.v2.ApplicationService/CreateApplication" \
  -H "Authorization: Bearer $PAT" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{
    "projectId": "'$PROJECT_ID'",
    "name": "'$APP_NAME'",
    "oidcConfiguration": {
      "redirectUris": ["'$REDIRECT_URI'"],
      "responseTypes": ["OIDC_RESPONSE_TYPE_CODE"],
      "grantTypes": ["OIDC_GRANT_TYPE_AUTHORIZATION_CODE"],
      "applicationType": "OIDC_APP_TYPE_WEB",
      "authMethodType": "OIDC_AUTH_METHOD_TYPE_BASIC",
      "version": "OIDC_VERSION_1_0",
      "accessTokenType": "OIDC_TOKEN_TYPE_BEARER"
    }
  }')

CLIENT_ID=$(echo "$APP_RESP" | jq -r '.oidcConfiguration.clientId')
CLIENT_SECRET=$(echo "$APP_RESP" | jq -r '.oidcConfiguration.clientSecret')
APP_ID=$(echo "$APP_RESP" | jq -r '.applicationId')

echo "Created app $APP_NAME (id=$APP_ID, client=$CLIENT_ID)"

# Store in K8s secret
kubectl create secret generic "${APP_NAME}-oidc-secret" \
  -n "$NAMESPACE" \
  --from-literal=client-id="$CLIENT_ID" \
  --from-literal=client-secret="$CLIENT_SECRET" \
  --dry-run=client -o yaml | kubectl apply -f -

echo "Stored in secret ${APP_NAME}-oidc-secret in namespace $NAMESPACE"
```

### Workflow C: Fix Grafana OIDC Redirect URIs + Regenerate Secret

```bash
export PAT=$(kubectl get secret iam-admin-pat -n zitadel -o jsonpath='{.data.pat}' | base64 -d)
export BASE="https://auth.example.com"
export PROJ_ID="123456789012345678"
export APP_ID="123456789012345680"

# Update redirect URIs
curl -s -X POST "$BASE/zitadel.application.v2.ApplicationService/UpdateApplication" \
  -H "Authorization: Bearer $PAT" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{
    "applicationId": "'$APP_ID'",
    "projectId": "'$PROJ_ID'",
    "oidcConfiguration": {
      "redirectUris": ["https://grafana.example.com/login/generic_oauth"],
      "postLogoutRedirectUris": ["https://grafana.example.com"]
    }
  }' | jq

# Regenerate client secret and update K8s secret
NEW_SECRET=$(curl -s -X POST "$BASE/zitadel.application.v2.ApplicationService/GenerateClientSecret" \
  -H "Authorization: Bearer $PAT" \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{"applicationId": "'$APP_ID'", "projectId": "'$PROJ_ID'"}' | jq -r '.clientSecret')

kubectl patch secret grafana-oidc-secret -n monitoring \
  --type='json' \
  -p="[{\"op\": \"replace\", \"path\": \"/data/client-secret\", \"value\": \"$(echo -n "$NEW_SECRET" | base64 -w0)\"}]"

echo "Grafana client secret updated"
```

### Workflow D: Migrate Existing App from v1 (AppService Deprecated) to v2

The old `zitadel.app.v2.AppService/AddApp` or `zitadel.app.v2beta.AppService/CreateApplication` paths are gone.

**If you get 404 when trying to manage apps, check:**

| Symptom | Fix |
|---------|-----|
| 404 on `zitadel.app.v2.AppService/*` | Change to `zitadel.application.v2.ApplicationService/*` |
| 404 on `PATCH /v2/applications/{id}` | Use RPC-style POST to `zitadel.application.v2.ApplicationService/UpdateApplication` |
| 404 on `AddOIDCApp` | Use `CreateApplication` with `oidcConfiguration` field |

**Before → After mapping:**
```text
# OLD (404):
POST /zitadel.app.v2.AppService/AddOIDCApp

# NEW (200):
POST /zitadel.application.v2.ApplicationService/CreateApplication

# OLD PATCH REST (404):
PATCH /v2/applications/{id}  {"oidcConfiguration": {...}}

# NEW RPC (200):
POST /zitadel.application.v2.ApplicationService/UpdateApplication
{"applicationId": "...", "projectId": "...", "oidcConfiguration": {...}}
```

### Workflow E: Bulk Create Roles + Assign to Multiple Users

```bash
export PAT=$(kubectl get secret iam-admin-pat -n zitadel -o jsonpath='{.data.pat}' | base64 -d)
PROJ_ID="123456789012345678"
ORG_ID="374839521201686400"

# Create roles in one loop
for role in admin viewer editor auditor operator; do
  curl -s -X POST "https://auth.example.com/zitadel.project.v2.ProjectService/AddProjectRole" \
    -H "Authorization: Bearer $PAT" \
    -H "Content-Type: application/json" \
    -H "Connect-Protocol-Version: 1" \
    -d "{\"projectId\": \"$PROJ_ID\", \"roleKey\": \"$role\", \"displayName\": \"Role: $role\", \"group\": \"default\"}"
  echo ""
done

# Grant specific roles to users (v2 AuthorizationService)
declare -A USERS
USERS[uid1]="admin"
USERS[uid2]="viewer"
USERS[uid3]="editor,viewer"

for user_id in "${!USERS[@]}"; do
  roles="${USERS[$user_id]}"
  roles_json=$(echo "$roles" | jq -R 'split(",")')
  
  curl -s -X POST "https://auth.example.com/zitadel.authorization.v2.AuthorizationService/CreateAuthorization" \
    -H "Authorization: Bearer $PAT" \
    -H "Content-Type: application/json" \
    -H "Connect-Protocol-Version: 1" \
    -d "{\"userId\": \"$user_id\", \"projectId\": \"$PROJ_ID\", \"organizationId\": \"$ORG_ID\", \"roleKeys\": $roles_json}"
  echo ""
done
```

### Workflow F: Audit All OIDC Apps with Secrets (via DB)

When API access is limited or you need to audit all apps across all orgs:

```bash
kubectl exec -n zitadel deploy/zitadel -- psql -h localhost -d zitadel -U zitadel -t -A -F',' <<'SQL'
SELECT 
  o.name AS org_name,
  p.name AS project_name,
  a.name AS app_name,
  c.client_id,
  c.client_secret,
  c.redirect_uris::text AS redirects
FROM projections.apps7 AS a
JOIN projections.apps7_oidc_configs AS c ON a.id = c.app_id
JOIN projections.apps7_projects AS p ON a.project_id = p.project_id
JOIN projections.orgs AS o ON p.resource_owner = o.id
WHERE a.state = 1
ORDER BY o.name, p.name, a.name;
SQL
```

## API Endpoint Reference

### Full RPC endpoints (Connect POST)

| Operation | Endpoint | Method |
|-----------|----------|--------|
| Create Org | `zitadel.org.v2.OrganizationService/CreateOrganization` | POST |
| List Orgs | `zitadel.org.v2.OrganizationService/ListOrganizations` | POST |
| Create Project | `zitadel.project.v2.ProjectService/CreateProject` | POST |
| List Projects | `zitadel.project.v2.ProjectService/ListProjects` | POST |
| Add Role | `zitadel.project.v2.ProjectService/AddProjectRole` | POST |
| List Roles | `zitadel.project.v2.ProjectService/ListProjectRoles` | POST |
| Delete Role | `zitadel.project.v2.ProjectService/DeleteProjectRole` | POST |
| **Create App** (OIDC/SAML/API) | **`zitadel.application.v2.ApplicationService/CreateApplication`** | POST |
| List Apps | `zitadel.application.v2.ApplicationService/ListApplications` | POST |
| Get App | `zitadel.application.v2.ApplicationService/GetApplication` | POST |
| Update App (OIDC/SAML/API) | `zitadel.application.v2.ApplicationService/UpdateApplication` | POST |
| Delete App | `zitadel.application.v2.ApplicationService/DeleteApplication` | POST |
| Deactivate App | `zitadel.application.v2.ApplicationService/DeactivateApplication` | POST |
| Reactivate App | `zitadel.application.v2.ApplicationService/ReactivateApplication` | POST |
| Gen Client Secret (OIDC/API only) | `zitadel.application.v2.ApplicationService/GenerateClientSecret` | POST |
| Create App Key (API JWT) | `zitadel.application.v2.ApplicationService/CreateApplicationKey` | POST |

| Create Authorization (Role Assignment) | `zitadel.authorization.v2.AuthorizationService/CreateAuthorization` | POST |
| List Authorizations | `zitadel.authorization.v2.AuthorizationService/ListAuthorizations` | POST |
| Activate Authorization | `zitadel.authorization.v2.AuthorizationService/ActivateAuthorization` | POST |
| Deactivate Authorization | `zitadel.authorization.v2.AuthorizationService/DeactivateAuthorization` | POST |

### REST endpoints

| Operation | Method | Endpoint |
|-----------|--------|----------|
| Create User | POST | `https://${DOMAIN}/v2/users/human` |
| Get Current User | GET | `https://${DOMAIN}/v2/users/me` |
| Set Password | POST | `https://${DOMAIN}/management/v1/users/{userId}/password` |
| Enable Project Role Check | PUT | `https://${DOMAIN}/management/v1/projects/{projectId}` |

## Error Handling

| HTTP Status | Meaning | Common Cause |
|-------------|---------|-------------|
| 200 | Success | — |
| 400 | Bad Request | Invalid JSON, missing required fields, wrong enum value |
| 403 | Forbidden | PAT lacks permission or scope; wrong org context |
| 404 | Not Found | Wrong project/app ID; **wrong endpoint path** (common: using old AppService path) |
| 409 | Conflict | Duplicate role key; app already exists |

**Common errors:**

- `404 Not Found` on CreateApplication → Wrong service name. Use `zitadel.application.v2.ApplicationService` not `zitadel.app.v2.AppService` or `zitadel.app.v2beta.AppService`.
- `403 permission denied` → PAT user doesn't have `project.app.write` permission on the target project/org.
- `400 InvalidArgument` → Missing required fields (project_id, name, or app configuration); or invalid enum value (use string enum names like `"OIDC_GRANT_TYPE_AUTHORIZATION_CODE"`, `"API_AUTH_METHOD_TYPE_BASIC"`).
- `400 missing samlConfiguration.metadataXml or samlConfiguration.metadataUrl` → SAML requires exactly one of metadataXml or metadataUrl.
- `"No changes"` on UpdateApplication → New values match existing. Still returns 200 with previous changeDate.
- `"User could not be found"` on user operations → User belongs to a deleted org; query DB directly to find user.
- PAT returning 403 on project operations → PAT was created in a different org than the target project.

## Tips

**App type quick reference:**

| Type | Config key | Auth method | Returns secret | Use case |
|------|-----------|-------------|----------------|----------|
| OIDC | `oidcConfiguration` | Authorization code, PKCE, implicit, device code, JWT | Yes (except PKCE) | User-facing apps |
| SAML | `samlConfiguration` | SAML 2.0 metadata | No | Enterprise SSO |
| API | `apiConfiguration` | Basic / Private Key JWT | Yes (basic) / No (JWT) | M2M, server-to-server |

- v2 RPC endpoints use `POST` for ALL operations (even reads). No GET reads in RPC.
- `organizationId` goes in request body for v2 RPCs when needed, NOT headers.
- v2 REST-style `PATCH /v2/applications/{id}` is DEPRECATED. Use `zitadel.application.v2.ApplicationService/UpdateApplication` RPC instead.
- **OIDC enum values are now string names** (e.g., `"OIDC_AUTH_METHOD_TYPE_NONE"`) in JSON, not numeric values. The Connect RPC gateway auto-converts.
- Client secret is returned ONCE on creation or regeneration. Store it immediately in K8s secret.
- Sending an email verification code: set `"sendCode": {}` in the email field when creating a user. This sends a verification email from ZITADEL.
- For setting initial password without verification code, use the management API: `POST /management/v1/users/{userId}/password`.
- Fetch complete API index: `https://mintlify.com/zitadel/zitadel/llms.txt`
- Legacy v1 operations (role grants, some admin functions) remain at `/management/v1/` with `x-zitadel-orgid` header.
- When in doubt about endpoint path, check the proto files at `https://github.com/zitadel/zitadel/tree/main/proto/zitadel/application/v2/`.
- **Role assignments**: Use `zitadel.authorization.v2.AuthorizationService/CreateAuthorization` for same-org role grants (user and project in same org). The v1 `management/v1/projects/{id}/grants` is cross-org only.
- **Project role check**: Enable `projectRoleCheck=true` on a project to restrict login to users with role assignments. Otherwise any instance user can authenticate. Manage via `PUT /management/v1/projects/{id}`.
- **Authorization != OAuth**: In ZITADEL API naming, "Authorization" means a role assignment (user grant), not OAuth2 authorization. The service is at `zitadel.authorization.v2.AuthorizationService`.
