---
title: Projects, Targets & Environments
slug: projects-targets-environments
order: 3
description: Create and organize the DotEnv hierarchy — projects, their targets, and the environments where secrets live.
tags: [web-app, projects, targets, environments, hierarchy]
---

# Projects, Targets & Environments

Below the organization, DotEnv organizes your secrets into three nested levels:
**Project → Target → Environment**. You build this structure from the **Projects** area of the
dashboard (`/dashboard/projects`).

## Projects

A **project** represents one application or service. Each project has a name and a **slug** (a
short, URL-safe identifier used by the CLI and SDKs).

### Create a project

1. Go to **Projects** (`/dashboard/projects`).
2. Click **Create project** (`/dashboard/projects/create`).
3. Enter a name (the slug is derived from it) and save.

> Project creation is subject to your plan's project limit. If you have reached the limit you
> will be prompted to upgrade. See [Billing & Plans](billing-and-plans).

### Manage a project

Open a project (`/dashboard/projects/{project}`) to see its targets and settings. From there you
can:

- **Edit** the project (`/edit`) — including its name and the **Secret versioning** toggle.
  See [Secret Versioning](secret-versioning).
- **Delete** the project (`/delete`) — this removes its targets, environments, and secrets.

## Targets

A **target** is a deployment context inside a project. Use targets to separate, for example, a
**backend** service from a **web** front end, or to split by region. Each target has a name and
a slug.

### Create a target

1. Open a project.
2. Choose **Create target** (`/dashboard/projects/{project}/targets/create`).
3. Enter a name and save.

You can **edit** or **delete** a target from its page
(`/dashboard/projects/{project}/targets/{target}`).

## Environments

An **environment** is where secrets actually live. Typical environments are **production**,
**staging**, **development**, and **local**, but you are free to create your own.

### Environment status

Every environment has a **status**, which controls how it is labeled and color-coded:

- **Production** (highlighted as the highest-risk context)
- **Staging**
- **Development**
- **Local**
- **Custom** — pick "custom" and type your own status; it is automatically slugified
  (lowercased, spaces become hyphens).

There is also an **Is production** flag you can set when creating or editing an environment.
Marking an environment as production signals that it holds live, sensitive values and should be
treated with extra care.

### Create an environment

1. Open a target.
2. Choose **Create environment**
   (`/dashboard/projects/{project}/targets/{target}/environments/create`).
3. Enter a name, pick a status (or a custom one), optionally add a description, set the
   **Is production** flag if appropriate, and save.

You can **edit** or **delete** an environment from its page.

## How the pieces fit together

Once your hierarchy exists, you add and edit the actual variables in the
[Multi-Level Secret Editor](managing-secrets), which lets you pick a
project → target → environment and edit secrets at all three levels with overrides cascading
down. The structure you build here is also exactly what you reference from the CLI and SDKs
(by organization, project slug, target slug, and environment slug).

## Tips for organizing

- Keep **project-level** secrets for values shared everywhere (e.g. a third-party API base URL).
- Use **target-level** secrets for values that differ between deployment contexts.
- Reserve **environment-level** secrets for values that change per environment (e.g. the
  production database URL vs. the local one).
- This mirrors the most-specific-wins merge, so you only override what truly differs.
