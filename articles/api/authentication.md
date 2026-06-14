---
title: Authentication
slug: authentication
order: 2
description: Authenticate with organization API tokens (Bearer), understand the scope system, and use the OAuth2 PKCE token and revoke endpoints.
tags: [api, authentication, tokens, scopes, oauth2, pkce]
---

# Authentication

The DotEnv API supports two ways to authenticate programmatically:

1. **Organization API tokens** — long-lived Bearer tokens with scoped abilities. This is the
   primary method for the CLI, SDKs, and CI/CD.
2. **OAuth2 with PKCE** — for third-party apps acting on behalf of a user.

Both result in a Bearer token presented in the `Authorization` header:

```
Authorization: Bearer <token>
```

## Organization API tokens

API tokens are Laravel Sanctum personal access tokens scoped to an organization. They carry a
fixed set of **abilities** (scopes) chosen at creation time, and they never expire unless you set
an expiry.

### Creating a token

Create and manage tokens from the web app under **API Keys**, or via the API itself
(see [API Keys](api-keys)):

```bash
curl -X POST https://api.dotenv.cloud/api/v1/organizations/01ARZ3NDEKTSV4RRFFQ69G5FAV/api-keys \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "CI/CD Pipeline Token",
    "abilities": ["secret:read", "project:read"]
  }'
```

The plaintext token is returned **once**, in the `token` field of the response. Store it
securely — it cannot be retrieved again.

## Scopes (abilities)

Every endpoint requires a specific scope on the token. Scopes follow the pattern
`resource:action`. A token granted `*` (full access) satisfies every scope.

| Scope | Grants |
| --- | --- |
| `secret:read` | Read encrypted secrets |
| `secret:write` | Create, update, or delete secrets |
| `secret:decrypt` | Decrypt secrets server-side (server-managed keys only) |
| `secret:history` | List and read secret version history |
| `secret:restore` | Restore a secret version |
| `secret:purge` | Delete versions / purge history |
| `key:retrieve` | Retrieve encryption keys |
| `key:rotate` | Rotate encryption keys |
| `project:create` | Create projects |
| `project:read` | Read projects |
| `project:update` | Update projects |
| `project:delete` | Delete projects |
| `target:create` | Create targets |
| `target:read` | Read targets |
| `target:update` | Update targets |
| `target:delete` | Delete targets |
| `environment:create` | Create environments |
| `environment:read` | Read environments |
| `environment:update` | Update environments |
| `environment:delete` | Delete environments |
| `team:create` | Create teams |
| `team:read` | Read teams |
| `team:update` | Update teams |
| `team:delete` | Delete teams |
| `organization:create` | Create organizations |
| `organization:read` | Read organizations |
| `organization:update` | Update organizations |
| `organization:delete` | Delete organizations |
| `api-token:create` | Create API tokens |
| `api-token:read` | Read API tokens |
| `api-token:update` | Update API tokens |
| `api-token:delete` | Revoke API tokens |
| `member:create` | Add members to an organization |
| `member:read` | View members |
| `member:update` | Update member permissions |
| `member:delete` | Remove members |
| `billing:read` | View billing information |
| `billing:write` | Manage subscriptions and payments |

### Read-only preset

A common preset grants read access to the resource hierarchy:

```json
["secret:read", "project:read", "target:read", "environment:read", "organization:read"]
```

### Full access

```json
["*"]
```

A `*` token satisfies any scope check.

> A request with insufficient scope returns `403 Forbidden` with
> `{"error": "insufficient_permissions", ...}`.

## OAuth2 with PKCE

For applications acting on behalf of a user, DotEnv implements the OAuth2 authorization-code flow
with PKCE (RFC 7636). The token and revoke endpoints below are public (no Bearer required) and
rate-limited to 10 requests/minute per IP.

### Exchange a code or refresh token

`POST /api/oauth/token`

Authorization-code grant:

```bash
curl -X POST https://api.dotenv.cloud/api/oauth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "authorization_code",
    "code": "def50200a1b2c3d4e5f6...",
    "client_id": "01ARZ3NDEKTSV4RRFFQ69G5FAV",
    "code_verifier": "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk",
    "redirect_uri": "https://app.example.com/callback"
  }'
```

Refresh-token grant:

```bash
curl -X POST https://api.dotenv.cloud/api/oauth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "refresh_token",
    "refresh_token": "def50200f6e5d4c3b2a1...",
    "client_id": "01ARZ3NDEKTSV4RRFFQ69G5FAV"
  }'
```

Successful response:

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "refresh_a1b2c3d4e5f6g7h8i9j0"
}
```

The `code_verifier` must be 43–128 characters. Failures return an OAuth error body
(`{"error": "invalid_grant" | "unsupported_grant_type", "error_description": "..."}`) with a
`400` status.

### Revoke a token

`POST /api/oauth/revoke` (RFC 7009)

Revocation is possession-based: presenting the refresh token is the proof of ownership. Because
the access and refresh tokens share a single record, revoking invalidates both. The endpoint
**always returns `200`** — even for an unknown token — to avoid leaking which tokens exist.

```bash
curl -X POST https://api.dotenv.cloud/api/oauth/revoke \
  -H "Content-Type: application/json" \
  -d '{
    "token": "def50200f6e5d4c3b2a1...",
    "client_id": "01ARZ3NDEKTSV4RRFFQ69G5FAV"
  }'
```

## The authenticated user

`GET /api/v1/user` returns the authenticated principal and the organizations it can access. It is
org-agnostic (no organization in the path) and is what `dotenv login` uses.

```bash
curl https://api.dotenv.cloud/api/v1/user \
  -H "Authorization: Bearer <token>" \
  -H "Accept: application/json"
```
