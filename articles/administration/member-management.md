---
title: Member Management
slug: member-management
order: 1
description: Invite and remove organization members, accept invitations, and switch between organizations.
tags: [administration, members, invitations, organizations]
---

# Member Management

People in your DotEnv organization are **members**, each with a [role](/documentation/administration/roles-and-permissions)
that controls what they can do. Members are managed from the organization's **Members** tab in the
dashboard (`/dashboard/members`).

## Inviting a member

1. Open the **Members** tab and start a new invitation.
2. Enter the person's **email address** and choose the **role** to assign.
3. Send the invitation.

The invitee receives an email containing a **signed link**. Pending invitations are listed alongside
members so you can see who hasn't joined yet.

> Adding members may count against your plan's **member** limit, which is a hard cap. If you've
> reached it, you'll be prompted to upgrade or remove someone first. See
> [Plan Limits & Usage](/documentation/administration/plan-limits-and-usage).

## Accepting an invitation

The invitee clicks the **signed link** in the invitation email. Following it joins them to the
organization with the role you assigned. The link is the only thing required — no separate access
request or approval step.

If a link has expired or stopped working, send a fresh invitation from the **Members** tab.

## Removing a member

From the **Members** tab, remove the member you no longer want in the organization. They immediately
lose access to that organization's projects and resources.

Removing a member frees a slot against your plan's member limit. To revoke automated access instead
of a person, delete or re-scope the relevant
[API key](/documentation/administration/roles-and-permissions) rather than removing a member.

## Switching organizations

A single account can belong to more than one organization. The organization currently in context
determines what you see and act on.

- **In the dashboard**, switch the active organization from the organization switcher.
- **In the CLI**, organization switching is available when you authenticate with **OAuth**
  (`dotenv login`):

  ```bash
  dotenv org list          # see organizations you can access
  dotenv org use <org>     # switch the active organization
  dotenv status            # confirm which organization is active
  ```

  An **API key** is tied to a single organization, so it will only ever show that one — use OAuth if
  you need to list or switch between several.

## Two-factor authentication enforcement

An organization can **enforce two-factor authentication (2FA)** for all its members. When enforcement
is turned on, members who don't yet have 2FA enabled get a **grace period** (default **7 days**)
during which they're warned to set it up. After the grace period expires, those members are required
to enable 2FA before they can continue using the organization.

If you administer an organization, enable enforcement well ahead of any deadline so members have time
to act within the grace window. See [Account & Security](/documentation/web-app/account-and-security)
for enabling 2FA on your own account.
