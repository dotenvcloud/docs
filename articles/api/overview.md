---
title: API Overview
slug: overview
order: 1
description: Base URL, versioning, path conventions, ULID vs slug identifiers, and your first request against the DotEnv HTTP API.
tags: [api, overview, getting-started, rest]
---

# API Overview

The DotEnv HTTP API lets you manage organizations, projects, targets, environments, secrets,
encryption keys, API tokens, and billing programmatically. It is the same API the CLI and the
official SDKs use.

## Base URL

```
https://api.dotenv.cloud
```

All API endpoints live under the `/api/v1` prefix:

```
https://api.dotenv.cloud/api/v1
```

A staging environment is also available at `https://staging-api.dotenv.cloud`.

## Versioning

The API is versioned in the URL path (`/api/v1`). Breaking changes ship as a new path version.

You may additionally send an explicit version header:

```
X-API-Version: 1
```

Responses echo the negotiated version and include a standard set of security headers
(`X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `X-XSS-Protection: 1; mode=block`,
`Strict-Transport-Security`).

## Request headers

| Header | When | Value |
| --- | --- | --- |
| `Authorization` | All authenticated endpoints | `Bearer <token>` |
| `Accept` | Always (recommended) | `application/json` |
| `Content-Type` | `POST` / `PUT` / `PATCH` | `application/json` |
| `X-API-Version` | Optional | `1` |

## Path conventions

Resources follow a strict hierarchy:

```
Organization
└── Project
    └── Target
        └── Environment
```

Two URL formats exist. **Use the shorter format for new integrations.**

- **Recommended (short):** `/api/v1/{organization}/{project}/{target}/{environment}/...`
- **Legacy (deprecated):** `/api/v1/organizations/{organization}/projects/{project}/targets/{target}/environments/{environment}/...`

The legacy `organizations/.../projects/...` form still works but returns deprecation signaling.
Some endpoints (secret store/delete, project/target/environment write operations) are **only**
available on the short format.

## Identifiers: ULID vs slug

- **Organization** is identified by its **ULID** — a 26-character Crockford base32 string
  (pattern `^[0-9a-hjkmnp-tv-z]{26}$`), e.g. `01ARZ3NDEKTSV4RRFFQ69G5FAV`.
- **Project**, **target**, and **environment** are identified by their **slug** — lowercase
  letters, numbers, and hyphens (pattern `^[a-z0-9-]+$`), e.g. `production`, `us-east`.

Slugs are generated from the resource name and are immutable once created (project, target, and
environment slugs can be changed via an explicit `slug` field on update).

## Your first request

Fetch the organization tied to your token:

```bash
curl https://api.dotenv.cloud/api/v1/organization \
  -H "Authorization: Bearer <token>" \
  -H "Accept: application/json"
```

List the projects in an organization (short format):

```bash
curl https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/projects \
  -H "Authorization: Bearer <token>" \
  -H "Accept: application/json"
```

A successful response is JSON. Single resources are wrapped in a `data` object; collections are
wrapped in a `data` array. Errors use a flat `{ "error", "message", "details?" }` envelope — see
[Errors](errors).

## Next steps

- [Authentication](authentication) — tokens, scopes, and OAuth2 PKCE.
- [Errors](errors) — error envelope and status codes.
- [Rate limiting & pagination](rate-limiting-and-pagination).
- [Secrets](secrets) — the encrypted-blob model and how to read/write secrets.
