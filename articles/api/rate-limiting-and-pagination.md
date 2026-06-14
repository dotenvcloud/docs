---
title: Rate Limiting & Pagination
slug: rate-limiting-and-pagination
order: 4
description: API rate limits and headers, plus pagination parameters and metadata for list endpoints.
tags: [api, rate-limiting, pagination, headers]
---

# Rate Limiting & Pagination

## Rate limiting

Different endpoint groups have different limits:

| Endpoint group | Limit | Scope |
| --- | --- | --- |
| OAuth token (`/api/oauth/token`) | 10 / minute | per IP |
| OAuth revoke (`/api/oauth/revoke`) | 10 / minute | per IP |
| Encryption key rotation (`.../encryption-key/rotate`) | 10 / hour | per organization |
| All other API endpoints | 100 / minute | per organization |
| Billing write actions (`payment-methods`, `subscription`, `charges`, ...) | additional per-action limits | per organization |

### Rate limit headers

Responses include:

| Header | Description |
| --- | --- |
| `X-RateLimit-Limit` | Maximum requests allowed in the window. |
| `X-RateLimit-Remaining` | Requests remaining in the current window. |
| `X-RateLimit-Reset` | Unix timestamp when the window resets. |
| `Retry-After` | Seconds to wait before retrying (only on `429` responses). |

When you exceed a limit you receive `429 Too Many Requests`. Honor `Retry-After` before retrying.

## Pagination

Paginated list endpoints accept `page` and `per_page`.

| Parameter | Type | Default | Bounds |
| --- | --- | --- | --- |
| `page` | integer | 1 | ≥ 1 |
| `per_page` | integer | 25 | 1–100 |

The main paginated endpoint is **secret version history**
(`POST /api/v1/{organization}/secrets/versions`), where these are sent in the request body:

```bash
curl -X POST https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/secrets/versions \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "project": "production",
    "target": "us-east",
    "environment": "production",
    "page": 1,
    "per_page": 25
  }'
```

Paginated responses include a `meta` object alongside the `data` array, for example:

```json
{
  "data": [ ],
  "meta": {
    "current_page": 1,
    "per_page": 25,
    "total": 137,
    "last_page": 6
  }
}
```
