---
title: API Keys
slug: api-keys
order: 10
description: Create, list, read, update, delete, and rotate organization API tokens, including required scopes.
tags: [api, api-keys, tokens, scopes]
---

# API Keys

API keys are organization-scoped Bearer tokens with a fixed set of abilities (scopes). Manage them
through these endpoints or the web app's **API Keys** page. The plaintext key value is shown
**once** at creation/rotation and cannot be retrieved later.

These endpoints use the legacy long format with the `organizations/{organization}` prefix.

## Endpoints

| Method | Path | Scope | Description |
| --- | --- | --- | --- |
| `GET` | `/api/v1/organizations/{organization}/api-keys` | `api-token:read` | List API keys. |
| `POST` | `/api/v1/organizations/{organization}/api-keys` | `api-token:create` | Create an API key. |
| `GET` | `/api/v1/organizations/{organization}/api-keys/{token}` | `api-token:read` | Get an API key. |
| `PUT` | `/api/v1/organizations/{organization}/api-keys/{token}` | `api-token:update` | Update an API key. |
| `DELETE` | `/api/v1/organizations/{organization}/api-keys/{token}` | `api-token:delete` | Delete (revoke) an API key. |
| `POST` | `/api/v1/organizations/{organization}/api-keys/{token}/rotate` | `api-token:create` + `api-token:delete` | Rotate an API key. |

`{token}` is the API token's ID.

## List API keys

```bash
curl https://api.dotenv.cloud/api/v1/organizations/01ARZ3NDEKTSV4RRFFQ69G5FAV/api-keys \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"
```

Returns `data` (array of token resources, without the secret value).

## Create an API key

`abilities` is the list of scopes (see [Authentication](authentication)). `expires_at` is optional.

```bash
curl -X POST https://api.dotenv.cloud/api/v1/organizations/01ARZ3NDEKTSV4RRFFQ69G5FAV/api-keys \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "CI/CD Pipeline Token",
    "abilities": ["secret:read", "project:read"],
    "expires_at": "2025-12-31T23:59:59Z"
  }'
```

Returns `201 Created`. The plaintext token is in the `token` field — store it now:

```json
{
  "success": true,
  "message": "API token created successfully",
  "token": "dotenv_01ARZ3NDEKTSV4RRFFQ69G5FAV_a1b2c3d4e5f6...",
  "api_key": {
    "id": "01ARZ3NDEKTSV4RRFFQ69G5FAV",
    "name": "CI/CD Pipeline Token",
    "abilities": ["secret:read", "project:read"],
    "expires_at": "2025-12-31T23:59:59Z",
    "created_at": "2024-01-15T10:30:00Z"
  }
}
```

## Get an API key

```bash
curl https://api.dotenv.cloud/api/v1/organizations/01ARZ3NDEKTSV4RRFFQ69G5FAV/api-keys/01ARZ3NDEKTSV4RRFFQ69G5FAV \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"
```

## Update an API key

You can change `name`, `abilities`, and `expires_at`.

```bash
curl -X PUT https://api.dotenv.cloud/api/v1/organizations/01ARZ3NDEKTSV4RRFFQ69G5FAV/api-keys/01ARZ3NDEKTSV4RRFFQ69G5FAV \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Renamed Deploy Token",
    "abilities": ["secret:read"]
  }'
```

## Delete an API key

Returns `204 No Content`. The token is immediately revoked.

```bash
curl -X DELETE https://api.dotenv.cloud/api/v1/organizations/01ARZ3NDEKTSV4RRFFQ69G5FAV/api-keys/01ARZ3NDEKTSV4RRFFQ69G5FAV \
  -H "Authorization: Bearer <token>"
```

## Rotate an API key

Revokes the old value and issues a new one with the same settings. Requires both
`api-token:create` and `api-token:delete`. The new plaintext token is returned once.

```bash
curl -X POST https://api.dotenv.cloud/api/v1/organizations/01ARZ3NDEKTSV4RRFFQ69G5FAV/api-keys/01ARZ3NDEKTSV4RRFFQ69G5FAV/rotate \
  -H "Authorization: Bearer <token>"
```

Returns the same creation envelope as `POST .../api-keys` with the new `token`.
