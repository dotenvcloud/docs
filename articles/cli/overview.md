---
title: CLI Overview
slug: overview
order: 1
description: What the DotEnv CLI is, how to install it, and the mental model for getting secrets into a .env file.
tags: [cli, overview, getting-started, secrets]
---

# CLI Overview

The DotEnv CLI (`dotenv`) is a single, cross-platform Go binary for managing environment
variables securely. It talks to the DotEnv API at `https://api.dotenv.cloud` and supports both
server-managed and client-managed encryption, so secret values can be decrypted on your machine
and never leave it in plaintext.

## What it does

- Pull secrets into a local `.env` file (or stdout) and export them in several formats.
- Push local `.env` files back up to the cloud, encrypted.
- Browse your projects, targets, and environments (`list`, `tree`, `explore`, `path`).
- Manage projects, targets, and environments (`project`, `target`, `environment`).
- Manage authentication, accounts, and organizations (`login`, `account`, `org`, `status`).
- Manage API keys and encryption keys (`apikeys`, `key`).
- View and restore previous secret versions (`secret versions`, `secret restore`, ...).

## Install

The quickest install on macOS and Linux:

```bash
curl -sSL https://dotenv.cloud/install.sh | bash
```

See [Installation](./installation.md) for every method (Homebrew, `go install`, Windows,
nightly builds) and for updating.

## The mental model: pull → .env

The DotEnv hierarchy has three levels, written as a slash-separated path:

```
project/target/environment
```

A secret stored at a level is inherited by deeper levels, so a value set on the project applies
to every target and environment underneath it unless overridden.

The core workflow is: **authenticate once, then pull secrets into a `.env` file.**

```bash
# 1. Authenticate (opens your browser)
dotenv login

# 2. Pull the merged secrets for an environment into .env
dotenv pull myapp/production/api --output .env
```

`pull` merges all levels in the path (project, then target, then environment) by default, with
deeper levels overriding shallower ones. Use [`export`](./pulling-and-pushing.md) when you want
a specific output format (shell, JSON, YAML, Dockerfile).

> The CLI does **not** run or wrap your process. There is no `dotenv run`. The CLI's job is to
> produce a `.env` file (or formatted output) that your app, container, or CI loads itself.

## Command map

| Area | Commands |
|------|----------|
| Setup | `init`, `login`, `logout`, `status`, `version`, `update`, `completion` |
| Secrets I/O | `pull`, `push`, `export`, `secret delete` |
| Browse | `list`, `tree`, `explore`, `path` |
| Resources | `project`, `target`, `environment` |
| Accounts & orgs | `account`, `org`, `refresh` |
| API keys | `apikeys` |
| Encryption keys | `key` |
| Secret versions | `secret versions`, `secret show`, `secret diff`, `secret restore`, `secret purge` |

See the [full command reference](./commands.md) for every flag and an example of each command.

## Where to go next

- [Installation](./installation.md) — install, update, verify.
- [Authentication](./authentication.md) — OAuth, API keys, accounts, organizations, CI.
- [Pulling and pushing](./pulling-and-pushing.md) — paths, formats, hierarchy merge.
- [Configuration](./configuration.md) — config directory and environment variables.
- [Client-side encryption](./client-side-encryption.md) — server- vs client-managed keys.
- [Secret versioning](./secret-versioning.md) — history, diff, restore.
- [Troubleshooting](./troubleshooting.md) — common failures and fixes.
