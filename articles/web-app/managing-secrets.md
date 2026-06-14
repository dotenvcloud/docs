---
title: Managing Secrets
slug: managing-secrets
order: 4
description: Use the Multi-Level Secret Editor to edit secrets across project, target, and environment levels, with formats, overrides, merge behavior, and variable interpolation.
tags: [web-app, secrets, editor, overrides, merge, interpolation]
---

# Managing Secrets

Secrets are edited in the **Multi-Level Secret Editor** at `/dashboard/secrets/editor`. It is
the single place to view and change the variables for a project, across all three levels of the
hierarchy at once.

## Opening the editor

1. Go to **Secret Editor** (`/dashboard/secrets/editor`).
2. Pick the **project**, then the **target**, then the **environment** you want to work on.

Once a selection is made, the editor loads the secrets defined at each level and shows how they
combine.

## The three levels

The editor works on three levels simultaneously:

- **Project** — defaults shared across the whole project.
- **Target** — values for a specific deployment context within the project.
- **Environment** — values for a specific environment.

You edit each level independently. What you can edit depends on your permissions: the editor
checks read/write access for each level separately, so you may be able to view some levels and
edit others.

## Overrides and the merge (most-specific-wins)

When the same variable name is defined at more than one level, the **most specific level wins**:

```
environment  overrides  target  overrides  project
```

So a `DATABASE_URL` set at the project level is used everywhere — unless a target or environment
defines its own `DATABASE_URL`, in which case that more-specific value takes over. The editor
shows you the **merged** result so you can see exactly which value will be used and where it
came from, and it can show a **diff** so you can review changes before saving.

This is the same merge the CLI and SDKs apply when they pull secrets.

## Formats: plain, JSON, and YAML

You can author secrets in different formats:

- **Plain** — standard `KEY=value` lines, like a `.env` file.
- **JSON**
- **YAML**

The format is parsed **in your browser** into key/value pairs. The server only ever stores the
**encrypted blob** — it does not parse or read your formatted content. This keeps your values
private while letting you write them in whichever style you prefer.

## Variable interpolation (`${VAR}`)

You can reference other variables using `${VAR}` syntax, for example:

```
HOST=db.internal
DATABASE_URL=postgres://user@${HOST}/app
```

Important: **interpolation is resolved by the CLI/SDK when secrets are pulled**, not by the
server. The dashboard stores the literal `${VAR}` text; the substitution happens at the point of
use. This means references can resolve against values merged from any level.

## Saving

You can save **one level at a time** or save **all levels at once** (a bulk save). On success
the editor confirms how many levels were saved. Saves are encrypted before they leave your
browser according to the project's encryption setup — see [Encryption Keys](encryption-keys).

## Editor modes

The editor offers a richer code-style editor and a simpler text-area view; your preference is
remembered per user. You can also work in an individual or bulk editing mode depending on
whether you want to focus on one level or manage several together.

## Permissions

If you cannot edit a level, the editor tells you and prevents the change. Write access is granted
through your role in the organization (and, for API access, through the permissions on an API
key). See [Teams & Members](teams-and-members) and [API Keys](api-keys).

## Related

- Turn on history and restore for a project: [Secret Versioning](secret-versioning).
- Control how the project is encrypted: [Encryption Keys](encryption-keys).
