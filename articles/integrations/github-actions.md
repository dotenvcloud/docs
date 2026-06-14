---
title: GitHub Actions
slug: github-actions
order: 2
description: Pull DotEnv secrets in GitHub Actions with the official dotenvcloud/action-github action — all inputs, a full workflow, and secret setup.
tags: [integrations, github-actions, ci-cd, action, secrets, api-key]
---

# GitHub Actions

The official [`dotenvcloud/action-github`](https://github.com/dotenvcloud/action-github) action is
the supported way to use DotEnv on GitHub Actions. It installs the CLI, authenticates with a
secret, pulls your secrets, and writes a `.env` file — or exposes them as job environment
variables — in a few lines of YAML.

## Set up the secret first

The action needs a **read-only** organization API key, stored as a GitHub Actions secret. CI only
ever reads secrets, so the key it uses should not be able to modify or delete anything.

1. Create a read-only API key with [`dotenv apikeys create`](/documentation/cli/commands).
2. In your repository, go to **Settings → Secrets and variables → Actions → New repository
   secret**.
3. Name it `DOTENV_API_KEY` and paste the key.

Reference it in the workflow as `${{ secrets.DOTENV_API_KEY }}`. **Never** inline the key in the
workflow file.

## Minimal usage

```yaml
- name: Pull secrets
  uses: dotenvcloud/action-github@v1
  with:
    api-key: ${{ secrets.DOTENV_API_KEY }}
    project: myapp
    target: production
    environment: api
```

This writes the merged secrets to `.env` in the workspace. Later steps load that file however they
normally would.

## Full workflow example

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Pull DotEnv secrets
        uses: dotenvcloud/action-github@v1
        with:
          api-key: ${{ secrets.DOTENV_API_KEY }}
          project: myapp
          target: production
          environment: api
          output-file: .env
          export-variables: true

      # With export-variables: true, secrets are available as env vars here
      - name: Run deploy
        run: ./scripts/deploy.sh
```

Setting `export-variables: true` exposes each pulled secret as an environment variable for the
remaining steps in the job, in addition to writing the file. If you only want the file, omit it
and load `.env` yourself:

```yaml
      - name: Load .env
        run: |
          set -a; . ./.env; set +a
          ./scripts/deploy.sh
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `api-key` | Yes | — | Read-only organization API key. Use `${{ secrets.DOTENV_API_KEY }}`. |
| `project` | Yes | — | Project name. |
| `target` | No | — | Target (deployment stage) within the project. |
| `environment` | No | — | Environment within the target. |
| `output-file` | No | `.env` | Path to write the secrets file. |
| `format` | No | `env` | Output format: `env`, `json`, `yaml`, `shell`, `dockerfile`. |
| `export-variables` | No | `false` | Also expose pulled secrets as job environment variables. |
| `organization` | No | — | Organization slug, if the key isn't already scoped to one. |
| `api-url` | No | — | Override the API base URL (for self-hosted or staging APIs). |
| `decrypt` | No | `true` | Decrypt values. Set `false` to fetch raw encrypted blobs. |
| `resolve` | No | `false` | Expand `${VAR}` references between secrets before output. |
| `quiet` | No | `false` | Suppress non-error output. |
| `merge` | No | `true` | Merge the full path (project → target → environment). |
| `cli-version` | No | latest | Pin a specific CLI version to install. |

## Outputs

| Output | Description |
|--------|-------------|
| `env-file` | Path to the file the action wrote. |

Use it in later steps:

```yaml
      - name: Show where secrets landed
        run: echo "Wrote ${{ steps.dotenv.outputs.env-file }}"
```

(Give the DotEnv step an `id:` to reference its outputs.)

## Keep .env out of the repo

The pulled file holds real values. Make sure `.env` is gitignored so a build checkout can never
commit it back:

```gitignore
.env
.env.*
!.env.example
```

## Where to go next

- [CI/CD and containers](/documentation/integrations/ci-cd-and-containers) — the same idea on
  GitLab, Jenkins, CircleCI, and Docker.
- [Integrations overview](/documentation/integrations/overview) — when to use the Action vs the
  CLI.
- [CI/CD pipelines](/documentation/use-cases/ci-cd) — the general pattern and read-only keys.
- [Authentication](/documentation/cli/authentication) — creating and scoping API keys.
