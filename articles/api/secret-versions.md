---
title: Secret Versions
slug: secret-versions
order: 8
description: List, fetch, restore, delete, and purge secret backup versions, including retention behavior and client-managed restore.
tags: [api, secrets, versions, history, restore, purge]
---

# Secret Versions

Every secret write records a backup version (unless the write set `no_backup`). Versions let you
inspect history and roll back. Versions are identified by an integer `{version}`.

## Endpoints

| Method | Path | Scope | Description |
| --- | --- | --- | --- |
| `POST` | `/api/v1/{organization}/secrets/versions` | `secret:history` | List version history for a level (paginated). |
| `GET` | `/api/v1/{organization}/secrets/versions/{version}` | `secret:history` | Fetch one version (encrypted content + key descriptor). |
| `POST` | `/api/v1/{organization}/secrets/versions/{version}/restore` | `secret:restore` | Restore a secret to a previous version. |
| `DELETE` | `/api/v1/{organization}/secrets/versions/{version}` | `secret:purge` | Delete a single backup version. |
| `POST` | `/api/v1/{organization}/secrets/versions/purge` | `secret:purge` | Purge version history for a level or whole project. |

`{version}` must be numeric.

## List versions

`POST /api/v1/{organization}/secrets/versions` — requires `secret:history`. Identifies the level by
slugs in the body and supports `page` / `per_page` (see [Pagination](rate-limiting-and-pagination)).

```bash
curl -X POST https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/secrets/versions \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "project": "my-app",
    "target": "backend",
    "environment": "production",
    "page": 1,
    "per_page": 25
  }'
```

Returns `data` (array of versions) and `meta` (pagination).

## Fetch a single version

`GET /api/v1/{organization}/secrets/versions/{version}` — requires `secret:history`. Returns the
version with its encrypted content and the key descriptor needed to decrypt it client-side.

```bash
curl https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/secrets/versions/42 \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"
```

> If the version is outside your plan's retention window, the API returns
> `403 {"error": "version_locked", "details": {"retention_days": ..., "version_created_at": ...}}`.
> Upgrade the plan to access it.

## Restore a version

`POST /api/v1/{organization}/secrets/versions/{version}/restore` — requires `secret:restore`.
Restoring is append-only (it creates a new current version from the old one).

```bash
curl -X POST https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/secrets/versions/42/restore \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{}'
```

For **server-managed** projects no body is needed. For **client-managed** projects where the
version was encrypted under an **old** key, you must re-encrypt the version under the current key
and submit it:

```bash
curl -X POST https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/secrets/versions/42/restore \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "k7sBvP2xQ1m9Yb3eGf8nRwTzA4hLcD6oXyU0iJpKqNrShV=",
    "key_proof": "cGJrZGYyJHNoYTI1NiQ2MDAwMDAk..."
  }'
```

The retention gate is checked before any key logic. Possible errors: `403 version_locked`,
`422 content_required_for_old_key`, `422 key_proof_mismatch`. Success returns
`{"data": {"type": "secrets", "secret_id": ..., "restored_from": "..."}, "message": "..."}`.

## Delete a single version

`DELETE /api/v1/{organization}/secrets/versions/{version}` — requires `secret:purge`. Returns
`204 No Content`.

```bash
curl -X DELETE https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/secrets/versions/42 \
  -H "Authorization: Bearer <token>"
```

> Deleting the only restore snapshot of a deleted secret returns `422 {"error": "version_protected"}`.
> Pass `force` to override.

## Purge history

`POST /api/v1/{organization}/secrets/versions/purge` — requires `secret:purge`. Destructive; must
include `confirmed: true`. Use `scope: level` (default) for one level, or `scope: project` for the
whole project.

```bash
curl -X POST https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/secrets/versions/purge \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "project": "my-app",
    "target": "backend",
    "environment": "production",
    "scope": "level",
    "confirmed": true
  }'
```

Returns `{"data": {"purged_count": N}, "message": "..."}`. Omitting `confirmed` returns `422`.
