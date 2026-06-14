---
title: Local Development
slug: local-development
order: 1
description: Pull your project's secrets into a local .env file for everyday development, and keep that file out of git.
tags: [use-cases, local-development, cli, env, secrets, dotenv]
---

# Local Development

The most common way to use DotEnv is also the simplest: pull the secrets for your development
environment into a local `.env` file and let your framework load it the way it always has. No
secrets live in your repository, and every teammate gets the same values from a single source of
truth.

## Install the CLI

```bash
curl -sSL https://dotenv.cloud/install.sh | bash
```

See the [CLI installation guide](/documentation/cli/installation) for Homebrew, Windows, and
other methods.

## Authenticate once

For local work, log in through your browser. This stores a token in your local config so you
don't have to re-authenticate every time.

```bash
dotenv login
```

## Pull secrets into .env

DotEnv organizes secrets as `project/target/environment`. Pull the environment you develop
against into a `.env` file:

```bash
dotenv pull myapp/development/local --output .env
```

`pull` merges every level in the path — project, then target, then environment — with deeper
levels overriding shallower ones. The result is a flat `.env` file your app can load directly.

Your framework picks it up automatically:

- **Laravel / PHP** — `.env` is read on boot.
- **Node.js** — `dotenv`, `dotenvx`, Vite, and Next.js all read `.env`.
- **Rails / Python / Go** — use your usual `.env` loader (`dotenv`, `python-dotenv`, etc.).

If you need a different shape, choose a format:

```bash
dotenv pull myapp/development/local --output config.json --format json
```

Available formats are `env`, `json`, `yaml`, `shell`, and `dockerfile`. See
[pulling and pushing](/documentation/cli/pulling-and-pushing) for the full list.

## Keep .env out of git

A pulled `.env` contains real secret values. **Never commit it.** Add it to `.gitignore`:

```gitignore
# .gitignore
.env
.env.*
!.env.example
```

You can safely commit a `.env.example` with empty or placeholder values so teammates know which
keys to expect — the real values come from `dotenv pull`.

## Refreshing after a change

When a teammate updates a secret in DotEnv, pull again to get the latest values:

```bash
dotenv pull myapp/development/local --output .env
```

If a `.env` already exists, the CLI offers to back it up before overwriting.

## A note on what the CLI does (and doesn't)

The CLI's job is to **produce a `.env` file or formatted output**. It does not run or wrap your
process — there is no `dotenv run`. You start your app exactly as you always have; the CLI just
keeps `.env` populated. If you want to load the values into your current shell instead of a
framework, source the file:

```bash
dotenv pull myapp/development/local --output .env
set -a; . ./.env; set +a   # export every variable into the shell
```

## Where to go next

- [Multi-environment workflows](/documentation/use-cases/multi-environment) — manage dev,
  staging, and production.
- [Team collaboration](/documentation/use-cases/team-collaboration) — onboard teammates.
- [Migrating from .env files](/documentation/use-cases/migrating-from-dotenv-files) — import what
  you already have.
- [Set up DotEnv with an AI agent](/guides/ai-setup) — let your coding agent wire this up for you.
