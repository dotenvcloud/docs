---
title: Installation
slug: installation
order: 2
description: Create your DotEnv web account and install the DotEnv CLI on macOS, Linux, and Windows, then verify it works.
tags: [getting-started, installation, install, cli, account, signup]
---

# Installation

Getting set up with DotEnv takes two things:

1. A **web account** at [dotenv.cloud](https://dotenv.cloud), where you create projects and
   manage secrets in the dashboard.
2. The **DotEnv CLI**, a single binary that pulls those secrets onto your machine and into CI.

This page covers both. If you'd rather see the whole flow end to end in one go, jump to the
[Quick Start](/documentation/getting-started/quick-start).

## Create your web account

1. Open [dotenv.cloud](https://dotenv.cloud) and **register** an account (or log in if you
   already have one).
2. **Verify your email.** Until you do, most of the dashboard stays locked.
3. Sign in to land on your dashboard. A first organization is set up for you automatically — it
   sits at the top of the [hierarchy](/documentation/getting-started/first-project) and owns your
   projects, members, and billing.

That's all you need to start creating projects and adding secrets in the browser. For a tour of
the dashboard, see the [Web Dashboard Overview](/documentation/web-app/overview).

## Install the CLI

The DotEnv CLI ships as a single self-contained binary named `dotenv`. Pick the method that
fits your platform.

### macOS and Linux

**Install script (recommended).** Detects your OS and architecture, downloads the matching
binary, and installs it:

```bash
curl -sSL https://dotenv.cloud/install.sh | bash
```

**Homebrew (macOS).** If you use Homebrew, tap the DotEnv formula and install:

```bash
brew tap dotenvcloud/tap
brew install dotenv-cli
```

The Homebrew formula is named `dotenv-cli`; the installed binary is still `dotenv`.

### Windows

**Scoop:**

```powershell
scoop bucket add dotenv https://github.com/dotenvcloud/scoop-bucket
scoop install dotenv-cli
```

**Or the PowerShell install script:**

```powershell
irm https://dotenv.cloud/install.ps1 | iex
```

### From source (any platform with Go)

If you have a Go toolchain installed, you can build and install from source:

```bash
go install github.com/dotenvcloud/cli@latest
```

The binary is placed in your `$(go env GOPATH)/bin` directory — make sure that directory is on
your `PATH`.

> For the complete reference — including nightly builds, self-updating, and shell completion —
> see the [CLI Installation](/documentation/cli/installation) guide.

## Verify the install

Confirm the binary is on your `PATH` and check the version:

```bash
dotenv version
```

If you see version details, you're ready to go. If `dotenv` is **not found**, make sure the
install directory (`/usr/local/bin`, `~/.local/bin`, or your Go bin directory) is on your `PATH`,
then open a new terminal and try again.

## Next steps

- [Quick Start](/documentation/getting-started/quick-start) — go from zero to a populated `.env`
  in about five minutes.
- [Create your first project](/documentation/getting-started/first-project) — set up the
  project, target, and environment hierarchy.
- [CLI Authentication](/documentation/cli/authentication) — log the CLI in to your account.
