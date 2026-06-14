---
title: Projects, Targets & Environments
slug: projects-targets-environments
order: 6
description: CRUD endpoints for the project, target, and environment hierarchy, with required scopes and curl examples.
tags: [api, projects, targets, environments, hierarchy]
---

# Projects, Targets & Environments

Secrets live in a three-level hierarchy below an organization:

```
Organization
└── Project        (slug, e.g. "my-app")
    └── Target     (slug, e.g. "backend")
        └── Environment   (slug, e.g. "production")
```

Each level is identified by a slug. The examples below use the **recommended short format**.

## Projects

| Method | Path | Scope |
| --- | --- | --- |
| `GET` | `/api/v1/{organization}/projects` | `project:read` |
| `POST` | `/api/v1/{organization}/projects` | `project:create` |
| `GET` | `/api/v1/{organization}/{project}` | `project:read` |
| `PUT` / `PATCH` | `/api/v1/{organization}/{project}` | `project:update` |
| `DELETE` | `/api/v1/{organization}/{project}` | `project:delete` |

### List projects

```bash
curl https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/projects \
  -H "Authorization: Bearer <token>" \
  -H "Accept: application/json"
```

### Create a project

A project's `storage_mode` determines whether the encryption key is server-managed (a key is
generated when omitted) or client-managed. Client-managed projects must supply the key-proof
fields (`key_check`, `key_check_salt`, `key_check_iterations`) — see [Encryption keys](encryption-keys).

```bash
curl -X POST https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/projects \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "production",
    "description": "Production application secrets",
    "secret_format": "env",
    "storage_mode": "server"
  }'
```

Returns `201 Created` with the project in `data`.

### Get / update / delete a project

```bash
# Get
curl https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"

# Update (name, description, slug, metadata)
curl -X PATCH https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{ "description": "Updated description" }'

# Delete (204 No Content)
curl -X DELETE https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app \
  -H "Authorization: Bearer <token>"
```

## Targets

| Method | Path | Scope |
| --- | --- | --- |
| `GET` | `/api/v1/{organization}/{project}/targets` | `target:read` |
| `POST` | `/api/v1/{organization}/{project}/targets` | `target:create` |
| `GET` | `/api/v1/{organization}/{project}/{target}` | `target:read` |
| `PUT` / `PATCH` | `/api/v1/{organization}/{project}/{target}` | `target:update` |
| `DELETE` | `/api/v1/{organization}/{project}/{target}` | `target:delete` |

### List / create targets

```bash
# List
curl https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app/targets \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"

# Create
curl -X POST https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app/targets \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{ "name": "us-east", "description": "US East region deployment" }'
```

### Get / update / delete a target

```bash
curl https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app/us-east \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"

curl -X PATCH https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app/us-east \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{ "name": "US East", "slug": "us-east" }'

curl -X DELETE https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app/us-east \
  -H "Authorization: Bearer <token>"
```

## Environments

| Method | Path | Scope |
| --- | --- | --- |
| `GET` | `/api/v1/{organization}/{project}/{target}/environments` | `environment:read` |
| `POST` | `/api/v1/{organization}/{project}/{target}/environments` | `environment:create` |
| `GET` | `/api/v1/{organization}/{project}/{target}/{environment}` | `environment:read` |
| `PUT` / `PATCH` | `/api/v1/{organization}/{project}/{target}/{environment}` | `environment:update` |
| `DELETE` | `/api/v1/{organization}/{project}/{target}/{environment}` | `environment:delete` |

### List / create environments

```bash
# List
curl https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app/us-east/environments \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"

# Create
curl -X POST https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app/us-east/environments \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{
    "name": "production",
    "status": "production",
    "is_production": true
  }'
```

### Get / update / delete an environment

```bash
curl https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app/us-east/production \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"

curl -X PATCH https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app/us-east/production \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{ "is_production": true, "status": "production" }'

curl -X DELETE https://api.dotenv.cloud/api/v1/01ARZ3NDEKTSV4RRFFQ69G5FAV/my-app/us-east/production \
  -H "Authorization: Bearer <token>"
```

> The legacy long format (`/api/v1/organizations/{organization}/projects/{project}/...`) supports
> the **read** (`GET`) operations for this hierarchy. Create, update, and delete are only available
> on the short format shown above.

## Reading secrets at each level

Each level exposes a `secrets` sub-resource for reading the encrypted blob — see [Secrets](secrets).
