---
title: Secret Inheritance
slug: secret-inheritance
order: 3
description: How DotEnv merges the project, target, and environment levels into one set of values — most-specific-wins — plus how ${VAR} interpolation is resolved at pull time.
tags: [core-concepts, inheritance, merge, override, interpolation, secrets]
---

# Secret Inheritance

Secrets live at three levels — **project**, **target**, and **environment**. When you retrieve
them, DotEnv doesn't make you pick just one level; it **merges** them into a single final set of
values. This is what lets you define a value once at a broad level and override it only where it
differs.

## The merge: most-specific-wins

When you pull a full path (project → target → environment), the levels are combined in this
order:

```
environment  overrides  target  overrides  project
```

The **most specific level wins**. A variable set on the project applies everywhere — unless a
target redefines it, and unless an environment redefines it on top of that. Variables that exist
only at one level simply pass through.

This is the same merge whether you use the dashboard's Multi-Level Secret Editor or the CLI. It
happens on the **client**, after the encrypted blobs for each level are decrypted — the server
stores each level separately and never pre-merges them.

## A worked example

Suppose `acme-store/backend/production` has these values defined at each level:

| Variable | Project | Target (`backend`) | Environment (`production`) |
| --- | --- | --- | --- |
| `APP_NAME` | `Acme Store` | — | — |
| `LOG_LEVEL` | `info` | `debug` | — |
| `DATABASE_URL` | `postgres://dev-db/acme` | — | `postgres://prod-db/acme` |
| `STRIPE_SECRET_KEY` | — | — | `sk_live_123` |

Pulling the full `acme-store/backend/production` path produces the merged result:

```env
APP_NAME=Acme Store              # from project (not overridden anywhere)
LOG_LEVEL=debug                  # target overrides project
DATABASE_URL=postgres://prod-db/acme   # environment overrides project
STRIPE_SECRET_KEY=sk_live_123    # only defined on the environment
```

Reading it level by level:

- `APP_NAME` exists only on the project, so it carries all the way through.
- `LOG_LEVEL` is `info` on the project but the `backend` target sets `debug`; the more-specific
  target value wins.
- `DATABASE_URL` is overridden again at the environment level, the most specific of all.
- `STRIPE_SECRET_KEY` only exists at the environment level and appears as-is.

## Pulling a single level

You don't have to merge. With the CLI you can pull only the level you named, ignoring inherited
parent values, with `--level-only`:

```bash
# Only the secrets defined directly on the production environment
dotenv pull acme-store/backend/production --level-only
```

That would return just `DATABASE_URL` and `STRIPE_SECRET_KEY` from the example above — not the
project- and target-level values. By default, though, the merge is on.

## Variable interpolation (`${VAR}`)

Secrets can reference other secrets using `${VAR}` syntax. For example:

```env
HOST=api.acme.com
API_BASE_URL=https://${HOST}/v1
```

Interpolation is **not** resolved by the server. DotEnv stores `${HOST}` literally inside the
encrypted blob. References are expanded by the **client** at pull time, and only when you ask for
it. With the CLI, opt in with `--resolve` (`-r`):

```bash
dotenv pull acme-store/backend/production --resolve
```

With `--resolve`, `API_BASE_URL` comes out as `https://api.acme.com/v1`. Without it, the raw
`https://${HOST}/v1` is returned untouched. References are resolved against the **merged** set, so
a `${VAR}` can point at a value that was inherited from a broader level. If a reference can't be
resolved, the CLI warns and leaves the raw value in place rather than guessing.

## Why this design

- **Define once, override narrowly.** Shared defaults live high in the hierarchy; only true
  differences are repeated deeper.
- **Predictable precedence.** "Most specific wins" is the single rule you need to remember.
- **Privacy preserved.** Because merging and interpolation happen client-side, the server can
  store everything encrypted without ever interpreting your values. See
  [Security Model](/documentation/core-concepts/security-model).

For the levels themselves, see [The DotEnv Hierarchy](/documentation/core-concepts/hierarchy).
