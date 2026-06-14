---
title: Organizations
slug: organizations
order: 2
description: Create and switch organizations, configure organization settings, enforce two-factor authentication, and use the danger zone.
tags: [web-app, organizations, settings, 2fa, danger-zone]
---

# Organizations

An **organization** is the top of the DotEnv hierarchy. It owns your projects, members, teams,
API keys, and billing. Every user belongs to at least one organization and can belong to many.

## Creating an organization

1. Go to **Organizations** (`/dashboard/organizations`).
2. Click **Create organization** (`/dashboard/organizations/create`).
3. Give it a name and complete the form.

The new organization becomes available in your organization switcher.

## Switching the active organization

The dashboard always operates on **one active organization** at a time — the projects,
secrets, members and billing you see all belong to it.

To change it, use the organization switcher in the dashboard. (Internally this hits
`/switch-organization/{id}`.) After switching, the whole dashboard reflects the newly selected
organization.

## The organization page and its tabs

Open an organization to see its detail page, which is split into tabs:

| Tab | What it shows |
| --- | --- |
| **Overview** | A summary of the organization. |
| **Members** | People in the organization, their roles, and pending invitations. |
| **Teams** | Teams that group members together. |
| **Settings** | Organization name, 2FA enforcement, and the danger zone. |
| **Billing** | Plan, subscription, usage, and invoices. |

You can jump straight to a tab at `/dashboard/{tab}` (for example `/dashboard/settings`).

For members, roles, and teams, see [Teams & Members](teams-and-members). For billing, see
[Billing & Plans](billing-and-plans).

## Settings

The **Settings** tab is where organization-wide configuration lives.

### Enforcing two-factor authentication

You can require every member of the organization to have two-factor authentication (2FA)
enabled:

1. Open **Settings**.
2. Turn on **Enforce 2FA**.
3. Set a **grace period** (in days). This gives members who don't yet have 2FA a window to set
   it up before they are blocked from the dashboard.

While enforcement is active:

- Members without 2FA are guided to set it up.
- Account settings remain reachable so that members can actually configure 2FA, but the rest of
  the dashboard is gated until they comply.
- After the grace period ends, members who still have not enabled 2FA cannot access protected
  organization resources.

The grace period is configurable (commonly defaulting to 7 days). Members set up their own 2FA
methods from their [Account & Security](account-and-security) settings.

## Danger zone

The danger zone (in **Settings**) holds destructive, organization-level actions. Treat anything
here as irreversible and confirm carefully before proceeding.

> **Tip:** Deleting an organization removes its projects, targets, environments, and secrets.
> If you only need to step away temporarily, consider removing yourself as a member or rotating
> credentials instead.

## Organization status

An organization can be placed into a restricted state — for example **suspended**, **locked**,
or **cancelled** — typically tied to billing. When that happens you are routed to a status page
(`/organization/suspended`, `/organization/locked`, or `/organization/cancelled`) that explains
what is wrong and how to resolve it. See [Billing & Plans](billing-and-plans) for how plan
limits and overages affect status.
