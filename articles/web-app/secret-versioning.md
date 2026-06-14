---
title: Secret Versioning
slug: secret-versioning
order: 5
description: Enable per-project secret versioning, browse version history, restore previous versions, and purge old history.
tags: [web-app, secrets, versioning, history, restore, purge]
---

# Secret Versioning

Secret versioning keeps a history of changes to your secrets so you can review past values and
roll back when something goes wrong. Versioning is a **per-project** setting.

## Enabling versioning

Versioning is controlled by the **Secret versioning** toggle on a project:

1. Open the project (`/dashboard/projects/{project}`).
2. Click **Edit** (`/dashboard/projects/{project}/edit`).
3. Turn on **Secret versioning** and save.

Once enabled, each time a secret is saved a new version is recorded for that project.

## Viewing history

When versioning is on, the secret editor exposes a **version history** for each level. The
history lists previous versions so you can see what changed and when.

Like everything else in DotEnv, versions are stored **encrypted**. The history panel returns
encrypted payloads, which are decrypted in your browser using the project's encryption setup, so
you can read a previous value to compare it against the current one.

## Restoring a version

From the history you can **restore** a previous version, which makes that older value the current
one again. Restore respects the project's encryption model:

- For server-managed projects, the server can restore directly.
- For client-managed projects, your browser re-encrypts the chosen version under the current
  active key before it is saved, so the restored value stays readable.

> **Rotated keys and history:** if you have rotated the project's encryption key, historical
> versions remain decryptable with the key version they were written under. Rotated (old) key
> versions are kept immutable specifically so old history can still be read. See
> [Encryption Keys](encryption-keys).

## Deleting and purging history

You can clean up history when you no longer need it:

- **Delete a single version** — remove one specific entry from the history.
- **Purge history** — clear the stored history for a secret. Purge is a destructive action and
  asks you to confirm before proceeding.

## Permissions

Versioning actions map to fine-grained permissions, so access can be granted independently:

- **List and read history** (`secret:history`)
- **Restore a version** (`secret:restore`)
- **Delete versions / purge history** (`secret:purge`)

These can also be assigned to [API Keys](api-keys), letting automation read history or restore
versions without full write access.
