---
title: Common Issues
slug: common-issues
order: 1
description: Symptom-cause-fix guide for the most frequent DotEnv problems — authentication failures, empty pulls, .env not loading, client-key mismatches, TLS errors, and permission denials.
tags: [troubleshooting, cli, authentication, encryption, permissions, tls]
---

# Common Issues

This page lists the problems people hit most often, organized as **symptom → cause → fix**. Most
involve the CLI; some apply to any client (SDKs, API). For the precise error payloads behind these,
see the [Error Reference](/documentation/troubleshooting/error-reference).

> Tip: run any CLI command with `--debug` for extra diagnostic output, and run `dotenv status` to
> see which account, organization, and API URL are currently in effect.

## Authentication fails

**Symptom** — Commands report `authentication failed`, `your session has expired`, or
`no accounts configured`.

**Cause** — No credentials are configured, the supplied API key is wrong, or an OAuth session has
expired.

**Fix**

- Check what's configured with `dotenv status`.
- If nothing is configured, authenticate with one of:
  - `dotenv login` — interactive OAuth login (lets you list/switch organizations).
  - `dotenv login --api-key <key>` — log in with an organization API key.
  - `export DOTENV_API_KEY=<key>` — set the key in the environment (best for CI).
- If `dotenv status` shows an **expired** OAuth token, refresh it with `dotenv account refresh`.
  If the refresh token has also expired, run `dotenv login` again. API keys never need refreshing —
  they keep working until deleted or rotated.

See [Member Management](/documentation/administration/member-management) for switching
organizations.

## A pull returns nothing (empty result)

**Symptom** — `dotenv pull` succeeds but returns no secrets, or far fewer than expected.

**Cause** — Usually the wrong **path** (project/target/environment) or a **token scoped** so
narrowly it can't see the level you asked for. An empty level legitimately returns nothing.

**Fix**

- Verify the path components exist and are spelled correctly:

  ```bash
  dotenv list projects
  dotenv list targets <project>
  dotenv list environments <project>/<target>
  ```

- Confirm you're pointed at the right organization with `dotenv status`; if not, select it
  (`dotenv org list`, then `dotenv org use <organization>`).
- If you authenticate with an API key whose permissions are **scoped** to specific
  projects/targets/environments, a path outside that scope returns nothing (or a permission error).
  Check the key's scopes — see
  [Roles & Permissions](/documentation/administration/roles-and-permissions).

## `.env` file isn't loading

**Symptom** — Your application can't see the variables you pulled.

**Cause** — The pull output never reached a file your app reads, or it went to the wrong location.

**Fix**

- `dotenv pull` writes to stdout unless you direct it. Send it to a file, or use `--output`:

  ```bash
  dotenv pull <project>/<target>/<environment> --format env > .env
  # or
  dotenv pull <project>/<target>/<environment> --output .env
  ```

- Make sure the file is where your framework/runtime expects it (typically the project root) and
  that your app actually loads a `.env` file.
- If `--output` targets an existing file, the CLI offers to back it up to `<file>.backup` first.
  Declining cancels the write — so the file you expected may be unchanged.

There is **no** `dotenv run` command; the CLI writes/export the values, and your app or shell loads
them.

## Client-managed key mismatch

**Symptom** — Pull fails to decrypt, or push is refused with a message about the encryption key not
matching the project ("key proof mismatch").

**Cause** — The project uses **client-managed encryption** and the key you supplied is wrong,
mistyped, or simply not the key the project was established with. On push, the CLI verifies your key
against the project's stored proof and refuses rather than orphaning the level with content nobody
can decrypt.

**Fix**

- Supply the project's **established** key. The CLI resolves a client key in this order (safest
  first):
  1. `--client-key <file>` — path to a file containing the key (recommended; no warning).
  2. `--client-key <value>` — the key value itself (warned: leaks via shell history / process list).
  3. `DOTENV_CLIENT_KEY=<value>` — environment variable holding the key value (warned).
  4. Interactive prompt — when none of the above is provided.
- A `--client-key` value that looks like a path but doesn't exist is treated as an **error**, not a
  silent key, so typos are caught instead of producing a wrong-key mismatch.
- If you genuinely lost the key, you cannot recover the encrypted content — the server never has it.
  Re-establish the project's key from the web dashboard.

See [Client-side encryption](/documentation/cli/client-side-encryption) for how keys are resolved.

## TLS / certificate errors

**Symptom** — Requests fail with a TLS or certificate verification error.

**Cause** — The endpoint's certificate can't be verified — commonly when talking to a local
development server with a self-signed certificate.

**Fix**

- Against the production API the CLI **requires** valid TLS; fix the certificate or network path
  rather than disabling verification.
- For **local development only**, `DOTENV_TLS_SKIP_VERIFY=1` skips verification — but it is honored
  **only** when the API URL points at a local host (`localhost`, `127.0.0.1`, `::1`, or a `.test`,
  `.local`, or `.localhost` domain). It is ignored against real hosts like `api.dotenv.cloud`, so a
  stray value can't silently expose your tokens and secrets to a man-in-the-middle.

  ```bash
  export DOTENV_API_URL="https://dotenv.test"
  export DOTENV_TLS_SKIP_VERIFY=1
  dotenv list projects
  ```

## Permission denied

**Symptom** — `access denied to <resource>`, or an API call returns `403`.

**Cause** — Your account's **role**, or your API key's **scopes**, don't include the permission the
action requires.

**Fix**

- Confirm you're operating in the right organization (`dotenv status`).
- Check that your role grants the action. Read-only roles (e.g. **Auditor**) and read-only API keys
  can't write, rotate keys, or restore versions — see
  [Roles & Permissions](/documentation/administration/roles-and-permissions).
- For automation, mint or adjust an API key with the required permission scope.
- If you should have access but don't, ask an **Owner** or **Administrator** of the organization to
  adjust your role.
