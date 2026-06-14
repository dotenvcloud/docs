---
title: Teams & Members
slug: teams-and-members
order: 8
description: Invite members, assign roles (including custom roles on higher plans), and group people into teams.
tags: [web-app, members, invitations, roles, teams, permissions]
---

# Teams & Members

People in your organization are **members**, each with a **role** that controls what they can do.
You can also group members into **teams**. Both are managed from the organization page.

## Members

Open your organization and choose the **Members** tab
(`/dashboard/members` or `/dashboard/organizations/current`).

### Inviting a member

1. On the **Members** tab, start a new invitation.
2. Enter the person's email address and choose the **role** to assign.
3. Send the invitation.

The invitee receives an email with a **signed link**. When they accept it
(`/invitation/{invitation}`), they join the organization with the role you chose. Pending
invitations are listed alongside members so you can track who hasn't accepted yet.

> Adding members may count against your plan's member limit. If you hit the limit you'll be
> prompted to upgrade. See [Billing & Plans](billing-and-plans).

## Roles

Roles bundle a set of permissions. DotEnv ships with **system roles** (which are immutable):

| Role | What it's for |
| --- | --- |
| **Owner** | Full control over the organization. |
| **Administrator** | Manage projects and teams, but not billing. |
| **Member** | Basic access to work with projects. |
| **Developer** | Enhanced access for development teams. |
| **Auditor** | Read-only access for compliance and auditing. |
| **Billing Manager** | Manage billing and subscriptions only. |
| **API User** | Limited access intended for integrations. |

Assign whichever role matches what each person should be able to do. Follow least privilege —
for example, give finance staff **Billing Manager** rather than **Administrator**, and give
auditors the read-only **Auditor** role.

### Custom roles

On **Team** plans and above, you can define **custom roles** with your own combination of
permissions, in addition to the system roles. This lets you tailor access precisely to how your
organization works. If custom roles aren't available, your current plan doesn't include them —
see [Billing & Plans](billing-and-plans) to upgrade.

## Teams

A **team** groups members together, which makes it easier to organize larger organizations and to
reason about access. Manage teams from the **Teams** tab (`/dashboard/teams`).

### Creating a team

1. Open the **Teams** tab.
2. Choose **Create team** (`/dashboard/organizations/teams/create`).
3. Name the team and save.

### Managing a team

Open a team to see its detail page, which has **Overview**, **Members**, and **Settings** tabs.
From there you add or remove members and adjust the team's settings.

Teams also matter for **API keys**: a key's permissions can be **scoped to a specific team**, so
automation only touches the projects and resources that belong to that team. See
[API Keys](api-keys).
