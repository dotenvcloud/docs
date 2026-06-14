---
title: Roles & Permissions
slug: roles-and-permissions
order: 2
description: The DotEnv system roles and what each can do, custom roles on Team+ plans, and how permissions map to API token scopes.
tags: [administration, roles, permissions, api-keys, scopes, security]
---

# Roles & Permissions

Every member has a **role** that bundles a set of permissions. DotEnv ships with **system roles**
(which are immutable) and, on higher plans, lets you define your own **custom roles**. Assign roles
following least privilege — give people the narrowest role that lets them do their job.

## System roles

These roles exist in every organization and cannot be edited or deleted:

| Role | What it's for |
| --- | --- |
| **Owner** | Full control over the organization, including billing and deletion. |
| **Administrator** | Manage projects, teams, and members — but not billing. |
| **Member** | Basic access to work with projects. |
| **Developer** | Enhanced access intended for development teams. |
| **Auditor** | Read-only access for compliance and auditing. |
| **Billing Manager** | Manage billing and subscriptions only, without broader administrative access. |
| **API User** | Limited access intended for integrations. |

Pick the role that matches what each person should be able to do. For example, give finance staff
**Billing Manager** rather than **Administrator**, and give reviewers the read-only **Auditor** role.

## Custom roles (Team plans and above)

On **Team** plans and above you can define **custom roles** with your own combination of permissions,
in addition to the system roles. This lets you tailor access precisely to how your organization works
— for instance, a role that can read and write secrets in one project but nothing else.

If custom roles aren't available to you, your current plan doesn't include them; see
[Plan Limits & Usage](/documentation/administration/plan-limits-and-usage) to upgrade.

## Permissions

Permissions are fine-grained and grouped by resource. Roles (and API keys) are built from these:

| Group | Permissions |
| --- | --- |
| **Secrets** | read, write, decrypt (server-side only), history, restore, purge; plus encryption-key retrieve and rotate |
| **Projects** | create, read, update, delete |
| **Targets** | create, read, update, delete |
| **Environments** | create, read, update, delete |
| **Teams** | create, read, update, delete |
| **Organizations** | create, read, update, delete |
| **API Tokens** | create, read, update, delete |
| **Members** | create, read, update, delete |
| **Billing** | read, write |

## Mapping to API token scopes

API keys carry the **same permissions as a list of scopes**, expressed as `resource:action`
strings — for example `secret:read`, `secret:write`, `project:create`, `billing:read`. A key only
allows what its scopes allow.

Common shapes:

- **Full access** — the wildcard scope `*`.
- **Read-only** — typically `secret:read`, `project:read`, `target:read`, `environment:read`,
  `organization:read`.

### Scoped permissions

Most read/update/delete permissions can be **narrowed to a part of the hierarchy**, so a key only
touches the resources you intend. The level of scoping depends on the permission:

| Permission family | Can be scoped to |
| --- | --- |
| Secret and encryption-key permissions | any level (project, target, or environment) |
| Project permissions (read/update/delete) | a specific project |
| Target permissions (read/update/delete) | a specific project + target |
| Environment permissions (read/update/delete) | a specific project + target + environment |
| Team, organization, and API-token permissions (read/update/delete) | the organization/team |

`create` permissions can't be scoped to an existing resource (there's nothing yet to scope to). A
request that falls outside a key's scope is treated as **not found** or **forbidden** — see the
[Error Reference](/documentation/troubleshooting/error-reference).

Manage API keys and their scopes from the dashboard; see [API Keys](/documentation/web-app/api-keys).
