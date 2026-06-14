---
title: Secret Versioning
slug: secret-versioning
order: 7
description: List, inspect, diff, restore, and purge previous versions of a secret with the dotenv secret commands.
tags: [cli, secrets, versioning, history, restore, diff, purge]
---

# Secret Versioning

Every push (and, by default, every delete) records a backup version of the secret blob at a
level. The `dotenv secret` subcommands let you list those versions, inspect or diff them
locally, restore an earlier one, or permanently purge history.

Versions are **per level**, so a version belongs to a specific `project`, `project/target`, or
`project/target/environment` path.

## Listing versions

```bash
dotenv secret versions myapp/production/api
```

Paginate with `--limit` (default 50, max 100) and `--page`:

```bash
dotenv secret versions myapp/production/api --limit 100 --page 2
```

The table shows each version's ID, action, size, encryption key version, creation time, author,
and a lock status. Versions outside your plan's history window appear as `🔒 locked` and cannot
be shown or restored until you upgrade.

## Showing a version

Decrypt and print one version locally (plaintext never leaves your machine). The `--version`
flag is required and takes a version ID from the listing:

```bash
dotenv secret show myapp/production/api --version 42
```

Write the plaintext to a file instead of stdout:

```bash
dotenv secret show myapp/production/api --version 42 --output old.env
```

For client-managed projects you provide the current key with `--client-key`. A version
encrypted under a **rotated** client-managed key needs that old key — supply it with
`--old-key` (a file or value, repeatable) or enter it at the prompt; each candidate is verified
against the stored proof before use:

```bash
dotenv secret show myapp/production/api --version 42 \
  --client-key=./current.key --old-key=./old.key
```

## Diffing versions

Show which keys were added, removed, or changed. With a single version, the diff is taken
against the **live current secret**:

```bash
dotenv secret diff myapp/production/api 42
```

Pass a second version to diff one version against another (the diff reads from the first/older
state to the second):

```bash
dotenv secret diff myapp/production/api 42 50
```

Values are masked by default; reveal them with `--show-values`:

```bash
dotenv secret diff myapp/production/api 42 --show-values
```

The diff also accepts `--client-key` and `--old-key` for client-managed and rotated keys.

## Restoring a version

Restore is append-only: the current value is first preserved as a new version, then overwritten
with the restored one — nothing is lost. Restore by version ID:

```bash
dotenv secret restore 42
```

Skip the confirmation prompt with `-f` / `--force`:

```bash
dotenv secret restore 42 --force
```

A version encrypted under an old client-managed key must be re-encrypted under the current key
before it can be restored; for now, use the web dashboard for that case.

## Purging history

Permanently delete backup version history. This cannot be undone.

```bash
# Purge history for one level
dotenv secret purge myapp/production/api
```

| Flag | Description |
|------|-------------|
| `--project-wide` | Purge history for the entire project, not just this level. |
| `-f, --force` | Skip the confirmation prompt(s). |

A project-wide purge is the most destructive operation here — without `--force` you are asked to
type the project name back to confirm.

## Deleting with or without a backup

`dotenv secret delete <path>` clears a level's secret but keeps a backup version by default, so
it can be restored. Passing `--no-backup` to delete also **purges the entire version history and
hard-deletes** the secret — nothing can be recovered afterward.

> Note: these endpoints require a server that supports secret versioning. Older servers return
> 404 for them.
