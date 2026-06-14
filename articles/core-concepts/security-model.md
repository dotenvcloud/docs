---
title: Security Model
slug: security-model
order: 4
description: How DotEnv encrypts secrets with AES-256-GCM, the difference between server-managed and client-managed (zero-knowledge) keys, and exactly what DotEnv can and cannot see.
tags: [core-concepts, security, encryption, aes-gcm, pbkdf2, zero-knowledge, keys]
---

# Security Model

DotEnv is built so that secrets are encrypted before they are stored, and so that — when you want
it — DotEnv itself can never read your plaintext values. This page explains the encryption, the
two key-custody models, and the precise boundary of what DotEnv can and cannot see.

## Encryption

Every secret value is encrypted with **AES-256-GCM**, an authenticated encryption scheme that
provides both confidentiality and integrity (tampering is detected on decryption).

- The encryption key is **derived with PBKDF2** (a key-derivation function with a salt and many
  iterations).
- Each ciphertext is stored as **`base64(IV + ciphertext + tag)`** — a 12-byte GCM IV, the
  ciphertext, and the GCM authentication tag, concatenated and base64-encoded for safe transport
  and storage.
- DotEnv stores the secrets for each level as a single **encrypted blob**. The server never parses
  your `KEY=value` content; it stores and returns opaque encrypted data.

Each **project** has its own encryption key, and you choose how that key is managed.

## Two key-custody models

### Server-managed keys

DotEnv stores the (encrypted) data key for the project. This is the simplest model: you don't
have to remember a passphrase, and the server can decrypt on your behalf when that's useful —
for example, server-side secret retrieval through the API. The trade-off is that DotEnv holds the
key material and is therefore technically able to decrypt your secrets.

### Client-managed keys (zero-knowledge)

**You** hold a passphrase; DotEnv never stores the key. From your passphrase, your client (the
browser, the CLI) derives the data key with PBKDF2 and uses it to encrypt and decrypt locally.

What the server stores instead is a non-secret **PBKDF2 proof** (along with the salt and
iteration parameters). The proof lets the server verify that the key you present is the correct
one — without being able to reconstruct the key from it. This is what makes the model
**zero-knowledge**: the server can confirm you have the right key, but it can't derive the key or
read your values.

The consequence is deliberate: **if you lose the passphrase, the secrets cannot be recovered** —
not by you, and not by DotEnv. Maximum privacy, with the responsibility resting on you.

## What DotEnv can and cannot see

| | Server-managed | Client-managed (zero-knowledge) |
| --- | --- | --- |
| Plaintext secret values | Can decrypt (holds the key) | **Never** — only opaque ciphertext |
| The encryption key | Stored (encrypted) on the server | Not stored; only a PBKDF2 proof + salt |
| Can decrypt server-side | Yes | No |
| Recovery if key/passphrase lost | DotEnv holds the key | Not possible |

In **both** models, the server always sees:

- **Ciphertext blobs** — the encrypted secrets.
- **Metadata** — the hierarchy structure (organization, project, target, environment names and
  slugs), variable counts and versions, timestamps, and who did what.
- **Variable names are inside the encrypted blob**, not stored separately — the server does not
  index your `KEY` names.

What the server **never** sees in **client-managed** mode is the **plaintext values** or the
**key**.

## Where decryption and merging happen

A consistent principle ties this together: the **client** does the sensitive work.

- The CLI and the browser app derive keys, decrypt blobs, merge the
  [hierarchy levels](/documentation/core-concepts/secret-inheritance), and resolve `${VAR}`
  interpolation.
- The server stores, versions, and serves encrypted blobs and metadata.

This is why client-managed mode can be zero-knowledge while still offering organization, sharing,
and versioning.

## Practical guidance

- Choose **server-managed** when you want convenience and need server-side decryption (e.g.
  pulling secrets in automation without distributing a passphrase).
- Choose **client-managed** when you need the strongest privacy guarantee and can manage a
  passphrase safely (store it in a password manager — there is no reset).
- Either way, control who can access secrets with roles and scoped tokens; see
  [Access Control](/documentation/core-concepts/access-control).
