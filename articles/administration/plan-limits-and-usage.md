---
title: Plan Limits & Usage
slug: plan-limits-and-usage
order: 3
description: How DotEnv plan limits work, tracking usage in the dashboard, soft vs hard overage, the grace period, and recovering a locked organization.
tags: [administration, plans, limits, usage, overage, billing]
---

# Plan Limits & Usage

Each plan defines how much of DotEnv your organization can use. Limits, usage, and overage behavior
are all managed per organization from the dashboard.

## What's limited

Your plan sets caps on:

- **Projects** — how many projects the organization can have.
- **Members** — how many people can belong to the organization.
- **Targets per project** and **environments per target** — the depth of your hierarchy.
- **API calls** — metered per month.
- **Storage** — metered (this is what bounds how many secrets you can store; there is no separate
  secret-count limit).

Higher tiers also unlock **features** beyond raw limits, such as **custom roles** and longer
audit-log retention. A limit of **0 means unlimited** for that resource on your plan.

## Tracking usage

The dashboard shows your current consumption against your plan's limits — projects used vs. allowed,
members, API calls, and storage. Check it regularly, especially before adding projects or inviting
people, so you're not surprised by a cap. Changes across the organization are recorded in the
**activity / audit log**, which is useful for understanding what consumed a limit and when.

## Soft vs. hard overage

DotEnv handles approaching and exceeding limits gracefully rather than abruptly:

- **Soft overage (warnings)** — as you near or pass certain limits, you're warned in the dashboard
  (and, depending on your notification preferences, by email) so you can upgrade or reduce usage.
  Keep billing notifications enabled to see these early.
- **Hard limits** — some caps are firm. For example, when the **member** cap is reached you can't add
  more members until you upgrade or remove some; likewise you'll be prompted to upgrade if you try to
  create a project beyond your project limit.

## Grace period and account lock

If the organization falls out of good standing — for example a billing problem or sustained overage —
it can enter a restricted state with a **grace period** to resolve the issue.

- During the grace period, you're warned and given time to fix the underlying problem (upgrade,
  settle billing, or reduce usage).
- If it isn't resolved in time, the organization is **locked** and members are routed to a status
  page explaining what to do. An organization's status can be **Active**, **Suspended**, **Locked**,
  or **Cancelled** — only an **Active** organization can access its resources.

## Recovering a locked organization

When an organization is locked (or suspended), resolve the cause to restore access:

1. Open the organization and follow the prompts on the lock/status page.
2. Fix the underlying issue — most often this means upgrading the plan, completing payment, or
   bringing usage back within limits.
3. Once the organization returns to **Active** status, access to projects and resources is restored.

Billing actions require the right permissions, so make sure someone with the **Owner** or
**Billing Manager** role handles the recovery. See
[Roles & Permissions](/documentation/administration/roles-and-permissions).
