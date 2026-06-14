---
title: Troubleshooting
slug: troubleshooting
order: 9
description: Diagnose and fix common DotEnv CLI failures — authentication, organization selection, encryption keys, TLS, and more.
tags: [cli, troubleshooting, errors, debugging, auth, encryption]
---

# Troubleshooting

Run any command with `--debug` for extra diagnostic output:

```bash
dotenv --debug pull myapp/production/api
```

## Authentication

### "no accounts configured. Run 'dotenv login' to add an account"

You have no stored account and no `DOTENV_API_KEY` set. Either log in:

```bash
dotenv login
```

or, for CI, set an API key in the environment:

```bash
export DOTENV_API_KEY="your-org-api-key"
```

### OAuth token expired

`dotenv status` shows the token status. If it is expired, refresh it:

```bash
dotenv account refresh   # or: dotenv refresh
```

If the refresh token has also expired, log in again with `dotenv login`. API keys cannot be
refreshed — they keep working until deleted or rotated.

### "no organization selected"

Most commands act on the current organization. For OAuth accounts, list and select one:

```bash
dotenv org list
dotenv org use acme-corp
```

If the list is empty or stale, refresh it:

```bash
dotenv org refresh
```

### API key only shows one organization

This is expected. An organization API key is tied to a single organization. Use OAuth
(`dotenv login`) if you need to list or switch between multiple organizations.

## Encryption keys

### Wrong key on push (key proof mismatch)

When pushing to a client-managed project, the CLI verifies your key against the project's stored
proof and refuses the push if it does not match:

> the encryption key does not match project '…' — refusing to push secrets encrypted with a
> different key

Supply the project's established key (a wrong or mistyped key would orphan the level). See
[Client-side encryption](./client-side-encryption.md) for how the key is resolved.

### "no client-key verification configured"

The client-managed project has no proof set up. Re-establish its encryption key through the web
key setup, or recreate the project with `dotenv project create --storage client`.

### Cannot decrypt: encryption key not provided

The level holds encrypted content but no key was available. For a client-managed project, pass
`--client-key=<file>` (or set `DOTENV_CLIENT_KEY`). To fetch the raw encrypted blob without
decrypting, use `--decrypt=false`.

### A version needs an old key

A backup version encrypted under a rotated client-managed key requires that old key. Pass it
with `--old-key` (file or value, repeatable) or enter it at the prompt:

```bash
dotenv secret show myapp/production/api --version 42 --old-key=./old.key
```

Versions under old **server-managed** keys are never exposed by the API; use
`dotenv secret restore <id>` instead so the server re-encrypts onto the current key.

### Client-managed key cannot be rotated from the CLI

This is by design. Rotate client-managed keys from the web dashboard, then use
`dotenv key re-encrypt-history` to move old backup versions onto the new key locally.

## Secret versions

### 404 on version commands

The server may predate secret versioning. Confirm the version ID is correct; older servers
return 404 for these endpoints.

### Version is "locked"

The version falls outside your plan's history window. Upgrade your plan to view or restore it.
`dotenv secret versions <path>` marks locked versions.

## Network and TLS

### TLS certificate errors

The CLI requires valid TLS against real endpoints. `DOTENV_TLS_SKIP_VERIFY` is honored **only**
when the API URL points at a local development host (`localhost`, `127.0.0.1`, `*.test`,
`*.local`, `*.localhost`); it is ignored against `api.dotenv.cloud` and other real hosts.

For a local dev server:

```bash
export DOTENV_API_URL="https://dotenv.test"
export DOTENV_TLS_SKIP_VERIFY=1
dotenv list projects
```

### Wrong API endpoint

Check which API URL is in effect with `dotenv status`. Override it with `DOTENV_API_URL` or by
selecting the right account.

## Files and output

### "File … exists. Create backup?"

When `pull --output` targets an existing file, the CLI offers to back it up to `<file>.backup`
first. Answer yes to proceed, or use `--quiet` to skip the prompt (the existing file is still
backed up). Declining cancels the operation.

### Push asks to overwrite

The destination level already has secrets. Confirm the overwrite, or pass `--force` to skip the
prompt.

## Updating

### `dotenv update` cannot find a release

Verify network access to GitHub. If you installed via Homebrew, update with
`brew upgrade dotenv`; if via `go install`, re-run
`go install github.com/dotenvcloud/cli@latest`.

### Verifying your version

```bash
dotenv version          # full details
dotenv version --short  # version number only
```
