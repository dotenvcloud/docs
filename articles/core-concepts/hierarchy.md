---
title: The DotEnv Hierarchy
slug: hierarchy
order: 2
description: How DotEnv organizes secrets — organization, project, target, environment, and the secrets themselves — with examples of when to use each level.
tags: [core-concepts, hierarchy, organization, project, target, environment, secrets]
---

# The DotEnv Hierarchy

DotEnv organizes everything into a nested structure. Each level narrows the scope, and secrets
can live at the project, target, or environment level. Understanding the hierarchy is the key to
understanding everything else — inheritance, access control, and how you address secrets from the
CLI and API all follow it.

```
Organization
└── Project
    └── Target
        └── Environment
            └── Secrets
```

## Organization

The **organization** is the top-level container and the billing and membership boundary. People
are members of an organization, roles are assigned within it, and API tokens belong to it. Every
project lives inside exactly one organization.

Organizations are identified by a **ULID** (a sortable, URL-safe identifier such as
`01ARZ3NDEKTSV4RRFFQ69G5FAV`). You normally see your organization by name in the dashboard; the
ULID is what the API uses in paths.

## Project

A **project** represents one application or service — for example `acme-store` or `billing-api`.
It is the unit you grant access to and the unit an encryption key belongs to. A project has a
human-readable name and a **slug** (a short, URL-safe identifier like `acme-store`) used by the
CLI and API.

> Each project has its own encryption key. See
> [Security Model](/documentation/core-concepts/security-model).

## Target

A **target** is a logical grouping inside a project, identified by a **slug**. Targets let you
separate parts of the same application that need different configuration. Common splits:

- `backend` and `frontend`
- `api`, `worker`, and `web`
- by region, such as `eu` and `us`

Targets are how you keep, say, a backend service's database credentials separate from a frontend
build's public API URLs while still living under one project.

## Environment

An **environment** is where secrets actually live, identified by a **slug**. Each environment
carries a **status** that signals how sensitive it is:

- **Production**
- **Staging**
- **Development**
- **Local**

Typical environments mirror these statuses (`production`, `staging`, `development`, `local`), and
the production status flags an environment as holding live, sensitive values that deserve extra
care.

## Secrets

A **secret** is a single configuration value — a `KEY=value` pair such as
`DATABASE_URL=postgres://...`. Secrets can be defined at the **project**, **target**, or
**environment** level.

Under the hood, DotEnv stores the secrets for each level as a single **encrypted blob** — it does
not store one row per variable, and it never parses your `KEY=value` content on the server. When
you retrieve secrets, the blobs for the levels in your path are decrypted and merged into a final
set. How that merge resolves overrides is covered in
[Secret Inheritance](/documentation/core-concepts/secret-inheritance).

## A worked example

Imagine an organization `Acme Inc` with a project `acme-store`, split into a `backend` target and
a `frontend` target, each with `production` and `development` environments:

```
Acme Inc (organization, ULID 01ARZ3...)
└── acme-store (project)
    ├── backend (target)
    │   ├── production (environment, status: Production)
    │   │   ├── DATABASE_URL=postgres://prod...
    │   │   └── STRIPE_SECRET_KEY=sk_live_...
    │   └── development (environment, status: Development)
    │       └── DATABASE_URL=postgres://localhost...
    └── frontend (target)
        ├── production (environment, status: Production)
        │   └── API_BASE_URL=https://api.acme.com
        └── development (environment, status: Development)
            └── API_BASE_URL=http://localhost:8000
```

## How you address it

The same structure is how you reference secrets everywhere:

- **Dashboard** — pick project → target → environment in the Secret Editor.
- **CLI** — a path mirrors the levels: `acme-store/backend/production`.
- **API** — paths combine the organization ULID with the project, target, and environment slugs.

```bash
# CLI: pull the backend production environment of acme-store
dotenv pull acme-store/backend/production
```

## Choosing where to put a secret

A simple rule keeps things tidy:

- **Project level** — values shared everywhere (a third-party base URL, a shared feature flag).
- **Target level** — values that differ between parts of the app but not between environments.
- **Environment level** — values that change per environment (the production database URL vs. the
  local one).

This naturally complements inheritance: put a value at the broadest level it's correct for, and
only override it deeper when it truly differs. Continue to
[Secret Inheritance](/documentation/core-concepts/secret-inheritance).
