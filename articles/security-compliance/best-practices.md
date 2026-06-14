---
title: Security Best Practices
slug: best-practices
order: 4
description: Practical hardening for DotEnv — least-privilege tokens, read-only CI access, two-factor authentication, key rotation cadence, client-side encryption, and secret hygiene.
tags: [security, best-practices, least-privilege, 2fa, rotation, ci, hygiene, compliance]
---

# Security Best Practices

DotEnv gives you strong encryption by default, but most real-world incidents come from how
secrets are *handled*, not how they're encrypted. This page is the short, practical checklist for
running DotEnv safely.

## Use least-privilege API tokens

API tokens carry **scoped permissions** — grant only what the integration actually needs:

- **Start from the read-only preset.** Most consumers (apps, deploy steps, dashboards) only need
  to *read* secrets, not write, decrypt server-side, or rotate keys. The built-in read-only preset
  grants exactly read access to secrets, projects, targets, environments, and the organization.
- **Scope by project and environment.** Permissions can be narrowed to a specific project,
  target, or environment, so a token for a staging service can't touch production.
- **Separate decrypt from read.** `secret:read` returns ciphertext; `secret:decrypt` is a distinct
  permission. Only grant `secret:decrypt` to tokens that genuinely need server-side decryption.
- **One token per consumer.** Give each service/CI job its own token so you can revoke it without
  disrupting everything else.

See [API Keys (web)](/documentation/web-app/api-keys) to create and scope tokens.

### Roles at a glance

For people (rather than tokens), assign the **least privileged role** that lets someone do their
job:

| Role            | Intended for                                  | Notes                                            |
| --------------- | --------------------------------------------- | ------------------------------------------------ |
| Owner           | Organization owner                            | Full control, including billing and deletion     |
| Administrator   | Team/project admins                           | Manage projects, members, and keys; **no billing** |
| Member          | Day-to-day contributors                       | Read/write secrets and history (no server decrypt) |
| Developer       | Development teams                             | Broader project/target access, can decrypt        |
| Auditor         | Compliance / reviewers                        | **Read-only**, incl. audit log; **no decrypt**     |
| Billing Manager | Finance                                       | Billing only                                      |
| API User        | Integrations                                  | Minimal read + decrypt for automation             |

## Lock down CI

CI is the most common place secrets leak. Treat it as untrusted:

- **Read-only by default.** Give CI a read-only, project/environment-scoped token. CI almost never
  needs to write secrets or rotate keys.
- **Inject keys via the runner's secret store**, never as plaintext in the workflow file.
- **Pass client keys by file, not value.** Use `--client-key=<file>` so the key never appears in
  the command line, shell history, or process listing. Avoid passing the literal key string.
- **Don't print secrets.** Never `echo` a secret or a key into build output — CI logs are often
  retained and broadly readable.

## Enforce two-factor authentication

- **Enable 2FA on every account.** DotEnv supports authenticator apps (**TOTP**), **email**, and
  **SMS** codes.
- **Enforce 2FA org-wide** for sensitive organizations. Enforcement includes a **grace period** so
  members can enrol before access is restricted; after it, members must keep at least one 2FA
  method enabled to retain access.
- Account settings stay reachable even under enforcement, so you can always finish enrolling.

See [Account & Security (web)](/documentation/web-app/account-and-security) and
[Organizations (web)](/documentation/web-app/organizations).

## Rotate keys on a cadence

- **Set a regular rotation schedule** for encryption keys, and rotate **immediately** if a key may
  have been exposed (leaked into chat, a log, a repo, or a departed team member's machine).
- **Rotated keys become immutable** so historical secret versions stay decryptable; use the
  **re-encrypt-history** flow when you need old versions readable under the new key. See
  [Key Management](/documentation/security-compliance/key-management).
- **Rotate API tokens too**, and revoke any token whose consumer is decommissioned.

## Prefer client-side encryption for high-sensitivity projects

For secrets that DotEnv should *never* be able to read, use **client-managed** keys. The platform
stores only a one-way key-check proof and cannot decrypt your values (zero-knowledge). You take on
key custody in exchange — store the passphrase in a password manager or vault, and ensure it's
recoverable by more than one trusted person. See
[Encryption Model](/documentation/security-compliance/encryption) and
[Client-Side Encryption (CLI)](/documentation/cli/client-side-encryption).

## Secret hygiene

- **`.gitignore` your `.env` files.** Pulled secrets land in local files — keep them out of git.
- **Never paste live secrets, keys, or passphrases into AI chats, tickets, or chat threads.** If
  it happens, treat the value as compromised and rotate it.
- **Don't reuse keys across projects or environments.** A compromise should be contained to one
  scope.
- **Treat secret *names* as non-sensitive** but secret *values* as always sensitive — only the
  value is encrypted.
- **Review the audit log periodically** to catch unexpected access or changes. See
  [Audit Logging](/documentation/security-compliance/audit-logging).

## Related

- [Encryption Model](/documentation/security-compliance/encryption)
- [Key Management](/documentation/security-compliance/key-management)
- [Audit Logging](/documentation/security-compliance/audit-logging)
- [API Keys (web)](/documentation/web-app/api-keys)
- [Account & Security (web)](/documentation/web-app/account-and-security)
