---
title: Multiple Environments
slug: multi-environment
order: 3
description: Manage production, staging, and development secrets with DotEnv targets and environments, and pull the right set per stage.
tags: [use-cases, multi-environment, production, staging, targets, environments, hierarchy]
---

# Multiple Environments

Real projects have more than one set of secrets: production, staging, and development at least,
often several services within each. DotEnv models this with a three-level hierarchy so you can
keep shared values in one place and override only what differs per stage.

## The hierarchy

Every path has the form:

```
project/target/environment
```

- **project** — the application (e.g. `myapp`).
- **target** — a deployment stage (e.g. `production`, `staging`, `development`).
- **environment** — a service or context within that stage (e.g. `api`, `worker`, `web`).

Values set on a shallower level are **inherited** by everything beneath them. A secret defined
on the project applies to every target and environment unless a deeper level redefines it.

```
myapp                       # shared by everything
myapp/production            # shared by all production environments
myapp/production/api        # specific to the production API
```

## Pull the right set per stage

`dotenv pull` merges the whole path — project → target → environment, deeper wins — into one
flat `.env`:

```bash
# Local development
dotenv pull myapp/development/local --output .env

# Staging deploy
dotenv pull myapp/staging/api --output .env --quiet

# Production deploy
dotenv pull myapp/production/api --output .env --quiet
```

Each command returns the fully merged values for that stage. You change one path segment to
target a different environment — the rest of your pipeline stays identical.

## Put shared values where they belong

Decide which level each secret lives on:

- **Project level** — values identical everywhere (e.g. a feature flag default, a vendor's
  public API base URL).
- **Target level** — values shared across a stage (e.g. the staging database host).
- **Environment level** — values unique to one service (e.g. that service's queue name).

This keeps each value in exactly one place. Update the staging database host once at
`myapp/staging` and every staging environment inherits it.

## See what a level resolves to

Pull merges by default. To inspect **only** the values defined directly on a level (ignoring
inherited parents), use `--level-only`:

```bash
dotenv pull myapp/production/api --level-only
```

This is useful when you're deciding whether a value belongs higher up the tree. Browse the whole
hierarchy with `dotenv tree` and `dotenv explore` (see the
[command reference](/documentation/cli/commands)).

## Promote a value between stages

Pull from one stage and push to another using a file as the carrier:

```bash
dotenv pull myapp/staging/api --output promote.env --level-only
dotenv push myapp/production/api promote.env
rm promote.env
```

Review the values before pushing — production usually needs different credentials than staging,
so you'll typically edit `promote.env` first.

## Where to go next

- [Local development](/documentation/use-cases/local-development) — the dev stage in detail.
- [CI/CD pipelines](/documentation/use-cases/ci-cd) — pull the right stage per pipeline.
- [Pulling and pushing](/documentation/cli/pulling-and-pushing) — paths, merge order, formats.
- [Team collaboration](/documentation/use-cases/team-collaboration) — who can see which stage.
