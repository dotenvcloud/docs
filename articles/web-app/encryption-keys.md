---
title: Encryption Keys
slug: encryption-keys
order: 6
description: Understand server-managed vs client-managed encryption, store/rotate/remove keys from Key Management, and learn what you must keep safe.
tags: [web-app, encryption, keys, security, rotation, pbkdf2]
---

# Encryption Keys

Every secret in DotEnv is encrypted with **AES-256-GCM**, using a key derived with **PBKDF2**.
Each project has its own encryption key, and you choose how that key is managed. You manage keys
from **Key Management** at `/dashboard/key-management`, which has three tabs: **Store**,
**Remove**, and **Rotate**.

## Two custody models

### Server-managed

DotEnv holds the encryption key for the project. This is the simplest option: you don't have to
remember a passphrase, and the server can decrypt on your behalf where needed (for example, for
server-side secret retrieval). The trade-off is that DotEnv has access to the key material.

### Client-managed

**You** hold the passphrase; the server never stores the key itself. Instead the server stores
only a non-secret PBKDF2 **proof** (and the salt/iteration parameters) that lets it verify the
key you provide is correct, without being able to reconstruct it. Your browser, the CLI, and the
SDKs all derive the same key from your passphrase to encrypt and decrypt.

With client-managed keys, **if you lose the passphrase, the secrets cannot be recovered** — not
even by DotEnv. That is the point: maximum privacy, but you own the responsibility.

## Switching custody (no re-encryption)

You can switch a project between server-managed and client-managed from a project's key storage
preferences:

- **Browser → server (store on server):** you confirm your account password, and the key you
  currently hold is stored on the server under the **same** salt — so no secrets are
  re-encrypted.
- **Server → browser (move to your browser):** you confirm your account password, the server
  reveals the key once so you can save it, you confirm you've backed it up, and the server then
  drops the stored key value (keeping only the verification proof). Again, **nothing is
  re-encrypted** — the underlying data key is preserved.

Because switching custody never re-encrypts your data, it is fast and safe. The key value simply
stops or starts being stored on the server.

## Rotating keys

Use the **Rotate** tab to replace a project's active key. Rotation:

- Generates a new active key version and **re-encrypts your secret versions** under it.
- Keeps previous key versions **immutable** so historical versions written under an old key can
  still be decrypted. (See [Secret Versioning](secret-versioning).)

Rotate when a key may have been exposed, on a regular schedule for hygiene, or when offboarding
someone who had access to a client-managed passphrase.

## Storing and removing keys

- **Store** (tab): provide/store the key for a project.
- **Remove** (tab): remove a stored key.

The exact options shown depend on whether the project is server- or client-managed.

## Key hint and recovery

When storing a client-managed key you can attach a **key hint** — a non-secret reminder to help
you recall which passphrase a project uses. You may also be able to generate a **recovery key**
as a backup path. A hint is *not* the key and should never contain the actual passphrase.

## What to keep safe

- **Client-managed passphrase:** store it in a password manager. There is **no reset** — losing
  it means losing access to the secrets.
- **Any key value the server reveals** during a server → browser switch: save it immediately and
  confirm before continuing.
- **Recovery keys:** treat them like the key itself.

## Quick reference

| Model | Who holds the key | Server can decrypt? | Recovery if passphrase lost |
| --- | --- | --- | --- |
| Server-managed | DotEnv | Yes | DotEnv holds the key |
| Client-managed | You | No (only verifies a proof) | Not possible |
