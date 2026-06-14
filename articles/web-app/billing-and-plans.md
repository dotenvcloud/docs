---
title: Billing & Plans
slug: billing-and-plans
order: 11
description: Choose a plan, manage your subscription, track usage against limits, view invoices, and understand overage warnings and account lock behavior.
tags: [web-app, billing, plans, subscription, usage, invoices, limits, overage]
---

# Billing & Plans

Billing is managed per organization, from the **Billing** tab on the organization page
(`/dashboard/billing`) and the related billing pages. This guide describes how plans, usage, and
invoices work in the dashboard.

## Plans

Open **Plans** (`/dashboard/organizations/billing/plans`) to compare the available plans and what
each includes. Plans differ by:

- **Limits** — such as the number of **projects**, **members**, **API calls**, and **storage**.
- **Features** — higher tiers unlock capabilities like **custom roles**, longer audit-log
  retention, and other team/enterprise features.

Choosing or changing a plan updates what your organization is allowed to do.

## Subscription

From the **Billing** tab you manage your subscription — subscribing to a plan and changing plans.
Billing is **provider-agnostic**: DotEnv routes payment through a billing abstraction, so the
exact payment experience may vary. In addition to standard payment, **wire transfer** is
supported:

- **Wire transfer instructions** (`/organizations/{organization}/billing/wire-transfer`) explain
  how to pay by bank transfer.

## Usage and limits

Track how much of your plan you're using on the usage page
(`/organizations/{organization}/billing/usage`). Your plan defines hard caps and metered limits,
and the dashboard shows your current consumption against them — for example projects used vs.
allowed, members, API calls, and storage.

### Overage warnings and lock behavior

As you approach or exceed your plan limits, DotEnv handles it gracefully rather than abruptly:

- **Soft overage warnings** — you'll be warned (in the dashboard and, depending on your
  preferences, by email) as you near or pass a limit, so you can upgrade or reduce usage.
- **Hard limits** — some limits are firm caps. For example, when the **member** cap is reached,
  you can't add more members until you upgrade or remove some. Likewise, you'll be prompted to
  upgrade if you try to create a project beyond your project limit.
- **Hard lock + grace period** — if the account falls out of good standing (for example a billing
  problem or sustained overage), the organization can enter a restricted state. You're given a
  **grace period** to resolve it. If it isn't resolved, the organization is locked and you're
  routed to a status page (`/organization/locked`, `/organization/suspended`, or
  `/organization/cancelled`) explaining what to do.

Keep billing notifications enabled so you see these warnings early. See
[Notifications & Activity](notifications-and-activity).

## Invoices and transactions

- **Invoices** (`/dashboard/organizations/billing/invoices`) lists your invoices. You can
  **download** any invoice for your records.
- **Transaction history** (`/organizations/{organization}/billing/transactions`) shows your
  billing transactions.

## Who can manage billing

Billing actions require appropriate permissions. Assign the **Billing Manager** role to people who
should handle payments and subscriptions without giving them broader administrative access. See
[Teams & Members](teams-and-members).
