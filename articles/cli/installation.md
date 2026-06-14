---
title: Installation
slug: installation
order: 2
description: Install the DotEnv CLI on macOS, Linux, and Windows, then update and verify it.
tags: [cli, installation, install, update, homebrew, windows]
---

# Installation

The DotEnv CLI ships as a single self-contained binary named `dotenv`. Pick the method that
fits your platform.

## macOS and Linux

### Install script (recommended)

```bash
curl -sSL https://dotenv.cloud/install.sh | bash
```

The script detects your OS and architecture, downloads the matching binary, and installs it
(falling back to `~/.local/bin` if it cannot write to `/usr/local/bin`).

### Homebrew (macOS)

```bash
brew tap dotenvcloud/tap
brew install dotenv-cli
```

The Homebrew formula is named `dotenv-cli`; the installed binary is still `dotenv`.

## Windows

### Scoop

```powershell
scoop bucket add dotenv https://github.com/dotenvcloud/scoop-bucket
scoop install dotenv-cli
```

### PowerShell install script

```powershell
irm https://dotenv.cloud/install.ps1 | iex
```

## From source (any platform with Go)

If you have a Go toolchain installed:

```bash
go install github.com/dotenvcloud/cli@latest
```

The binary is installed into your `$(go env GOPATH)/bin` directory; make sure that is on your
`PATH`.

## Nightly builds

Nightly builds track the latest commit on `main`. They are bleeding-edge and carry no
stability guarantees — use the stable channel for production.

```bash
curl -sSL https://dotenv.cloud/install.sh | bash -s -- --nightly
```

Nightly builds are only available through the install script, not through Homebrew.

## Updating

The CLI can update itself in place:

```bash
# Check for and install the latest version (prompts before installing)
dotenv update

# Only check, do not install
dotenv update --check

# Install a specific version
dotenv update --version=1.2.3

# Update even if you are already on the latest version
dotenv update --force
```

If you installed via Homebrew, prefer `brew upgrade dotenv-cli` so Homebrew keeps tracking the
binary. If you installed via `go install`, re-run `go install github.com/dotenvcloud/cli@latest`.

## Verifying the install

Confirm the binary is on your `PATH` and check the version:

```bash
# Full version details (version, commit, build date, Go version)
dotenv version

# Just the version number
dotenv version --short
```

If `dotenv` is not found after installing, make sure the install directory
(`/usr/local/bin`, `~/.local/bin`, or your Go bin directory) is on your `PATH`.

## Shell completion

Generate a completion script for your shell:

```bash
# Bash
source <(dotenv completion bash)

# Zsh
dotenv completion zsh > "${fpath[1]}/_dotenv"

# Fish
dotenv completion fish | source

# PowerShell
dotenv completion powershell | Out-String | Invoke-Expression
```

Run `dotenv completion --help` for instructions on installing the script permanently.
