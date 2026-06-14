---
title: Configuration
slug: configuration
order: 6
description: Where the DotEnv CLI stores config, every environment variable it reads, and how settings take precedence.
tags: [cli, configuration, config, environment-variables, env]
---

# Configuration

## Config directory and file

By default the CLI stores its configuration under `~/.dotenv/`:

- Config file: `~/.dotenv/config.yaml`
- The directory is created automatically (with `0700` permissions) on first run.

The config file holds your accounts, the current account, telemetry preference, and other
preferences. Stored credentials (API keys and OAuth access/refresh tokens) are encrypted at
rest, and the file is written with `0600` permissions.

You can point the CLI at a different config file or directory:

```bash
# Use a specific config file for one invocation
dotenv --config /path/to/config.yaml status

# Use a different config directory (config.yaml is created inside it)
export DOTENV_CONFIG_DIR=/path/to/dir
dotenv status
```

## Environment variables

| Variable | Purpose |
|----------|---------|
| `DOTENV_API_KEY` | API key for authentication. When set, the CLI uses it directly and bypasses the local account store (CI/CD). |
| `DOTENV_API_URL` | Override the API base URL (default `https://api.dotenv.cloud`). |
| `DOTENV_ORGANIZATION` | Organization to scope requests to when authenticating with an API key. |
| `DOTENV_CONTEXT` | Selects a stored context by name (legacy config field). |
| `DOTENV_CLIENT_KEY` | Client-managed encryption key **value** (not a file path). Consulted only when a client key is needed. |
| `DOTENV_DEBUG` | Enable debug output (truthy: `1`, `true`, `yes`, `on`). |
| `DOTENV_NO_COLOR` | Disable colored output (truthy values as above). |
| `DOTENV_QUIET` | Suppress non-error output (truthy values as above). |
| `DOTENV_TLS_SKIP_VERIFY` | Skip TLS certificate verification — **only honored when the API URL points at a local dev host** (`localhost`, `127.0.0.1`, `*.test`, `*.local`, `*.localhost`). Ignored against real endpoints. |
| `DOTENV_CONFIG_DIR` | Directory that holds `config.yaml` (overrides the default `~/.dotenv`). |

Boolean variables accept `1`, `true`, `yes`, or `on` (case-insensitive) as true.

### Examples

```bash
# CI/CD: authenticate with an API key, scoped to an organization
export DOTENV_API_KEY="your-org-api-key"
export DOTENV_ORGANIZATION="acme-corp"
dotenv pull myapp/production/api --output .env

# Point at a self-hosted / staging API
export DOTENV_API_URL="https://api.staging.example.com"
dotenv list projects

# Supply a client-managed key value (prefer a file via --client-key instead)
export DOTENV_CLIENT_KEY="my-passphrase"
dotenv pull myapp/production/api
```

## Precedence

### Authentication

1. An explicit API key — the global `--api-key` flag, or `DOTENV_API_KEY` — wins. It bypasses
   the local account store and is treated as ephemeral CI/CD credentials.
2. Otherwise the current stored account (set with `dotenv account use`) is used.
3. Otherwise no credentials are available; run `dotenv login`.

### API URL

1. `DOTENV_API_URL` if set.
2. The current account's stored API URL.
3. The default, `https://api.dotenv.cloud`.

### Output flags

Flags like `--debug`, `--quiet`, and `--no-color` can be set either on the command line or via
their `DOTENV_*` environment variables; the command-line flag takes effect when present.

### Client-managed encryption key

Resolved per command, safest first:

1. `--client-key=<file>` — a key file (recommended).
2. `--client-key=<value>` — the literal key value (warned: leaks via shell history).
3. `DOTENV_CLIENT_KEY` — the key value from the environment (warned).
4. Interactive prompt.

See [Client-side encryption](./client-side-encryption.md) for details.
