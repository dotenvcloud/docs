---
title: Team Setup
slug: team-setup
order: 6
description: Invite members to your organization, understand the available roles, switch between organizations, and create read-only API keys for CI.
tags: [getting-started, team, members, invitations, roles, organizations, api-keys]
---

# Team Setup

DotEnv is built for teams. Your **organization** sits at the top of the hierarchy and owns your
projects, members, teams, and API keys. This guide covers inviting people, choosing roles,
switching between organizations, and creating keys for automation.

## Invite members

People in your organization are **members**, each with a **role** that controls what they can do.

1. Open your organization and choose the **Members** tab.
2. Start a new invitation, enter the person's **email address**, and choose the **role** to
   assign.
3. Send it.

The invitee receives an email with a signed link. When they accept it, they join your
organization with the role you picked. Pending invitations are listed alongside members so you
can see who hasn't accepted yet.

> Adding members may count against your plan's member limit. If you hit it, you'll be prompted to
> upgrade. See [Billing & Plans](/documentation/web-app/billing-and-plans).

## Roles at a glance

Roles bundle a set of permissions. DotEnv ships with these immutable **system roles**:

| Role | What it's for |
| --- | --- |
| **Owner** | Full control over the organization. |
| **Administrator** | Manage projects and teams, but not billing. |
| **Member** | Basic access to work with projects. |
| **Developer** | Enhanced access for development teams. |
| **Auditor** | Read-only access for compliance and auditing. |
| **Billing Manager** | Manage billing and subscriptions only. |
| **API User** | Limited access intended for integrations. |

Follow **least privilege** — give each person the narrowest role that lets them do their job. For
example, give finance staff **Billing Manager** rather than **Administrator**, and give auditors
the read-only **Auditor** role.

On higher plans you can also define **custom roles** with your own combination of permissions. To
group members for larger organizations, you can organize them into **teams**. See
[Teams & Members](/documentation/web-app/teams-and-members) for both.

## Switch between organizations

You can belong to more than one organization (your own and a client's, say). The dashboard always
operates on **one active organization** at a time — the projects, secrets, members, and billing
you see all belong to it.

Use the organization switcher in the dashboard to change it; the whole dashboard then reflects the
newly selected organization. The CLI mirrors this:

```bash
# List organizations for the current account
dotenv org list

# Switch the active organization (by slug or ULID)
dotenv org use acme-corp

# Show the current organization
dotenv org show
```

See [Organizations](/documentation/web-app/organizations) for organization settings, and
[CLI Authentication](/documentation/cli/authentication) for managing accounts and organizations
from the command line.

## API keys for CI and automation

People log in with `dotenv login` (browser-based). For **CI pipelines and automation** — where
there's no browser and no person — create an **API key** instead. Keep these keys **read-only**
so a pipeline can pull secrets but never change them.

1. In the dashboard, go to **API Keys** and click **Create**.
2. Give the key a recognizable **name**.
3. Choose **permissions** — for a CI pull job, grant only read access to secrets. You can also
   **scope** the key to specific projects or teams.
4. Choose an **expiration** and create the key.

The token is shown **only once** — copy it immediately and store it in your CI's secret store. If
you lose it, you must rotate or recreate the key. See [API Keys](/documentation/web-app/api-keys)
for the full permission list and best practices.

Use the key in CI by passing it through the environment:

```bash
export DOTENV_API_KEY="your-read-only-key"
dotenv pull myapp/production/api --output .env --quiet
```

When `DOTENV_API_KEY` is set, the CLI uses it directly and skips browser login entirely.

## Next steps

- [Teams & Members](/documentation/web-app/teams-and-members) — roles, custom roles, and teams in
  full.
- [API Keys](/documentation/web-app/api-keys) — scoped, expiring, read-only keys.
- [CLI Authentication](/documentation/cli/authentication) — accounts, organizations, and CI auth.
