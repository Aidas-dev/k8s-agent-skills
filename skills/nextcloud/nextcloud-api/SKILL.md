---
name: nextcloud-api
description: Use when managing Nextcloud users, groups, and apps via the Provisioning API (OCS). Covers user CRUD, group management, app enable/disable, and curl examples. NOT for Helm deployment — use nextcloud-helm.
---

# Nextcloud Provisioning API

## Overview

The Provisioning API enables external systems to create, edit, delete and query user attributes, manage groups, set quotas, query storage, and manage apps. Enabled by default. Uses the OCS API endpoint format.

**Base URL:** `https://nextcloud.example.com/ocs/v1.php/cloud`

**Auth:** Basic HTTP Auth (admin username:password) + OCS-APIRequest header

**Headers (required for ALL calls):**
- `Authorization: Basic <base64(admin:password)>`
- `OCS-APIRequest: true`
- POST requests also require `Content-Type: application/x-www-form-urlencoded`

**Output format:** XML by default. Append `?format=json` for JSON.

## Users

### Add a New User

```bash
curl -X POST "https://nextcloud.example.com/ocs/v1.php/cloud/users" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "userid=newuser" \
  -d "password=SecurePass123!" \
  -d "displayName=New User" \
  -d "email=newuser@example.com" \
  -d "groups[]=admin" \
  -d "groups[]=group1" \
  -d "quota=10GB"
```

Status codes: 100=ok, 101=invalid, 102=already exists, 104=group missing

### List Users

```bash
curl -X GET "https://nextcloud.example.com/ocs/v1.php/cloud/users?search=filter&limit=100&offset=0" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true" | python3 -c "import sys, xml.etree.ElementTree as ET; print('\n'.join(e.text for e in ET.parse(sys.stdin).findall('.//users/element')))"
```

### Get Single User

```bash
curl -X GET "https://nextcloud.example.com/ocs/v1.php/cloud/users/newuser" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true" | python3 -c "
import sys, xml.etree.ElementTree as ET
root = ET.parse(sys.stdin).find('.//data')
for child in root:
    if child.tag == 'groups':
        print(f'{child.tag}: {[e.text for e in child]}')
    elif child.text and child.text.strip():
        print(f'{child.tag}: {child.text.strip()}')
"
```

Returns: enabled, id, quota, email, displayname, phone, address, website, twitter, groups.

### Edit User

```bash
curl -X PUT "https://nextcloud.example.com/ocs/v1.php/cloud/users/newuser" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "key=email" \
  -d "value=newemail@example.com"
```

Editable fields: `email`, `quota`, `displayname`, `phone`, `address`, `website`, `twitter`, `password`. Admin can set `quota`; users can only edit their own email, displayname, password.

### Disable User

```bash
curl -X PUT "https://nextcloud.example.com/ocs/v1.php/cloud/users/newuser/disable" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true"
```

### Enable User

```bash
curl -X PUT "https://nextcloud.example.com/ocs/v1.php/cloud/users/newuser/enable" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true"
```

### Delete User

```bash
curl -X DELETE "https://nextcloud.example.com/ocs/v1.php/cloud/users/newuser" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true"
```

### Get User Groups

```bash
curl -X GET "https://nextcloud.example.com/ocs/v1.php/cloud/users/newuser/groups" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true"
```

### Add User to Group

```bash
curl -X POST "https://nextcloud.example.com/ocs/v1.php/cloud/users/newuser/groups" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "groupid=newgroup"
```

### Remove User from Group

```bash
curl -X DELETE "https://nextcloud.example.com/ocs/v1.php/cloud/users/newuser/groups" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "groupid=newgroup"
```

### Promote to Subadmin

```bash
curl -X POST "https://nextcloud.example.com/ocs/v1.php/cloud/users/newuser/subadmins" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "groupid=group"
```

### Demote from Subadmin

```bash
curl -X DELETE "https://nextcloud.example.com/ocs/v1.php/cloud/users/newuser/subadmins" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "groupid=oldgroup"
```

