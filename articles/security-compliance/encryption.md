---
title: Encryption Model
slug: encryption
order: 1
description: How DotEnv encrypts your secrets end to end with AES-256-GCM and PBKDF2, what is and isn't encrypted, and exactly what DotEnv can see.
tags: [security, encryption, aes-256-gcm, pbkdf2, zero-knowledge, compliance]
---

# Encryption Model

Every secret value you store in DotEnv is encrypted with **AES-256-GCM** before it lands in the
database. The same cryptographic contract is implemented byte-for-byte across the web app, the
CLI, and every SDK (Go, JavaScript, PHP), so a value encrypted by one can always be decrypted by
another. This page explains the model end to end: how a secret is encrypted, how the key is
derived, and — most importantly — **what DotEnv can and cannot see**.

## The cipher

DotEnv uses authenticated encryption so that tampering is detectable, not just confidentiality:

| Property        | Value                                            |
| --------------- | ------------------------------------------------ |
| Algorithm       | AES-256-GCM (authenticated encryption)           |
| Key size        | 32 bytes (256-bit)                               |
| IV / nonce      | 12 bytes, randomly generated per encryption      |
| Auth tag        | 16 bytes (GCM tag)                               |
| Wire format     | `base64(IV ‖ ciphertext ‖ tag)`                  |

Each encryption operation generates a **fresh random 12-byte IV**, so encrypting the same value
twice produces different ciphertext. The 16-byte GCM authentication tag is appended to the
ciphertext and verified on decryption — if anything in the stored blob is altered, decryption
fails rather than returning corrupted plaintext.

## How the data key is derived

The 32-byte AES key used to encrypt your secrets — the **data key** — is not used raw. It is
derived from your project's key material using **PBKDF2-HMAC-SHA256**:

```text
data_key = PBKDF2-HMAC-SHA256(key, salt, iterations, dkLen = 32 bytes)
```

- **`key`** is the project's key material (a strong random value for server-managed projects, or
  your passphrase for client-managed projects).
- **`salt`** is a random per-project salt stored alongside the key record.
- **`iterations`** is a platform-fixed, high iteration count shared across all
  implementations so the derivation is identical everywhere.

This derivation is **unified**: a given `(key, salt)` always produces the same AES data key
regardless of who holds the key. That single design choice is why switching a project between
server-managed and client-managed custody **never requires re-encrypting your secrets** — the
derived data key is the same on both sides of the transition.

## What gets encrypted

- **Encrypted:** every secret *value*. Secret values are encrypted with the derived data key and
  stored only as `base64(IV ‖ ciphertext ‖ tag)`. Historical versions are encrypted the same way.
- **Not encrypted (metadata):** the structural information DotEnv needs to operate — secret
  *keys/names*, project / target / environment names, version numbers and timestamps, and audit
  records. Treat secret **names** as non-sensitive; put sensitive material only in the **value**.

## What DotEnv can see

This depends entirely on how the project's key is managed:

- **Server-managed projects** — DotEnv stores the (encrypted) project key, so the platform *can*
  derive the data key and decrypt values server-side when an authorized request asks it to. This
  enables features like the web editor and server-side decrypt for API tokens that carry the
  `secret:decrypt` permission.
- **Client-managed projects** — DotEnv stores **only a PBKDF2 key-check proof** (a salt,
  iteration count, and a derived proof value) and **never the key itself**. The platform cannot
  derive the data key and therefore **cannot decrypt your secrets** — this is a
  **zero-knowledge** arrangement. Encryption and decryption happen entirely on your machine.

The key-check proof exists so that DotEnv can verify a *write* was performed with the correct key
(a mistyped or wrong key is rejected before it can silently corrupt your secrets), without ever
being able to recover the key or the plaintext from the proof.

Which mode to choose, how rotation works, and what you must keep safe are covered in
[Key Management](/documentation/security-compliance/key-management).

## Transport and at-rest

- **In transit:** all API, CLI, and SDK traffic uses TLS (HTTPS).
- **At rest:** secret values are stored only as AES-256-GCM ciphertext. Server-managed project
  keys are themselves encrypted at the application layer before storage; client-managed projects
  store no key at all.

## Consistency across clients

Because the cipher, IV/tag layout, wire format, and PBKDF2 derivation are defined by a single
cross-language contract, you can encrypt with the CLI and decrypt with an SDK (or vice versa)
without surprises. If you ever see a decryption failure across clients, it is almost always a
**key mismatch**, not a format mismatch — verify you're using the right key for the project.

## Related

- [Key Management](/documentation/security-compliance/key-management)
- [Encryption Keys (web)](/documentation/web-app/encryption-keys)
- [Client-Side Encryption (CLI)](/documentation/cli/client-side-encryption)
