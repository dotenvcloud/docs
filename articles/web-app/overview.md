---
title: Web Dashboard Overview
slug: overview
order: 1
description: A tour of the DotEnv web dashboard, how to navigate it, and the Organization → Project → Target → Environment → Secret hierarchy.
tags: [web-app, dashboard, getting-started, hierarchy, navigation]
---

# Web Dashboard Overview

The DotEnv web dashboard is where you manage everything that the CLI and SDKs talk to:
organizations, projects, secrets, encryption keys, API keys, team members, and billing. This
page explains how the dashboard is laid out and the core hierarchy that all of DotEnv is built
around.

## Signing in

The dashboard lives behind authentication. To reach it you need to:

1. **Register** an account or **log in** at `/login`.
2. **Verify your email** — until you do, you are redirected to the verification notice and
   most of the app is unavailable.
3. Pass **two-factor authentication** if it is enabled on your account or **enforced** by your
   organization.

Once signed in, you land on the dashboard home at `/dashboard`.

## The hierarchy

Everything in DotEnv is organized into a strict, five-level hierarchy. Understanding it is the
single most important concept in the whole product:

```
Organization        (your account / company; identified by a unique ID)
  └── Project        (an application or service; identified by a slug)
       └── Target     (a deployment context, e.g. "backend", "web"; a slug)
            └── Environment   (e.g. production, staging, development, local; a slug)
                 └── Secret    (the actual key/value variables, stored encrypted)
```

- An **Organization** owns billing, members, teams, and API keys. You can belong to more than
  one and switch between them.
- A **Project** groups everything for one application.
- A **Target** lets you split a project into separate deployment contexts (for example a
  backend service and a frontend app, or different regions).
- An **Environment** is where secrets actually live. Each environment has a status
  (Production, Staging, Development, Local, or a custom value).
- **Secrets** are the environment variables themselves. They are stored encrypted and edited
  through the Multi-Level Secret Editor.

### Secrets merge most-specific-wins

Secrets can be set at the **project**, **target**, and **environment** levels. When a value is
pulled (by the CLI, an SDK, or shown merged in the editor), the levels are combined so that the
**most specific level wins**:

```
environment  ➜  overrides target  ➜  overrides project
```

This lets you define shared defaults once at the project level and override only what differs
in a specific target or environment. See
[Managing Secrets](managing-secrets) for the full details.

## Navigating the dashboard

The main areas you will use are:

| Area | Where | What it's for |
| --- | --- | --- |
| **Dashboard home** | `/dashboard` | Your starting point and overview. |
| **Organizations** | `/dashboard/organizations` | Overview, members, teams, settings, billing. |
| **Projects** | `/dashboard/projects` | Create and manage projects, targets, environments. |
| **Secret Editor** | `/dashboard/secrets/editor` | Edit secrets across all three levels at once. |
| **Key Management** | `/dashboard/key-management` | Store, rotate, or remove encryption keys. |
| **API Keys** | `/dashboard/organizations/api-keys` | Create and manage scoped API tokens. |
| **Activity** | `/dashboard/activity` | Audit log of actions in the organization. |
| **Notifications** | `/dashboard/notifications` | In-app notifications and email preferences. |
| **Account Settings** | `/account/settings` | Your profile, security, preferences, notifications. |

### Switching organizations

If you belong to multiple organizations, you switch the **active** organization from the
organization switcher. Everything you see in the dashboard — projects, secrets, members,
billing — always belongs to the currently active organization.

## Where to go next

- New to DotEnv? Start with [Organizations](organizations), then
  [Projects, Targets & Environments](projects-targets-environments).
- Ready to add secrets? See [Managing Secrets](managing-secrets).
- Connecting the CLI or an SDK? Create an [API Key](api-keys).
- Concerned about encryption? Read [Encryption Keys](encryption-keys).
