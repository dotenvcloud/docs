---
title: Client-Side Encryption
slug: client-side-encryption
order: 8
description: Server-managed vs client-managed encryption, the --client-key flag, key resolution order, and safety.
tags: [cli, encryption, security, client-key, keys]
---

# Client-Side Encryption

DotEnv projects use one of two encryption custody modes. Both encrypt secrets the same way; the
only difference is **who holds the key**.

## Server-managed vs client-managed

- **Server-managed** — the server stores the project's encryption key. When you pull, the
  server hands the CLI the key (and the derivation parameters) so it can decrypt locally. No key
  setup is required on your side.
- **Client-managed** — the server never sees the key. It stores only a non-secret proof used to
  verify the key, not the key itself. When you pull or push, you must supply the key, and all
  encryption/decryption happens on your machine.

For both modes the AES key is derived from your key string with PBKDF2 using a salt and
iteration count, so the CLI and the web app derive the same key and can read each other's data.

## Choosing a mode at project creation

`dotenv project create` defaults to **client-managed**:

```bash
# Client-managed (default): you choose and keep the key
dotenv project create my-app

# Provide the key explicitly (file recommended)
dotenv project create my-app --client-key=./encryption.key

# Server-managed: let the server hold the key
dotenv project create my-app --storage server
```

When you create a client-managed project, the CLI derives a PBKDF2 proof from your key and
registers only the proof — never the key. Keep your key safe: it is required to push and pull
and cannot be recovered from the server.

## Using a client-managed key with pull/push

Both `pull` and `push` accept `--client-key`, which can be a file path or the literal key value:

```bash
# From a key file (recommended)
dotenv pull myapp/production/api --client-key=./encryption.key

# Push, encrypting with the same key
dotenv push myapp/production/api .env --client-key=./encryption.key
```

If you pass `--client-key` to a **server-managed** project, the CLI ignores it and uses the
server key (with a warning).

## Key resolution order

For a client-managed project, the CLI resolves the key in this order — safest first:

1. **`--client-key=<file>`** — a path to a key file. Recommended; no warning.
2. **`--client-key=<value>`** — the literal key string. Warned: it can leak through shell
   history and the process list. A value that *looks* like a path but does not exist is treated
   as an error (to catch typos), not silently used as a wrong key.
3. **`DOTENV_CLIENT_KEY`** — the key value from the environment. Warned: it can leak through the
   process environment. Consulted only when a client key is actually needed, so a stray global
   value cannot affect a server-managed project.
4. **Interactive prompt** — if none of the above is provided.

```bash
# Prefer files over values
dotenv pull myapp/production/api --client-key=./key.txt        # safest
dotenv pull myapp/production/api --client-key="my-passphrase"  # warned
DOTENV_CLIENT_KEY="my-passphrase" dotenv pull myapp/production/api  # warned
```

Keys are used as raw strings — they are never hex- or base64-decoded — so a key file written
with a trailing newline still matches (the CLI trims surrounding whitespace).

## Safety behaviors

- **Wrong-key protection on push.** For client-managed projects the CLI derives a key proof that
  the server verifies before accepting a write. A mistyped or wrong key is rejected up front
  rather than orphaning the level under an unrecoverable key.
- **Plaintext guard.** Pushing with `--encrypt=false` to a client-managed project would upload
  plaintext and defeat client-side encryption, so it requires an explicit confirmation.
- **Local-only plaintext.** `secret show`, `secret diff`, and `key re-encrypt-history` decrypt
  on your machine; plaintext never leaves it.

## Rotating keys

- **Server-managed:** rotate from the CLI. The server generates a new key and re-encrypts the
  current secrets.

  ```bash
  dotenv key rotate my-app
  # Move backup versions onto the new key too (after a key exposure):
  dotenv key rotate my-app --history-policy=re_encrypt
  ```

- **Client-managed:** the active key cannot be rotated from the CLI (rotation needs to
  re-encrypt every secret against IDs the public API does not expose) — use the web dashboard.
  After rotating in the dashboard, you can move old backup versions onto the new key locally:

  ```bash
  dotenv key re-encrypt-history my-app --old-key=./old.key --client-key=./new.key
  ```

  Old keys are verified against their stored proof before use, and the operation is resumable —
  re-run it to continue where it stopped.

Inspect a project's key versions any time with:

```bash
dotenv key history my-app
```
