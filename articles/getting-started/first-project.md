---
title: Your First Project
slug: first-project
order: 4
description: Understand the DotEnv hierarchy and create your first project, target, and environment in the dashboard.
tags: [getting-started, projects, targets, environments, hierarchy, dashboard]
---

# Your First Project

DotEnv organizes everything into a strict, five-level hierarchy. Understanding it is the single
most important concept in the product — once it clicks, the dashboard, the CLI, and the API all
make sense.

```
Organization        your account or company (identified by a unique ULID)
  └── Project        an application or service (identified by a slug)
       └── Target     a deployment context, e.g. "backend", "web" (a slug)
            └── Environment   e.g. production, staging, development, local (a slug)
                 └── Secret    the actual key/value variables, stored encrypted
```

- An **Organization** owns billing, members, teams, and API keys. You can belong to more than
  one and switch between them.
- A **Project** groups everything for one application.
- A **Target** splits a project into separate deployment contexts (for example a backend service
  and a frontend app, or different regions).
- An **Environment** is where secrets actually live — production, staging, development, local, or
  your own.
- A **Secret** is a single `KEY=value` pair, encrypted at rest.

You build this structure in the dashboard, then reference it from the CLI and SDKs by
**project slug / target slug / environment slug** — for example `myapp/production/api`.

## Create a project

A project represents one application or service.

1. Go to **Projects** in the dashboard.
2. Click **Create project**.
3. Enter a **name**. The **slug** is derived from it automatically (a short, URL-safe identifier
   the CLI and SDKs use). Save.

> Project creation is subject to your plan's project limit. If you've reached it, you'll be
> prompted to upgrade. See [Billing & Plans](/documentation/web-app/billing-and-plans).

## Create a target

A target is a deployment context inside a project. Use targets to separate, for example, a
**backend** service from a **web** front end, or to split by region.

1. Open your project.
2. Choose **Create target**.
3. Enter a **name** and save. As with projects, the slug is derived from the name.

If you only have one deployment context, a single target (for example `production`) is perfectly
fine to start.

## Create an environment

An environment is where secrets actually live. Typical environments are **production**,
**staging**, **development**, and **local**, but you can create your own.

1. Open your target.
2. Choose **Create environment**.
3. Enter a **name**, pick a **status** (production, staging, development, local, or a custom one),
   optionally add a description, set the **Is production** flag if it holds live values, and save.

Marking an environment as production signals that it holds sensitive, live values and should be
treated with extra care.

## Add your first secrets

With the hierarchy in place, open the secret editor, pick your
**project → target → environment**, and add variables. You can set secrets at all three levels;
overrides cascade downward (most specific wins). See
[Managing Secrets](/documentation/web-app/managing-secrets) for the editor in depth and
[Environments & cascading](/documentation/getting-started/environments) for how the merge works.

## How it maps to the CLI

The structure you build here is exactly what you reference from the command line:

```bash
# project slug / target slug / environment slug
dotenv pull myapp/production/api --output .env
```

A shorter path acts on a shallower level:

```
myapp                  # project level
myapp/production       # target level
myapp/production/api   # environment level
```

## Tips for organizing

- Keep **project-level** secrets for values shared everywhere (e.g. a third-party API base URL).
- Use **target-level** secrets for values that differ between deployment contexts.
- Reserve **environment-level** secrets for values that change per environment (e.g. the
  production database URL vs. the local one).

## Next steps

- [Environments & cascading](/documentation/getting-started/environments) — how the merge works
  across levels.
- [Projects, Targets & Environments](/documentation/web-app/projects-targets-environments) — the
  full dashboard reference.
- [Quick Start](/documentation/getting-started/quick-start) — pull your new secrets with the CLI.
