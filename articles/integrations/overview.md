---
title: Integrations Overview
slug: overview
order: 1
description: The two real ways to integrate DotEnv — the official GitHub Action, or the CLI in any shell step — and how to choose between them.
tags: [integrations, overview, github-actions, cli, ci-cd, decision-guide]
---

# Integrations Overview

DotEnv integrates with the tools you already use through exactly two mechanisms. There are no
buildpacks, platform plugins, or cloud-native connectors to install — and that's deliberate.
Everything routes through the same CLI, so behavior is identical everywhere.

## The two real patterns

### 1. The GitHub Action

If you run on GitHub Actions, use the official
[`dotenvcloud/action-github`](/documentation/integrations/github-actions) action. It installs the
CLI, authenticates with a secret, pulls your secrets, and writes a `.env` file (or exports them as
job environment variables) — in a few lines of YAML.

### 2. The CLI in any shell step

Everywhere else — GitLab CI, Jenkins, CircleCI, Docker builds, your own scripts — you run the
**same CLI** inside a normal shell step:

```bash
curl -sSL https://dotenv.cloud/install.sh | bash
DOTENV_API_KEY="$DOTENV_API_KEY" \
  dotenv pull myapp/production/api --output .env --quiet
```

Any platform that can run a shell command and read a secret from its own store can use DotEnv.
See [CI/CD and containers](/documentation/integrations/ci-cd-and-containers) for one correct
recipe per platform.

## Decision guide

| Your platform | Use |
|---------------|-----|
| GitHub Actions | The [GitHub Action](/documentation/integrations/github-actions) |
| GitLab CI / Jenkins / CircleCI | [CLI in a shell step](/documentation/integrations/ci-cd-and-containers) |
| Docker build | [CLI with BuildKit secrets](/documentation/integrations/ci-cd-and-containers) |
| Any other CI, PaaS, or script | CLI in a shell step |
| Local development | [`dotenv pull` to a file](/documentation/use-cases/local-development) |

The rule of thumb: **on GitHub, use the Action; everywhere else, run the CLI.**

## What's not an integration

DotEnv does not ship CircleCI orbs, Heroku buildpacks, Netlify/Vercel plugins, or AWS/Azure/GCP
"native" connectors, and there is no `dotenv run` wrapper. Those would each be a thin shell around
the same `dotenv pull`, so instead you call the CLI directly. The CLI's only job is to produce a
`.env` file (or formatted output) that your platform then loads itself.

## Security model (applies to every integration)

- **Use read-only API keys.** CI and builds only need to read secrets. A leaked read-only key
  can't change or delete anything.
- **Store the key in the platform's secret store** — GitHub Actions secrets, GitLab masked +
  protected variables, Jenkins credentials, CircleCI contexts. Never inline a key in a config
  file or commit it.
- **Keep `.env` ephemeral and gitignored** so pulled secrets never persist or get committed.

## Where to go next

- [GitHub Actions](/documentation/integrations/github-actions) — the official action and a full
  workflow.
- [CI/CD and containers](/documentation/integrations/ci-cd-and-containers) — GitLab, Jenkins,
  CircleCI, Docker.
- [CI/CD pipelines](/documentation/use-cases/ci-cd) — the general pattern and why read-only.
- [Set up DotEnv with an AI agent](/guides/ai-setup) — let your coding agent pick and wire the
  right integration.
