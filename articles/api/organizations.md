---
title: Organizations
slug: organizations
order: 5
description: List, read, create, update, and delete organizations, plus the current-organization and member endpoints.
tags: [api, organizations, members]
---

# Organizations

An organization is the top-level container for projects, members, billing, and API tokens. It is
identified by a 26-character ULID.

## Endpoints

| Method | Path | Scope | Description |
| --- | --- | --- | --- |
| `GET` | `/api/v1/organizations` | — (auto-scoped) | List organizations the principal can access. |
| `POST` | `/api/v1/organizations` | — (any authenticated principal) | Create a new organization. |
| `GET` | `/api/v1/organizations/{organization}` | `organization:read` | Get an organization by ULID. |
| `PUT` / `PATCH` | `/api/v1/organizations/{organization}` | `organization:update` | Update an organization. |
| `DELETE` | `/api/v1/organizations/{organization}` | `organization:delete` | Delete an organization. |
| `GET` | `/api/v1/organization` | — (token's org) | Get the organization tied to the current token. |
| `POST` | `/api/v1/organizations/{organization}/members` | `member:create` | Add a member to the organization. |

> **Why some endpoints have no scope gate:** the list endpoint is automatically scoped to the
> authenticated principal (an org token sees only its own organization; a user token sees the
> user's organizations). Creating an organization has no pre-existing org to gate on, so any
> authenticated principal may create one and becomes its owner. The single-org `/api/v1/organization`
> endpoint returns the organization bound to the presented token.

## List organizations

`GET /api/v1/organizations`

```bash
curl https://api.dotenv.cloud/api/v1/organizations \
  -H "Authorization: Bearer <token>" \
  -H "Accept: application/json"
```

## Create an organization

`POST /api/v1/organizations`

```bash
curl -X POST https://api.dotenv.cloud/api/v1/organizations \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{ "name": "Acme Corporation" }'
```

Returns `201 Created` with the new organization in `data`.

## Get an organization

`GET /api/v1/organizations/{organization}` — requires `organization:read`.

```bash
curl https://api.dotenv.cloud/api/v1/organizations/01ARZ3NDEKTSV4RRFFQ69G5FAV \
  -H "Authorization: Bearer <token>" \
  -H "Accept: application/json"
```

Example response:

```json
{
  "data": {
    "id": "01ARZ3NDEKTSV4RRFFQ69G5FAV",
    "name": "Acme Corporation",
    "slug": "acme-corp",
    "status": "active"
  }
}
```

## Get the current organization

`GET /api/v1/organization` returns the organization associated with the current API token. It is
rate-limited but does not count against organization usage limits.

```bash
curl https://api.dotenv.cloud/api/v1/organization \
  -H "Authorization: Bearer <token>" \
  -H "Accept: application/json"
```

## Update an organization

`PUT` or `PATCH /api/v1/organizations/{organization}` — requires `organization:update`.

```bash
curl -X PATCH https://api.dotenv.cloud/api/v1/organizations/01ARZ3NDEKTSV4RRFFQ69G5FAV \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{ "name": "Acme Corp (renamed)" }'
```

## Delete an organization

`DELETE /api/v1/organizations/{organization}` — requires `organization:delete`. Returns `204 No Content`.

```bash
curl -X DELETE https://api.dotenv.cloud/api/v1/organizations/01ARZ3NDEKTSV4RRFFQ69G5FAV \
  -H "Authorization: Bearer <token>"
```

## Add a member

`POST /api/v1/organizations/{organization}/members` — requires `member:create`.

```bash
curl -X POST https://api.dotenv.cloud/api/v1/organizations/01ARZ3NDEKTSV4RRFFQ69G5FAV/members \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{ "email": "teammate@example.com", "role": "member" }'
```
