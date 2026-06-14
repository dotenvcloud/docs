---
title: Access Control
slug: access-control
order: 5
description: How DotEnv controls who can do what — organization roles for people in the web app, fine-grained scopes for API tokens, and the principle of least privilege.
tags: [core-concepts, access-control, roles, permissions, api-tokens, scopes, least-privilege]
---

# Access Control

DotEnv has two complementary ways to control access, matched to two kinds of actor:

- **People** working in the web app are governed by **organization roles**.
- **Programs** (the CLI, CI/CD, your own tooling) authenticate with **API tokens** that carry
  fine-grained **scopes**.

Both operate within an organization, and both are designed around **least privilege**: grant the
narrowest access that gets the job done.

## Roles (for people)

Every member of an organization has a **role** that bundles a set of permissions. DotEnv ships
with these immutable **system roles**:

| Role | What it's for |
| --- | --- |
| **Owner** | Full control over the organization. |
| **Administrator** | Manage projects and teams, but not billing. |
| **Member** | Basic access to work with projects. |
| **Developer** | Enhanced access for development teams. |
| **Auditor** | Read-only access for compliance and auditing. |
| **Billing Manager** | Manage billing and subscriptions only. |
| **API User** | Limited access intended for integrations. |

Assign whichever role matches what a person should be able to do — for example, give finance
staff **Billing Manager** rather than **Administrator**, and give compliance reviewers the
read-only **Auditor** role.

### Custom roles

On **Team** plans and above, you can define **custom roles** with your own combination of
permissions, alongside the system roles. This lets you tailor access precisely to how your
organization is structured.

## API tokens and scopes (for programs)

For programmatic access, you create **organization API tokens**. A token is presented as a Bearer
credential:

```
Authorization: Bearer <token>
```

Each token carries a fixed set of **scopes** (abilities) chosen when you create it. Scopes follow
the pattern `resource:action`, and every API endpoint requires a specific scope. A request that
lacks the required scope is rejected with `403 Forbidden`.

A few representative scopes:

| Scope | Grants |
| --- | --- |
| `secret:read` | Read encrypted secrets |
| `secret:write` | Create, update, or delete secrets |
| `secret:decrypt` | Decrypt secrets server-side (server-managed keys only) |
| `project:read` | Read projects |
| `key:retrieve` | Retrieve encryption keys |

A token granted `*` satisfies every scope check — use it sparingly.

### Read-only preset

A common starting point grants read access across the resource hierarchy:

```json
["secret:read", "project:read", "target:read", "environment:read", "organization:read"]
```

This is ideal for a CI job or service that only needs to **pull** secrets and never modify them.

### Scoping a token to part of the hierarchy

Beyond choosing *what* a token can do (its scopes), you can limit *where* it applies. A token can
be scoped to a specific **project**, **target**, or **environment**, so the credential you hand to
one pipeline can only ever touch that pipeline's slice of the hierarchy. A token scoped to
`acme-store/backend/production` with `secret:read` cannot read the `frontend` target or any other
project — even though the scope itself is otherwise broad.

## Least privilege in practice

- **Give people the smallest role that fits.** Auditors get **Auditor**; billing staff get
  **Billing Manager**; not everyone needs **Administrator**.
- **Create one token per integration**, not a shared master token, so you can revoke a single
  pipeline without disrupting others.
- **Prefer read-only tokens** for anything that only consumes secrets (most CI deploy steps just
  need `secret:read`).
- **Scope tokens to the hierarchy** so a compromised CI credential can't read unrelated projects.
- **Reserve `secret:decrypt`** for cases that genuinely need server-side decryption — and remember
  it only works for [server-managed keys](/documentation/core-concepts/security-model).
- **Rotate and revoke** tokens when they're no longer needed or may have been exposed.

## How it fits together

Roles and scopes work alongside the encryption model. Access control decides **who can request**
secrets; the [security model](/documentation/core-concepts/security-model) decides **whether they
can be decrypted** and by whom. With client-managed keys, even a fully authorized request returns
only ciphertext unless the requester also holds the passphrase — defense in depth across both
layers.

For the structure these permissions apply to, see
[The DotEnv Hierarchy](/documentation/core-concepts/hierarchy).
