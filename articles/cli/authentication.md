---
title: Authentication
slug: authentication
order: 3
description: Authenticate the DotEnv CLI with browser OAuth or an API key, manage accounts and organizations, and set up CI.
tags: [cli, authentication, login, oauth, api-key, accounts, organizations, ci]
---

# Authentication

The DotEnv CLI supports two authentication methods:

- **OAuth (browser login)** — best for local development. Gives access to every organization
  you belong to and supports automatic token refresh.
- **Organization API key** — best for CI/CD and automation. Tied to a single organization, no
  browser required.

## OAuth login

```bash
dotenv login
```

This opens your browser to complete authentication. If you cannot open a browser (for example
over SSH), print the URL instead and open it manually:

```bash
dotenv login --no-browser
```

Other login flags:

```bash
# Pin the local OAuth callback port (default: auto)
dotenv login --callback-port=8765

# Show authentication status (same as `dotenv status`)
dotenv login --status
```

`dotenv login` automatically falls back to API key entry when it detects a non-interactive
session or when the `CI` environment variable is set.

## API key login

To store an organization API key interactively:

```bash
dotenv login --api-key
```

You will be prompted for the key (input is hidden), an organization slug, and an account name.
The key is stored encrypted in your local config.

You can also create an account interactively with `dotenv init` (which guides you through API
URL, telemetry, and authentication) or `dotenv account add`.

## Checking status

```bash
dotenv status
```

`status` shows the current account, its type, the active organization, OAuth token expiry, and
any other configured accounts. It never makes an API call — it reports local state. Use
[`dotenv auth info`](./commands.md#auth) to fetch your user profile and organization
memberships from the server.

## Accounts

The CLI can hold several accounts at once (for example a personal OAuth account and a
work API-key account). Manage them with `dotenv account`:

```bash
# List configured accounts (the current one is marked)
dotenv account list

# Switch the active account
dotenv account use work@example.com

# Add a new account interactively (choose OAuth or API key)
dotenv account add

# Remove an account (revokes the OAuth token on the server first)
dotenv account remove old-account

# Refresh the current OAuth account's access token
dotenv account refresh

# Rename an account
dotenv account rename old-name new-name
```

`dotenv refresh` is a top-level alias for `dotenv account refresh`.

### Logging out

```bash
# Log out the current account
dotenv logout

# Log out a specific account
dotenv logout work@example.com

# Log out every account
dotenv logout --all

# Skip the confirmation prompt
dotenv logout --force
```

For OAuth accounts, logout revokes the token on the server so the session is invalidated
everywhere. API key accounts are only removed locally — the key keeps working until you delete
it with [`dotenv apikeys delete`](./commands.md#apikeys).

## Organizations

An OAuth account can belong to several organizations; commands act on the currently selected
one. Manage them with `dotenv org`:

```bash
# List organizations for the current account
dotenv org list

# Switch organization (by slug or ULID; interactive if omitted)
dotenv org use acme-corp

# Refresh the cached organization list from the server
dotenv org refresh

# Show the current organization's details
dotenv org show
```

API key accounts are tied to a single organization and cannot switch.

## Authentication in CI/CD

In CI, authenticate with an API key passed through the environment — no `dotenv login` needed:

```bash
export DOTENV_API_KEY="your-org-api-key"
dotenv pull myapp/production/api --output .env
```

When `DOTENV_API_KEY` is set (or you pass the global `--api-key` flag), the CLI uses those
credentials directly and bypasses the local account store entirely. To scope an API key
client to a specific organization, also set `DOTENV_ORGANIZATION`:

```bash
export DOTENV_API_KEY="your-org-api-key"
export DOTENV_ORGANIZATION="acme-corp"
dotenv list projects
```

Create scoped, read-only API keys for automation with
[`dotenv apikeys create`](./commands.md#apikeys). See [Configuration](./configuration.md) for
the full list of environment variables.
