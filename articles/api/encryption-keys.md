---
title: Encryption Keys
slug: encryption-keys
order: 9
description: Retrieve the active encryption key, view key history, rotate server-managed keys, and rotate client-managed keys with re-encryption.
tags: [api, encryption, keys, rotation, pbkdf2]
---

# Encryption Keys

Each project has an active encryption key. A key is either **server-managed** (the server stores
it encrypted and can decrypt secrets for you) or **client-managed** (the server never holds the
key and only verifies a one-way proof of it).

## Server-managed vs client-managed

| | Server-managed | Client-managed |
| --- | --- | --- |
| Key storage | Stored encrypted server-side | Never stored server-side |
| Server can decrypt | Yes (with `secret:decrypt`) | No |
| `GET encryption-key` returns | The key + version | Only proof params (`key_check_salt`, `key_check_iterations`) + version |
| Rotation endpoint | `encryption-key/rotate` | `secrets/rotate-client-keys` |

### The PBKDF2 key proof

For client-managed projects the server verifies a pushed key with a one-way proof instead of the
key itself. The proof is computed identically across all consumers (PHP/Go/JS):

```
proof = base64( PBKDF2-HMAC-SHA256(
  password    = padKey(key),   // the padded 32-byte AES key, not the raw string
  salt        = 16 random bytes,
  iterations  = 600000,
  dkLen       = 32 ) )
```

The client establishes `{key_check, key_check_salt, key_check_iterations}` at project creation /
key setup, and sends `key_proof` on every secrets write. The server compares it with `hash_equals`
and rejects a mismatch (`key_proof_mismatch`, 422).

## Endpoints

| Method | Path | Scope | Description |
| --- | --- | --- | --- |
| `GET` | `/api/v1/{organization}/{project}/encryption-key` | `key:retrieve` | Get the active key descriptor. |
| `GET` | `/api/v1/{organization}/{project}/encryption-key/history` | `key:retrieve` | List key version history (newest first). |
| `POST` | `/api/v1/{organization}/{project}/encryption-key/rotate` | `key:rotate` | Rotate a server-managed key. |
| `POST` | `/api/v1/{organization}/{project}/secrets/rotate-client-keys` | (token-validated) | Rotate a client-managed key with re-encrypted secrets. |
| `POST` | `/api/v1/{organization}/{project}/secrets/re-encrypt-history/pending` | `key:rotate` | List historical client-managed versions awaiting re-encryption. |
| `POST` | `/api/v1/{organization}/{project}/secrets/re-encrypt-history` | `key:rotate` | Submit historical versions re-encrypted under the current key. |

## Get the active key

`GET /api/v1/{organization}/{project}/encryption-key` — requires `key:retrieve`. Always returns
`200` (client-managed is not an error). The descriptor is discriminated by `managed`:
server-managed includes `key`; client-managed includes only the proof params.

```bash
curl https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app/encryption-key \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"
```

## Key history

`GET /api/v1/{organization}/{project}/encryption-key/history` — requires `key:retrieve`. Active key
first, then by descending version.

```bash
curl https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app/encryption-key/history \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"
```

## Rotate a server-managed key

`POST /api/v1/{organization}/{project}/encryption-key/rotate` — requires `key:rotate`. Limited to
10 rotations/hour per organization. Only works for server-managed keys (client-managed keys return
`400`).

The optional `history_policy` controls historical versions: `keep` leaves them under the old key;
`re_encrypt` queues a server-side job to migrate them onto the new key. Defaults to the project's
setting.

```bash
curl -X POST https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app/encryption-key/rotate \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{ "history_policy": "re_encrypt" }'
```

Returns `{"message": "...", "data": {"key": { ...descriptor... }}}`.

## Rotate a client-managed key

`POST /api/v1/{organization}/{project}/secrets/rotate-client-keys`. Because the server cannot
decrypt, **you** generate the new key locally, re-encrypt every current secret blob under it, and
submit the re-encrypted blobs plus the new key proof (`key_check`, `key_check_salt`,
`key_check_iterations`).

```bash
curl -X POST https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app/secrets/rotate-client-keys \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "secrets": [
      { "id": "01ARZ3NDEKTSV4RRFFQ69G5FAX", "content": "bGa8J2ZW5jcnlwdGVkX2Jhc2U2NF9zdHJpbmcuLi4=" },
      { "id": "01ARZ3NDEKTSV4RRFFQ69G5FAY", "content": "cEo5K3aX6kdyeXB0ZWRfYmFzZTY0X3N0cmluZy4uLg==" }
    ],
    "key_check": "NmvqZy0aWO8MAZ/l3xHShSRA3IRhdRwM6jCBBHDP+eE=",
    "key_check_salt": "AAAAAAAAAAAAAAAAAAAAAA==",
    "key_check_iterations": 600000
  }'
```

Returns `{"success": true, "message": "...", "data": {"count": N}}`.

## Re-encrypting historical versions (client-managed)

Rotating the current secrets does not move old **versions** onto the new key. Migrate them in
batches:

1. **List pending** versions still under the old key:

```bash
curl -X POST https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app/secrets/re-encrypt-history/pending \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{ "limit": 50 }'
```

Returns `data` (each `{id, content, key_version}`) and `meta.remaining`. If a rotation is in
progress you get `409 Conflict`.

2. **Decrypt locally** with the old key, re-encrypt with the new key, then **submit**:

```bash
curl -X POST https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app/secrets/re-encrypt-history \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "versions": [
      { "id": 42, "content": "k7sBvP2xQ1m9Yb3eGf8nRwTzA4hLcD6oXyU0iJpKqNrShV=" }
    ],
    "key_proof": "cGJrZGYyJHNoYTI1NiQ2MDAwMDAk..."
  }'
```

Returns `{"data": {"updated": N, "remaining": M}}`. A wrong proof returns `422 key_proof_mismatch`;
a racing rotation returns `422 key_version_conflict`. Repeat until `remaining` is `0`.