### Resend Welcome Email

```bash
curl -X POST "https://nextcloud.example.com/ocs/v1.php/cloud/users/newuser/welcome" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true"
```

## Groups

### List Groups

```bash
curl -X GET "https://nextcloud.example.com/ocs/v1.php/cloud/groups?search=adm&limit=50&offset=0" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true"
```

### Create Group

```bash
curl -X POST "https://nextcloud.example.com/ocs/v1.php/cloud/groups" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "groupid=newgroup"
```

Status codes: 100=ok, 101=invalid, 102=already exists

### Get Group Members

```bash
curl -X GET "https://nextcloud.example.com/ocs/v1.php/cloud/groups/admin" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true"
```

Returns list of users in the group.

### Get Group Subadmins

```bash
curl -X GET "https://nextcloud.example.com/ocs/v1.php/cloud/groups/mygroup/subadmins" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true"
```

### Edit Group Display Name

```bash
curl -X PUT "https://nextcloud.example.com/ocs/v1.php/cloud/groups/mygroup" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "key=displayname" \
  -d "value=My Group Name"
```

### Delete Group

```bash
curl -X DELETE "https://nextcloud.example.com/ocs/v1.php/cloud/groups/mygroup" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true"
```

Does not delete users in the group. Cannot delete the `admin` group.

## Apps

### List Apps

```bash
curl -X GET "https://nextcloud.example.com/ocs/v1.php/cloud/apps?filter=enabled" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true"
```

Filter: `enabled`, `disabled`, or omit for all.

### Get App Info

```bash
curl -X GET "https://nextcloud.example.com/ocs/v1.php/cloud/apps/files" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true"
```

### Enable App

```bash
curl -X POST "https://nextcloud.example.com/ocs/v1.php/cloud/apps/files_texteditor" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true"
```

### Disable App

```bash
curl -X DELETE "https://nextcloud.example.com/ocs/v1.php/cloud/apps/files_texteditor" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true"
```

## Editable Fields Reference

Available via API:

```bash
curl -X GET "https://nextcloud.example.com/ocs/v1.php/cloud/user/fields" \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "OCS-APIRequest: true"
```

Returns: `displayname`, `email`, `phone`, `address`, `website`, `twitter`.

## Error Handling

| Status Code | Meaning |
|---|---|
| 100 | Success |
| 101 | Invalid argument / failure |
| 102 | Already exists (user/group) |
| 103 | Cannot create sub-admins for admin group |
| 104 | Group does not exist (user add) / insufficient privileges (user add to group) |
| 105 | Failed to add user to group |
| 106 | No group specified (required for sub-admins) |
| 107 | Password policy violation |
| 108 | Email required for password link |
| 112 | Password change not supported by backend |
| 113 | Editing field not allowed or doesn't exist |

## Tips

- **JSON output** — Append `?format=json` to any endpoint URL for JSON responses instead of XML
- **OCS-APIRequest header is mandatory** — Without it, requests return empty or 404
- **Base URL path** — `ocs/v1.php/cloud/` (not `ocs/v2.php/` — that's the Share API)
- **Password-on-create** — Leave password empty and provide email to send welcome email with password setup link
- **Groups array** — Use `groups[]=group1` & `groups[]=group2` for multiple groups in create
- **Quota format** — Supports values like `1GB`, `500MB`, `unlimited` (or just enter `none` for unlimited)
- **Basic auth** — Use the Nextcloud admin user, not an OCC system user
- **OCC alternative** — For bulk operations or complex config, `occ` command inside the pod is more powerful:
  ```bash
  kubectl exec deploy/nextcloud -- php occ user:list
  kubectl exec deploy/nextcloud -- php occ group:add mygroup
  kubectl exec deploy/nextcloud -- php occ user:add --display-name="Layla Smith" --group="users" layla
  ```
- **App IDs** — App ID for the Provisioning API is `provisioning_api` (enabled by default); remove it to disable the API
- **Rate limiting** — Nextcloud has no built-in API rate limiting. Add at reverse proxy if needed.
