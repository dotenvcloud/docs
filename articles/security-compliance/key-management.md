---
title: Key Management
slug: key-management
order: 2
description: Server-managed vs client-managed encryption keys, how to choose, key rotation and history re-encryption, recovery, and what you must keep safe.
tags: [security, keys, key-management, rotation, client-managed, server-managed, zero-knowledge]
---

# Key Management

DotEnv encrypts every secret value with a key derived from your project's key material (see
[Encryption Model](/documentation/security-compliance/encryption)). How that key is *stored*
determines whether DotEnv can decrypt your secrets for you, or whether only you can. There are
two custody modes, and you choose per project.

## Server-managed vs client-managed

| | Server-managed | Client-managed |
| --- | --- | --- |
| Who holds the key | DotEnv stores the (encrypted) project key | **You** hold the passphrase; DotEnv never receives it |
| What DotEnv stores | The encrypted data key | Only a PBKDF2 **key-check proof** (salt + iterations + proof) |
| Can DotEnv decrypt? | Yes, for authorized requests | **No** — zero-knowledge |
| Web editor / server-side decrypt | Available | Not available (decrypt happens on your machine) |
| If you lose the key | DotEnv can recover access | **Secrets are unrecoverable** |

In **server-managed** mode, DotEnv can derive the data key and decrypt values for authorized
callers — convenient for the web dashboard and for API tokens granted `secret:decrypt`.

In **client-managed** mode, DotEnv stores **only** a key-check proof — a salt, an iteration
count, and a PBKDF2-derived proof value. The proof lets the server confirm a write used the
correct key (so a wrong key can't silently corrupt your data), but it **cannot** be reversed to
recover the key or the plaintext. The platform therefore cannot decrypt your secrets at all.

## Choosing a mode

- Choose **server-managed** when you want the convenience of the web editor and server-side
  decryption, and you trust DotEnv to hold the key. This is the simplest setup.
- Choose **client-managed** when you need **zero-knowledge** guarantees — i.e. you require that
  DotEnv (or anyone who compromises it) is *unable* to read your secret values. You accept full
  responsibility for storing the passphrase, because losing it means the secrets cannot be
  recovered by anyone.

You can switch a project between modes. Because the data key derivation is unified across both
modes, **switching custody never re-encrypts your secrets** — the derived data key stays the same.

## Key rotation

Rotating replaces a project's key material with a new key going forward, while keeping older
secret versions readable.

- **Server-managed rotation** re-encrypts on the server: the platform decrypts under the old key
  and re-encrypts under the new one.
- **Client-managed rotation** happens **client-side**: your machine decrypts with the old
  passphrase and re-encrypts with the new one, then submits the result. The new passphrase never
  reaches DotEnv.

### Rotated keys are immutable

When a key is rotated, the old key record is **frozen**: its proof, salt, and iteration count
can never be changed afterward. This is deliberate — **historical secret versions stay
decryptable** only while the salt and proof that match them survive. DotEnv enforces this at the
model level and will refuse any attempt to mutate the proof material of a rotated key.

### Re-encrypting history

Rotation protects new writes, but older versions are still encrypted under the previous key. If
you need *every historical version* to be readable under the new key (for example to fully retire
an exposed key), use the **re-encrypt-history** flow, which walks the version history and
re-encrypts each older version under the current key. For client-managed projects this runs on
your machine with both the old and new passphrases.

## Recovery

- **Server-managed:** if a team member leaves or loses access, the project key is still held by
  DotEnv, so access is recoverable through the dashboard and your organization's normal access
  controls.
- **Client-managed:** there is **no recovery path** if the passphrase is lost. DotEnv holds only
  a one-way proof. Store the passphrase in a password manager or secrets vault, and make sure more
  than one trusted person (or a secure backup) can reach it.

## What to keep safe (read this)

- **Never paste a live key, passphrase, or secret value into an AI chat, a ticket, a chat
  message, or a CI log.** If you do, treat it as compromised and rotate immediately.
- **Don't commit keys to git.** Keep key files out of the repo (add them to `.gitignore`) and out
  of the `.env` you commit by accident.
- **Pass keys to the CLI by file, not by value.** Use `dotenv ... --client-key=<file>` rather than
  the raw key string — a literal value can leak through shell history and process listings. See
  [Client-Side Encryption (CLI)](/documentation/cli/client-side-encryption).
- **In CI, inject keys via the runner's secret store**, not as plaintext in the workflow file, and
  prefer file-based mounting so the value never appears in command output.

## Related

- [Encryption Model](/documentation/security-compliance/encryption)
- [Encryption Keys (web)](/documentation/web-app/encryption-keys)
- [Client-Side Encryption (CLI)](/documentation/cli/client-side-encryption)
- [Best Practices](/documentation/security-compliance/best-practices)
