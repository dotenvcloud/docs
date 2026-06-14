---
title: Secrets
slug: secrets
order: 7
description: Retrieve, store, and delete encrypted secret blobs; understand the encrypted-blob model, hierarchy merging, and decryption responsibilities.
tags: [api, secrets, encryption, hierarchy, merge]
---

# Secrets

Secrets in DotEnv are stored as **encrypted blobs** per hierarchy level. Each level (project,
target, environment) holds one encrypted `.env` blob. The API never sees plaintext for
client-managed projects, and only decrypts server-side on explicit request for server-managed
projects.

## The encrypted-blob model

- Secrets are encrypted with **AES-256-GCM**: a 32-byte key, a 12-byte IV, and an authentication
  tag. The wire format is `base64(IV + ciphertext + tag)`.
- A blob is stored at the **deepest level provided** in a write (environment > target > project).
- On read, levels are returned separately by default, or merged so that more-specific levels
  override less-specific ones.

### Decryption responsibilities

- **Server-managed keys:** the server can decrypt the blob for you when you pass `decrypt=true`
  (requires `secret:decrypt`). It can also return ciphertext for you to decrypt locally.
- **Client-managed keys:** the server holds no key and **cannot** decrypt. It always returns
  ciphertext; you decrypt locally with your key. See [Encryption keys](encryption-keys).

## Reading secrets (hierarchical GET)

Read the encrypted blobs at each level. The deeper the path, the more levels are included.

| Method | Path | Scope |
| --- | --- | --- |
| `GET` | `/api/v1/{organization}/{project}/secrets` | `secret:read` |
| `GET` | `/api/v1/{organization}/{project}/{target}/secrets` | `secret:read` |
| `GET` | `/api/v1/{organization}/{project}/{target}/{environment}/secrets` | `secret:read` |

Query parameters:

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `decrypt` | boolean | `false` | Decrypt server-side (server-managed keys; also requires `secret:decrypt`). |
| `merge` | `false` \| `true` \| `both` | `false` | `false` = levels separate; `true` = merged flat only; `both` = both. |
| `raw` | boolean | `false` | Return raw key-value content without the response wrapper. |

```bash
# Encrypted, hierarchical (default)
curl "https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app/backend/production/secrets" \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"

# Decrypted, merged, raw — ready for an .env file (server-managed only)
curl "https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app/backend/production/secrets?decrypt=true&merge=true&raw=true" \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"
```

Default (structured, encrypted) response:

```json
{
  "data": {
    "type": "secrets",
    "attributes": {
      "encrypted": true,
      "format": "hierarchical",
      "levels": {
        "project": { "DATABASE_URL": "encrypted_base64_string..." },
        "target": { "NODE_ENV": "encrypted_base64_string..." },
        "environment": { "DEBUG": "encrypted_base64_string..." }
      }
    }
  }
}
```

> Passing `decrypt=true` without the `secret:decrypt` scope returns
> `403 {"error": "insufficient_permissions"}`.

## Retrieving secrets (POST retrieve)

For complex queries (filters, name selection, action selection) use the POST endpoint. It is
available on both formats.

| Method | Path | Scope |
| --- | --- | --- |
| `POST` | `/api/v1/{organization}/secrets/retrieve` | depends on `action` (below) |

The required scope depends on `action`:

- `read` (default) → `secret:read`
- `decrypt` → `secret:decrypt`
- `key:retrieve` → `key:retrieve`

```bash
curl -X POST https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/secrets/retrieve \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "project": "my-app",
    "target": "backend",
    "environment": "production",
    "action": "read",
    "filters": { "names": ["DATABASE_URL", "API_KEY"] },
    "merge": "false",
    "raw": false
  }'
```

Body fields: `project` (required), `target`, `environment`, `action`
(`read` | `decrypt` | `key:retrieve`), `filters` (`names`, `tags`, `search`),
`merge` (`false` | `true` | `both`), `raw`.

## Storing secrets (POST store)

Upsert the **already-encrypted** blob for a level. The level is the deepest slug provided.

| Method | Path | Scope |
| --- | --- | --- |
| `POST` | `/api/v1/{organization}/secrets/store` | `secret:write` |

```bash
curl -X POST https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/secrets/store \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "project": "my-app",
    "target": "backend",
    "environment": "production",
    "content": "k7sBvP2xQ1m9Yb3eGf8nRwTzA4hLcD6oXyU0iJpKqNrShV=",
    "no_backup": false
  }'
```

Body fields:

| Field | Required | Description |
| --- | --- | --- |
| `project` | yes | Project slug. |
| `target` | no | Target slug (selects the level). |
| `environment` | no | Environment slug (selects the level). |
| `content` | yes | The already-encrypted `.env` blob for the resolved level. |
| `key_proof` | for client-managed | PBKDF2 key proof; the server rejects a mismatch (`key_proof_mismatch`, 422). |
| `no_backup` | no | Skip recording a backup version for this write. |

Response:

```json
{
  "data": { "type": "secrets", "level": "environment", "source": "production", "bytes": 47 },
  "message": "Secrets stored successfully."
}
```

> Client-managed projects must send `key_proof`. If the project has no key verification configured
> yet, the server returns `422 {"error": "key_proof_required"}`. A wrong key returns
> `422 {"error": "key_proof_mismatch"}` and nothing is stored.

## Deleting secrets (POST delete)

Clear the blob for a level.

| Method | Path | Scope |
| --- | --- | --- |
| `POST` | `/api/v1/{organization}/secrets/delete` | `secret:write` (`secret:purge` if `no_backup`) |

```bash
curl -X POST https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/secrets/delete \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "project": "my-app",
    "target": "backend",
    "environment": "production",
    "no_backup": false
  }'
```

Returns `204 No Content`. `no_backup: true` purges the level's version history and hard-deletes
the row — it requires `confirmed: true` and the `secret:purge` scope.

## Version history

Every write records a backup version (unless `no_backup`). To list, fetch, restore, delete, or
purge versions, see [Secret versions](secret-versions).
