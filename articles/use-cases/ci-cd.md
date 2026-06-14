---
title: CI/CD Pipelines
slug: ci-cd
order: 2
description: Inject secrets into any CI/CD pipeline with a read-only API key and a single dotenv pull step.
tags: [use-cases, ci-cd, cli, api-key, secrets, automation, security]
---

# CI/CD Pipelines

In CI/CD you can't log in through a browser, so DotEnv authenticates with an **organization API
key** passed through the environment. The pattern is the same on every platform: store a
read-only key in the platform's secret store, install the CLI, then `dotenv pull` your secrets
into a `.env` file the rest of the pipeline can load.

## The pattern

1. **Create a read-only API key** for the organization (see
   [authentication](/documentation/cli/authentication)).
2. **Store it in your CI platform's secret store** — never inline it in the pipeline file.
3. **Install the CLI and pull** inside a job step.

```bash
# Install the CLI
curl -sSL https://dotenv.cloud/install.sh | bash

# Authenticate via the environment and pull (read-only key)
DOTENV_API_KEY="$DOTENV_API_KEY" \
  dotenv pull myapp/production/api --output .env --quiet
```

When `DOTENV_API_KEY` is set, the CLI uses it directly and skips any local account store. The
`--quiet` flag suppresses non-error output so secrets never appear in build logs.

## Why read-only

CI runs untrusted code (dependencies, forks, generated scripts). A pipeline only ever needs to
**read** secrets, so the key it uses should be **read-only**. A leaked read-only key cannot
modify or delete your secrets. Create scoped, read-only keys with
[`dotenv apikeys create`](/documentation/cli/commands).

## Loading the .env

How you consume the file depends on the job:

```bash
# Load into the current shell for subsequent steps
set -a; . ./.env; set +a

# Or hand the file to a tool that reads .env (docker compose, frameworks, test runners)
```

For a container build, produce a Dockerfile-style or env file directly:

```bash
dotenv pull myapp/production/api --format dockerfile --output Dockerfile.env --quiet
```

## Keep the file ephemeral

The pulled `.env` should exist only for the life of the job. Don't cache it, don't upload it as
an artifact, and make sure `.env` is in `.gitignore` so it can never be committed from a build
checkout.

## Platform specifics

Each platform stores the read-only key in a different place and loads it slightly differently:

- **GitHub Actions** — use the [`dotenvcloud/action-github`](/documentation/integrations/github-actions)
  action, which wraps install + pull for you.
- **GitLab CI, Jenkins, CircleCI, Docker** — run the CLI in a shell step. See
  [CI/CD and containers](/documentation/integrations/ci-cd-and-containers) for one correct
  recipe per platform, including where each stores the key.

## Where to go next

- [Integrations overview](/documentation/integrations/overview) — GitHub Action vs CLI-in-shell.
- [GitHub Actions](/documentation/integrations/github-actions) — the official action.
- [Multi-environment workflows](/documentation/use-cases/multi-environment) — promote across
  stages.
- [Set up DotEnv with an AI agent](/guides/ai-setup) — let your coding agent wire up the pipeline.
