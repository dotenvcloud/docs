---
title: Error Reference
slug: error-reference
order: 2
description: The DotEnv API error envelope, the HTTP status codes the API returns and what each means, and the common DotEnv CLI error messages.
tags: [troubleshooting, errors, api, cli, status-codes]
---

# Error Reference

This is a quick lookup for the errors you'll see from the DotEnv **API** and **CLI**. For
step-by-step fixes, see [Common Issues](/documentation/troubleshooting/common-issues).

## API error shape

API errors return a flat JSON envelope (there is no `data` wrapper) with a standard HTTP status
code:

```json
{
  "error": "insufficient_permissions",
  "message": "Insufficient permissions",
  "details": {}
}
```

| Field | Type | Description |
| --- | --- | --- |
| `error` | string | Stable machine-readable code. Match on this in code. |
| `message` | string | Human-readable description. |
| `details` | object | Optional. Present only for some errors (e.g. validation, retention). |

Validation failures (status `422`) additionally carry an `errors` map keyed by field name:

```json
{
  "message": "The name field is required.",
  "errors": {
    "name": ["The name field is required."]
  }
}
```

## Common HTTP status codes

| Status | Meaning | What it usually indicates |
| --- | --- | --- |
| `401 Unauthorized` | Missing or invalid credentials | No `Authorization` header, or an expired/revoked token or API key. Re-authenticate. |
| `403 Forbidden` | Authenticated, but not allowed | Your role or token scope lacks the required permission, or the requested version is outside your plan's retention window. |
| `404 Not Found` | Resource doesn't exist (here) | Unknown org/project/target/environment/secret — or an attempt to reach a resource in another organization. |
| `422 Unprocessable Entity` | Validation or proof error | Invalid request body, a key-proof mismatch, or a missing confirmation. Check `details`/`errors`. |
| `429 Too Many Requests` | Rate limited | You've exceeded the request rate. Back off and retry; respect the rate-limit headers. |

Other statuses you may encounter: `200/201/204` on success, `400` for malformed requests (e.g.
trying to rotate a client-managed key via the API), and `409` when an operation conflicts with one
already in progress (e.g. a key rotation).

### Distinguishing 401 vs 403

- **401** means *we don't know who you are* — your credentials are missing, expired, or revoked.
- **403** means *we know who you are, but you're not allowed* — fix the role or the token scope, not
  the credentials.

## Common CLI error messages

The CLI translates API errors into actionable messages. The most common:

| Message (abridged) | What it means | Fix |
| --- | --- | --- |
| `no accounts configured` | No stored account and no `DOTENV_API_KEY`. | `dotenv login`, or set `DOTENV_API_KEY`. |
| `authentication failed. Please check your credentials or run 'dotenv login'` | Credentials rejected (401). | Re-check the API key, or log in again. |
| `your session has expired. Please run 'dotenv login'` | OAuth token expired. | `dotenv account refresh`, or `dotenv login`. |
| `no organization selected` | No current org for an OAuth account. | `dotenv org list`, then `dotenv org use <organization>`. |
| `organization/project/target/environment '…' not found` | Wrong path or wrong org (404). | Verify with `dotenv list …`; confirm the selected org. |
| `access denied to <resource>` | Role/scope lacks permission (403). | Adjust your role or API-key scope; check the org. |
| `this project uses client-managed encryption. Provide your key via --client-key …` | A client-managed project needs your key to decrypt. | Pass `--client-key <file>` (or `DOTENV_CLIENT_KEY`), or answer the prompt. |
| `the encryption key does not match project '…' — refusing to push …` | Key-proof mismatch on push (wrong/mistyped key). | Use the project's established key. |
| `project '…' has no client-key verification configured …` | Client-managed project has no proof set up yet. | Re-establish the project's encryption key (web dashboard). |
| `rate limit exceeded. Please wait N seconds before trying again` | Rate limited (429). | Wait the indicated time, then retry. |
| `validation error: …` | Request body failed validation (422). | Correct the reported fields. |

Run the failing command again with `--debug` to see the underlying request and response.
