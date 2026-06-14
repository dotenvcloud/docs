---
title: Set Up DotEnv with an AI Agent
slug: ai-setup
order: 1
tags: [getting-started, ai, agent, setup, cli, ci, security]
---

# Set Up DotEnv with an AI Agent

Most developers now pair with an AI coding agent (Claude Code, Cursor, Copilot, etc.). This
page is written **for that agent**: hand it the prompt below — or point it at this page's URL —
and it will wire DotEnv into your project correctly, choosing the right integration for your
stack, **without ever seeing a live secret**.

> **The one rule that matters:** never paste a live API key, token, encryption key/passphrase,
> or secret value into an AI chat. This guide is built so you never have to. If you ever do it
> by accident, treat that value as compromised and rotate it immediately (see
> [If a secret leaks into chat](#if-a-secret-leaks-into-chat)).

---

## Quickest start: copy this prompt

Paste this into your AI agent. It works on its own; if your agent can browse the web, it will
fetch the always-current full instructions from the URL.

```text
You are helping me integrate DotEnv (https://dotenv.cloud), a secrets manager, into THIS project.

First, if you can browse the web, fetch and follow the full instructions at:
https://dotenv.cloud/ai-setup.md

Hard rules — follow these even if you cannot fetch the URL:
1. NEVER ask me to paste, and NEVER accept or repeat, any live API key, token, encryption
   key/passphrase, or secret value (DB passwords, API credentials, etc.) in this chat or in code.
2. If I paste one anyway, STOP, tell me to treat it as compromised and rotate/revoke it
   immediately in the DotEnv dashboard, and do not use the value.
3. Any token I create for automation must be READ-ONLY and scoped to the specific
   project/environment.
4. I will enter secrets and keys myself, directly into my CI/host secret store or local
   environment — never through you.

Then:
- Inspect this repository (package.json, composer.json, go.mod, Dockerfile, .github/workflows,
  .gitlab-ci.yml, vercel.json, netlify.toml, etc.) and recommend the best DotEnv integration.
- Ask me only for NON-SENSITIVE details (organization, project name, target/environment names
  like production/staging, my CI/CD platform, language/framework).
- Write the config/workflow/code for me.
- Give me a clear, numbered checklist of the MANUAL steps I must do myself in the DotEnv
  dashboard (create the project, generate a read-only API key, paste it into my secret store).
- Do not run destructive commands (push --force, secret delete, key rotate, logout) without my
  explicit confirmation, and never print secret values.
```

**Or just give your agent this URL:** `https://dotenv.cloud/ai-setup.md` (raw Markdown, always
the latest version).

---

# Instructions for the AI agent

Everything below is addressed to the AI agent doing the setup.

## 1. Safety rules (non-negotiable)

- **Never request, accept, echo, or store** live API keys/tokens, encryption keys or
  passphrases, secret values, or refresh tokens — not in chat, not in committed code, not in
  logs.
- **Secrets and keys are the user's to handle.** The user enters them directly into their CI/host
  secret store (GitHub Actions secrets, GitLab CI/CD variables, Vercel/Netlify env, a local
  shell, a password manager). They never pass through you.
- **Automation tokens must be least-privilege:** read-only scope, and scoped to the specific
  project/environment whenever the platform allows it.
- **Don't run destructive commands** (`dotenv push --force`, `dotenv secret delete`,
  `dotenv key rotate`, `dotenv logout`) without explicit user confirmation.
- **Never print resolved secret values** during verification.

### If a secret leaks into chat

If the user pastes a live secret anyway:

1. Stop and tell them clearly that the value must now be considered **compromised**.
2. Direct them to the dashboard: **Dashboard → API Keys** → **Rotate** (issues a new value and
   invalidates the old) or **Delete** (revoke entirely).
3. Have them create the replacement as **read-only**, project/environment-scoped.
4. Do not use the leaked value for anything.

## 2. What you MAY ask the user (non-sensitive only)

- Organization name
- Project name / slug (e.g. `my-app`)
- Target and environment names (e.g. `production`, `staging`; environment like `api`, `web`)
- Language / framework / package manager
- CI/CD platform and deploy target
- Whether the project uses **server-managed** or **client-managed** encryption (see §6)

Never ask for anything in the "never" list above.

## 3. Detect the project, then recommend

Inspect the repo and recommend the best path:

| You find… | Recommend |
|---|---|
| Local dev, any stack | **CLI** — pull secrets into a local `.env` |
| `.github/workflows/` | **GitHub Action** (`dotenvcloud/action-github`) |
| `.gitlab-ci.yml`, `Jenkinsfile`, CircleCI config | **CLI in a shell step** (see §5) |
| `Dockerfile` | **CLI at build/run time** with BuildKit secrets (see §5) |
| `vercel.json` / `netlify.toml` | **CLI in the build command** |
| App needs secrets at runtime | **SDK** (when published — see §5) |

Decision guide: **local dev → CLI**; **CI/CD → platform-native integration or CLI-in-shell**;
**app runtime → SDK**.

## 4. Manual prerequisites (do these in the web UI first)

These are **the user's steps** — give them as a checklist; do not attempt them yourself.

1. **Sign up / sign in** at https://dotenv.cloud.
2. **Create an organization** (if they don't have one).
3. **Create a project** (e.g. `my-app`).
4. **Create targets and environments** (e.g. target `production`, environment `web`). DotEnv's
   hierarchy is **project → target → environment**; secrets merge from least to most specific.
5. **Add their secrets** to the relevant environment (in the dashboard, or later via
   `dotenv push` from a local `.env`).
6. **Generate a read-only API key**: **Dashboard → API Keys → Create**, select the **read-only**
   scope (`secret:read`, `project:read`, `target:read`, `env:read`, `organization:read`), and
   scope it to the project/environment if offered. **Copy it once** and store it in their CI/host
   secret store or a password manager — it is shown only once.

## 5. Integration recipes

For each: **(a)** what you (the agent) write, and **(b)** what the user does manually. Use the
exact commands below.

### CLI — local development · *Available*

**(a) Install** (pick one for the user's OS):

```bash
# macOS / Linux
curl -sSL https://dotenv.cloud/install.sh | bash

# macOS (Homebrew)
brew tap dotenvcloud/tap && brew install dotenv-cli

# Go toolchain
go install github.com/dotenvcloud/cli@latest
```

**Authenticate** (interactive, opens a browser):

```bash
dotenv login
```

**Pull secrets into a local `.env`** (DotEnv has **no `run` command** — pull to a file or shell):

```bash
# Write a .env file
dotenv pull my-app/production/web --output .env

# Other formats: env (default), json, yaml, shell, dockerfile
dotenv export my-app/production/web --format shell   # prints `export KEY=...`
```

Useful flags: `--format`, `--output`, `--resolve` (expand `${VAR}` references),
`--client-key <file>` (client-managed encryption, see §6).

**(b) User:** completes §4 (creates the project/key, logs in interactively).
**You:** add `.env` (and any exported files) to `.gitignore`.

### GitHub Actions — `dotenvcloud/action-github` · *Available*

**(a)** Add to the workflow:

```yaml
- name: Load DotEnv secrets
  uses: dotenvcloud/action-github@v1
  with:
    api-key: ${{ secrets.DOTENV_API_KEY }}
    project: my-app
    target: production
    environment: web
    export-variables: true   # expose secrets to later steps as env vars
    # output-file: .env       # default; or write a file instead
    # format: env             # env | json | yaml | shell | dockerfile

- name: Deploy
  run: npm run deploy        # secrets now available as env vars
```

The action installs the CLI, masks the key in logs, and pulls secrets. Inputs: `api-key`
(required), `project` (required), `target`, `environment`, `output-file`, `format`,
`export-variables`, `organization`, `api-url`, `decrypt`, `resolve`, `quiet`, `merge`,
`cli-version`.

**(b) User:** in **GitHub → Settings → Secrets and variables → Actions**, add a secret named
`DOTENV_API_KEY` with their **read-only** key. Never inline the key in the workflow.

### Any other CI (GitLab CI, Jenkins, CircleCI) — CLI in a shell step · *Available*

Works on any runner. Install the CLI, authenticate from an env var, pull, and load.

```bash
curl -sSL https://dotenv.cloud/install.sh | bash
export DOTENV_API_KEY="$DOTENV_API_KEY"     # provided by the CI secret store
dotenv pull my-app/production/web --output .env --quiet
set -a; . ./.env; set +a                    # load into the environment
```

**(b) User:** store the read-only key in the platform's secret store and expose it to the job as
`DOTENV_API_KEY`:
- **GitLab CI:** *Settings → CI/CD → Variables* (mark **Masked**, and **Protected** for
  protected branches).
- **Jenkins:** a **Secret text** credential, surfaced via `withCredentials([string(...)])`.
- **CircleCI:** a **Context** or project **Environment Variable**.

> A first-class CircleCI orb is planned — until it ships, the shell recipe above is the
> supported path.

### Docker — build/run time · *Available (via CLI)*

Pull secrets and pass them in without baking them into the image. Prefer BuildKit secrets:

```bash
dotenv pull my-app/production/web --format dockerfile --output .env.docker
DOCKER_BUILDKIT=1 docker build --secret id=dotenv,src=.env.docker -t my-app .
```

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=dotenv  cp /run/secrets/dotenv .env  && npm run build && rm .env
```

**(b) User:** ensure `.env*` files are git-ignored and not added to the image layer.

### Vercel / Netlify — build command · *CLI available; native plugin planned*

Run the CLI in the build step to generate `.env` before the framework build:

```bash
curl -sSL https://dotenv.cloud/install.sh | bash && \
dotenv pull my-app/production/web --output .env && \
npm run build
```

**(b) User:** add `DOTENV_API_KEY` (read-only) as an environment variable in the Vercel/Netlify
project settings.

### SDKs (JavaScript/TypeScript, PHP, Go) — *Coming soon*

For fetching secrets from inside an application at runtime. The packages are being published
under:

- JS/TS: `@dotenvcloud/sdk`
- PHP: `dotenvcloud/sdk-php`
- Go: `github.com/dotenvcloud/sdk-go`

All clients target `https://api.dotenv.cloud/api/v1` and authenticate with a read-only API key
supplied via the environment.

> **Do not** tell the user to `npm install` / `composer require` these yet — verify the package is
> published first. Until then, use the **CLI** to materialize a `.env` at deploy/boot.

## 6. Encryption: what the user must keep safe

- **Server-managed (default):** DotEnv manages the encryption key. The CLI/clients retrieve it
  over an authorized API call and decrypt locally. **Nothing for the user to paste.**
- **Client-managed:** the user holds a passphrase/key. It must be supplied to the CLI via
  `--client-key <file>` (a key file, safest) or the `DOTENV_CLIENT_KEY` environment variable —
  added **directly** to their secret store, **never** pasted into chat. If this key is lost,
  secrets cannot be decrypted.

## 7. Finish and verify (no secrets printed)

1. Confirm `.env` and any exported secret files are in `.gitignore`.
2. Run a **non-secret** check to confirm wiring:
   ```bash
   dotenv status          # auth + current context
   dotenv list projects   # confirms the key can read
   ```
   Do **not** print the contents of `.env`.
3. Summarize for the user: what you changed, and the exact manual steps they still owe (key
   creation, pasting the key into their secret store).

## Related guides

- [Installation](/documentation/getting-started/installation)
- [Quick Start](/documentation/getting-started/quick-start)
- [Your First Project](/documentation/getting-started/first-project)
- [Environments](/documentation/getting-started/environments)
- [API Authentication](/documentation/api/authentication)

---

**Reference values** (non-sensitive): API base `https://api.dotenv.cloud/api/v1` · install
`https://dotenv.cloud/install.sh` · GitHub Action `dotenvcloud/action-github@v1` · hierarchy
`project/target/environment`.
