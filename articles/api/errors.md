---
title: Errors
slug: errors
order: 3
description: The DotEnv API error envelope and the HTTP status codes the API returns, including domain-specific error codes.
tags: [api, errors, status-codes, troubleshooting]
---

# Errors

## Error envelope

API errors use a flat JSON envelope — there is no `data` wrapper:

```json
{
  "error": "insufficient_permissions",
  "message": "Insufficient permissions",
  "details": { }
}
```

| Field | Type | Description |
| --- | --- | --- |
| `error` | string | Stable machine-readable code. Match on this in SDKs. |
| `message` | string | Human-readable description. |
| `details` | object | Optional. Present only for some errors (e.g. validation, retention). |

OAuth endpoints (`/api/oauth/token`, `/api/oauth/revoke`) use the OAuth error shape instead:
`{"error": "...", "error_description": "..."}`.

## HTTP status codes

| Status | Meaning | When it happens |
| --- | --- | --- |
| `200 OK` | Success | Reads, stores, rotations, restores, purges. |
| `201 Created` | Resource created | Project / target / environment / organization / API key creation. |
| `204 No Content` | Success, empty body | Deletes (resource, secret blob, API key). |
| `400 Bad Request` | Invalid request | Cannot rotate client-managed key via API; OAuth grant errors; client-key rotation validation. |
| `401 Unauthorized` | Missing / invalid token | No `Authorization` header or an expired/revoked token. |
| `403 Forbidden` | Insufficient scope or access | Token lacks the required scope, or version is outside the plan's retention window. |
| `404 Not Found` | Resource not found | Unknown org/project/target/environment, or cross-organization access attempt. |
| `409 Conflict` | Conflicting operation | A key rotation is already in progress (re-encrypt history pending). |
| `422 Unprocessable Entity` | Validation / proof error | Invalid body, key-proof mismatch, missing confirmation. |
| `429 Too Many Requests` | Rate limited | See [Rate limiting](rate-limiting-and-pagination). |

## Common error codes

These codes appear in the `error` field:

| Code | Status | Meaning |
| --- | --- | --- |
| `insufficient_permissions` | 403 | The token lacks the required scope. |
| `not_found` | 404 | The resource does not exist, or is not in this organization. |
| `invalid_parameter_combination` | 422 | Conflicting query parameters (e.g. invalid merge/raw combination). |
| `key_proof_required` | 422 | Client-managed project has no key verification configured yet. |
| `key_proof_mismatch` | 422 | The supplied key proof does not match the project's key. |
| `key_version_conflict` | 422 | A rotation raced a re-encryption submission. |
| `content_required_for_old_key` | 422 | Restoring a client-managed version under an old key needs re-encrypted content. |
| `version_locked` | 403 | The version is outside the plan's history retention window. `details` carries `retention_days` and `version_created_at`. |
| `version_protected` | 422 | Refused to delete the only restore snapshot of a deleted secret. Pass `force` to override. |

## Validation errors

Validation failures (status `422`) come from the framework and include a `message` plus an
`errors` map keyed by field name, for example:

```json
{
  "message": "The name field is required.",
  "errors": {
    "name": ["The name field is required."]
  }
}
```
