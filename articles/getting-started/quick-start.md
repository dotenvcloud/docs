---
title: Quick Start
slug: quick-start
order: 3
description: Go from zero to a populated .env file in about five minutes — sign up, build your hierarchy, add a secret, then pull it with the CLI.
tags: [getting-started, quick-start, tutorial, cli, pull, secrets]
---

# Quick Start

This is the fastest path through DotEnv. In about five minutes you'll create an account, build a
project, add a secret in the dashboard, and pull it onto your machine as a `.env` file.

If you haven't installed anything yet, the [Installation](/documentation/getting-started/installation)
guide covers the account and the CLI in detail. Each step below links out if you want more depth.

## 1. Sign up

Open [dotenv.cloud](https://dotenv.cloud), **register**, and **verify your email**. When you sign
in you land on your dashboard with a first organization already created for you.

## 2. Create a project, target, and environment

DotEnv organizes secrets into a nested hierarchy:
**Organization → Project → Target → Environment → Secret**. You only need one of each to start.

1. Go to **Projects** and click **Create project**. Give it a name (for example `myapp`); the
   **slug** is derived from it. You'll use that slug from the CLI.
2. Open the project and **Create target** — a deployment context such as `production`.
3. Open the target and **Create environment** — for example `api`. This is where secrets actually
   live.

You now have the path `myapp/production/api`. See
[Create your first project](/documentation/getting-started/first-project) for the full walkthrough.

## 3. Add a secret in the dashboard

Open your project's secret editor, select **`myapp → production → api`**, and add a variable:

```
DATABASE_URL = postgres://localhost:5432/myapp
```

Save it. Values are encrypted at rest with AES-256-GCM. For more on adding and overriding
secrets across levels, see [Managing Secrets](/documentation/web-app/managing-secrets).

## 4. Install the CLI

If you haven't already:

```bash
# macOS / Linux
curl -sSL https://dotenv.cloud/install.sh | bash
```

```powershell
# Windows (PowerShell)
irm https://dotenv.cloud/install.ps1 | iex
```

Verify it:

```bash
dotenv version
```

Other install methods are listed in
[Installation](/documentation/getting-started/installation).

## 5. Log in

Authenticate the CLI with your account using browser-based login:

```bash
dotenv login
```

This opens your browser to complete sign-in. (For CI and automation you'd use a read-only API
key instead — see [Team setup](/documentation/getting-started/team-setup) and
[CLI Authentication](/documentation/cli/authentication).)

## 6. Pull your secrets into `.env`

Pull the environment you created and write it to a local `.env` file:

```bash
dotenv pull myapp/production/api --output .env
```

That's it. Your `.env` now contains:

```
DATABASE_URL=postgres://localhost:5432/myapp
```

Load it with whatever your framework already uses to read `.env` files. The CLI's job is to put
the right secrets on disk securely; how your app reads them stays the same.

> **Tip:** the path merges every level it touches. `myapp/production/api` combines
> project-level, target-level, and environment-level secrets, with the most specific level
> winning. See [Environments & cascading](/documentation/getting-started/environments).

## Next steps

- [Create your first project](/documentation/getting-started/first-project) — the hierarchy in
  full.
- [Environments & cascading](/documentation/getting-started/environments) — how secrets merge
  across levels.
- [Pulling and pushing](/documentation/cli/pulling-and-pushing) — formats, interpolation, and
  more CLI options.
