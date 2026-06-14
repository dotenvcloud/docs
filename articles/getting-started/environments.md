---
title: Environments & Cascading
slug: environments
order: 5
description: How production, staging, development, and local environments work — and how secrets cascade across the project, target, and environment levels.
tags: [getting-started, environments, cascading, merge, hierarchy, secrets]
---

# Environments & Cascading

Most teams run the same application in several places — a live **production** deployment, a
**staging** copy for testing, a shared **development** server, and each developer's **local**
machine. DotEnv models this with **environments**, and lets you avoid repeating yourself by
**cascading** shared values down from higher levels.

## Common environments

An environment is where secrets actually live. Each one carries a **status** that controls how it
is labeled and color-coded in the dashboard:

| Environment | Typical use |
|-------------|-------------|
| **production** | The live system serving real users. Holds your most sensitive values. |
| **staging** | A production-like copy for final testing before release. |
| **development** | A shared dev server or integration environment. |
| **local** | An individual developer's machine. |
| **custom** | Anything else — pick "custom" and name your own; it's slugified automatically. |

Environments can also be flagged **Is production** to mark that they hold live, sensitive values
and deserve extra care. See
[Projects, Targets & Environments](/documentation/web-app/projects-targets-environments) for how
to create and manage them.

## How secrets cascade

Secrets don't only live at the environment level. You can set them at three levels of the
hierarchy, and they **merge from the top down**:

```
Project          (shared by every target and environment)
  └── Target       (shared by every environment in that target)
       └── Environment   (specific to this one environment)
```

The merge order is **project → target → environment**, and **the most specific level wins**. A
value set on the project applies everywhere unless a target or environment redefines it.

### A worked example

Say you set these values:

```
# Project level (myapp)
API_BASE_URL = https://api.example.com
LOG_LEVEL    = info

# Target level (myapp/production)
LOG_LEVEL    = warn

# Environment level (myapp/production/api)
DATABASE_URL = postgres://prod-db:5432/myapp
```

When you pull the environment, the levels merge:

```bash
dotenv pull myapp/production/api --output .env
```

```
API_BASE_URL=https://api.example.com    # inherited from the project
LOG_LEVEL=warn                          # project value overridden by the target
DATABASE_URL=postgres://prod-db:5432/myapp   # set on the environment
```

`API_BASE_URL` came all the way from the project, `LOG_LEVEL` was overridden at the target, and
`DATABASE_URL` is environment-specific. You only define each value where it actually differs.

## Pulling a specific level

The path you give the CLI controls how deep the merge goes:

```bash
dotenv pull myapp                  # project level only
dotenv pull myapp/production       # project + target merged
dotenv pull myapp/production/api   # project + target + environment merged (most common)
```

To read **only** the level you named — ignoring inherited parent values — use `--level-only`:

```bash
# Only the secrets defined directly on the api environment
dotenv pull myapp/production/api --level-only
```

See [Pulling and pushing](/documentation/cli/pulling-and-pushing) for the full set of options.

## Putting it to use

A typical layout for one app:

- **Project level:** values identical everywhere (third-party base URLs, feature flags).
- **Target level:** values that differ per deployment context (a backend vs. a web target).
- **Environment level:** values that change per environment (production vs. local database URLs,
  per-environment API keys).

Each developer then keeps their machine current with one command:

```bash
dotenv pull myapp/development/local --output .env
```

## Next steps

- [Team setup](/documentation/getting-started/team-setup) — invite people and control who can
  reach which environment.
- [Managing Secrets](/documentation/web-app/managing-secrets) — the multi-level editor in the
  dashboard.
- [Pulling and pushing](/documentation/cli/pulling-and-pushing) — formats, interpolation, and
  merge controls.
