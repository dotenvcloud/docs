---
title: API Keys
slug: api-keys
order: 7
description: Create fine-grained, scoped API keys (including read-only), set expirations, rotate or delete keys, test them with the API Explorer, and follow key security best practices.
tags: [web-app, api-keys, tokens, permissions, scopes, security, api-explorer]
---

# API Keys

API keys (tokens) let the CLI, SDKs, CI pipelines, and other integrations talk to DotEnv on
behalf of your organization. You manage them under **API Keys** at
`/dashboard/organizations/api-keys`.

## Creating a key

1. Go to **API Keys** (`/dashboard/organizations/api-keys`).
2. Click **Create** (`/dashboard/organizations/api-keys/create`).
3. Give the key a **name** you'll recognize later.
4. Choose **permissions** (and optionally **scope** them — see below).
5. Choose an **expiration**.
6. Create the key.

### The token is shown only once

When the key is created, its secret token is displayed **one time only**. Copy it immediately and
store it somewhere safe (a secrets manager or your CI's secret store). If you lose it, you cannot
view it again — you must rotate or recreate the key.

## Permissions

Permissions are fine-grained and grouped by area. You select exactly what the key is allowed to
do:

- **Secrets** — read, write, decrypt (server-side), version history, restore, purge, plus
  retrieve/rotate encryption keys.
- **Projects**, **Targets**, **Environments** — create / read / update / delete.
- **Teams**, **Organizations**, **Members** — read/update/delete (and create where applicable).
- **API Tokens** — manage other tokens.
- **Billing** — read or manage billing.

You can toggle an entire group at once, or pick individual permissions. There's also an
**allow all** option for full access — use it sparingly.

### Read-only preset

A **read-only** preset is available that grants just the read permissions (read secrets,
projects, targets, environments, and organization). This is ideal for integrations that only need
to **pull** secrets and should never modify anything.

## Scoping a key

Most permissions can be **scoped** so the key only applies to part of your hierarchy. You can
restrict a permission to a specific:

- **Team**
- **Project**
- **Target**
- **Environment**

Create permissions (like `project:create`) are not scopeable, since there's nothing yet to scope
to. Scoping follows the hierarchy — for example, a secret-read permission can be narrowed to a
single environment, while a project permission scopes at the project level. Combining a narrow
permission set with tight scopes gives you least-privilege keys (for example: "read secrets, only
in `production` of the `web` target of the `billing` project").

## Expiration

When creating a key you choose how long it lives:

- **Never** (no expiry)
- **30 / 60 / 90 days**
- **1 year**
- **Custom** date (must be in the future)

Prefer an expiration over "never" wherever practical, and rotate before keys lapse.

## Viewing, editing, rotating, and deleting

From the API Keys list you can:

- **Show** a key (`/api-keys/{token}`) to review its name, permissions, scopes, and expiration.
- **Edit** a key (`/api-keys/{token}/edit`).
- **Rotate** a key — generates a new secret token and invalidates the old one. The new token is,
  again, shown only once.
- **Delete** a key — immediately revokes it.

## Testing with the API Explorer

Each key has an **API Explorer** at `/dashboard/organizations/api-keys/{token}/explore`. Use it
to make real API calls with that key directly from the dashboard so you can confirm its
permissions and scopes behave as expected, inspect responses, and copy code snippets — without
wiring anything up locally first.

## Security guidance

- **Never share a key** or commit it to source control. Treat it like a password.
- **Scope and limit** every key to the minimum it needs — start from read-only when you can.
- **Set an expiration** rather than "never".
- **Rotate immediately if a key leaks**, then delete the old one. Anyone who has the token can
  act with its permissions until it is rotated or deleted.
- Give each integration its **own** key so you can revoke one without affecting others.
