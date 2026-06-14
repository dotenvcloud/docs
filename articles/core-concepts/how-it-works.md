---
title: How DotEnv Works
slug: how-it-works
order: 1
description: The big picture — store your secrets centrally, keep them encrypted, and pull or inject them wherever your code runs, through the web dashboard, the CLI, or the HTTP API.
tags: [core-concepts, overview, architecture, encryption, cli, api]
---

# How DotEnv Works

DotEnv is a place to store the configuration your applications need to run — database URLs, API
keys, tokens, feature flags — **encrypted, organized, and shared** across your team and your
deployments. Instead of passing `.env` files around in chat or copying them between machines, you
keep one source of truth in DotEnv and pull the right values where you need them.

This page is the big picture. The rest of Core Concepts drills into each part.

## The core idea

1. **Store secrets centrally.** You organize secrets into a hierarchy — organization, project,
   target, environment — so every value has an obvious home. See
   [The DotEnv Hierarchy](/documentation/core-concepts/hierarchy).
2. **Everything is encrypted.** Secret values are encrypted with AES-256-GCM before they are
   stored. Depending on the key model you choose, DotEnv may never see your plaintext at all. See
   [Security Model](/documentation/core-concepts/security-model).
3. **Pull or inject where needed.** When an application, a CI job, or a developer needs the
   secrets, they retrieve them on demand. Values from the different levels are merged together —
   most-specific-wins — into the final set the app sees. See
   [Secret Inheritance](/documentation/core-concepts/secret-inheritance).

The result: one authoritative, encrypted store, and a consistent set of values everywhere your
code runs.

## Three ways to use DotEnv

You interact with the same secrets through whichever interface fits the task.

### Web dashboard

The browser app is where you set things up and do day-to-day editing: create the hierarchy,
invite people, manage roles, choose your encryption model, and edit secrets in the Multi-Level
Secret Editor (which shows you exactly how the levels merge). It is the most visual way to work
and the best place to administer your organization.

### CLI

The command-line tool is built for developers and automation. You authenticate once, then
`pull` secrets down into a local `.env` (or JSON, YAML, shell, or Dockerfile output) and `push`
files back up. It applies the same hierarchy merge as the dashboard, can resolve `${VAR}`
references between secrets, and handles client-side encryption when a project uses a passphrase
you hold. This is what you wire into local development and CI/CD.

### HTTP API

Everything the dashboard and CLI do sits on top of a versioned HTTP API
(`https://api.dotenv.cloud/api/v1`). You can call it directly from your own tooling using an
organization API token. The API is the integration surface for anything not covered by the
dashboard or CLI.

> **A note on SDKs:** language SDKs are not yet published. Until they are, integrate through the
> CLI or the HTTP API.

## Where the work happens

A key design choice runs through DotEnv: **the server stores encrypted blobs, and the client does
the sensitive interpretation.**

- The server keeps each level's secrets as an encrypted blob. It does not parse your `KEY=value`
  content, and in the client-managed key model it cannot read the plaintext at all.
- **Merging** levels into a final set and **resolving** `${VAR}` interpolation happen in the
  client — the CLI (or the dashboard in your browser) — at the moment you pull, not on the server.

This keeps your values private while still letting DotEnv organize, version, and share them.

## Putting it together

A typical flow looks like this:

1. In the dashboard, create a **project**, add a **target** (e.g. `backend`), and add an
   **environment** (e.g. `production`).
2. Choose an encryption model — let DotEnv manage the key, or hold a passphrase yourself for
   zero-knowledge storage.
3. Add secrets at the level where they belong: shared defaults on the project, context-specific
   values on the target, environment-specific values on the environment.
4. Create a scoped API token for your CI pipeline.
5. In CI, run the CLI to `pull` the merged, decrypted secrets into a `.env` for the build or
   deploy.

From here, follow [The DotEnv Hierarchy](/documentation/core-concepts/hierarchy) to learn how the
structure is organized.
