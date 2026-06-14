---
title: Command Reference
slug: commands
order: 4
description: Complete reference for every DotEnv CLI command, its real flags, and an example of each.
tags: [cli, commands, reference, flags]
---

# Command Reference

Every command and flag below is taken directly from the CLI source. Paths use the
`project/target/environment` hierarchy.

## Global flags

These persistent flags work on every command:

| Flag | Description |
|------|-------------|
| `--config <file>` | Config file to use (default `$HOME/.dotenv/config.yaml`). |
| `--debug` | Enable debug output. |
| `--quiet` | Suppress non-error output. |
| `--no-color` | Disable colored output. |
| `--api-key <key>` | API key for authentication; overrides the local account system. |

```bash
dotenv --debug list projects
dotenv --api-key=xxxxx pull myapp/production/api
```

---

## Setup commands

### init

Initialize the CLI config interactively (API URL, telemetry, and first account).

```bash
dotenv init
dotenv init --force   # overwrite existing configuration
```

| Flag | Description |
|------|-------------|
| `--force` | Overwrite an existing configuration. |

### login

Authenticate via OAuth or API key. See [Authentication](./authentication.md).

```bash
dotenv login
```

| Flag | Description |
|------|-------------|
| `--no-browser` | Print the OAuth URL instead of opening a browser. |
| `--callback-port <port>` | Pin the OAuth callback port (default: auto). |
| `--api-key` | Use organization API key authentication instead of OAuth. |
| `--status` | Show current authentication status. |

### logout

Log out of one or all accounts.

```bash
dotenv logout
dotenv logout work@example.com
dotenv logout --all
```

| Flag | Description |
|------|-------------|
| `--all` | Log out of all accounts. |
| `-f, --force` | Skip the confirmation prompt. |

### status

Show current account, organization, and token status (local state only).

```bash
dotenv status
```

### version

Print version information.

```bash
dotenv version
dotenv version --short
```

| Flag | Description |
|------|-------------|
| `-s, --short` | Print just the version number. |

### update

Update the CLI to the latest (or a specific) version.

```bash
dotenv update
```

| Flag | Description |
|------|-------------|
| `--check` | Only check for updates, do not install. |
| `--version <ver>` | Install a specific version. |
| `--force` | Update even if already on the latest version. |

### completion

Generate a shell completion script.

```bash
dotenv completion zsh
```

Supported shells: `bash`, `zsh`, `fish`, `powershell`.

---

## Secrets I/O

### pull

Pull secrets for a path and write them to stdout or a file. See
[Pulling and pushing](./pulling-and-pushing.md) for the full guide.

```bash
dotenv pull myapp/production/api --output .env
```

| Flag | Description |
|------|-------------|
| `-o, --output <file>` | Write to a file instead of stdout. |
| `-r, --resolve` | Resolve `${VAR}` interpolation. |
| `-f, --format <fmt>` | Output format: `env` (default), `json`, `yaml`, `shell`, `dockerfile`. |
| `--client-key <file\|value>` | Client encryption key file or value (client-managed projects). |
| `--decrypt` | Decrypt secrets (default `true`; set `--decrypt=false` for raw encrypted values). |
| `-q, --quiet` | Suppress output (exit code only). |
| `-m, --merge` | Merge secrets across all hierarchy levels (default `true`). |
| `--level-only` | Show only the specified level, not merged with parents. |

### push

Push a local file (or per-level files) to a path.

```bash
dotenv push myapp/production/api .env
```

| Flag | Description |
|------|-------------|
| `--project <file>` | File containing project-level secrets (multi-file mode). |
| `--target <file>` | File containing target-level secrets (multi-file mode). |
| `--env <file>` | File containing environment-level secrets (multi-file mode). |
| `-f, --force` | Overwrite existing secrets without prompting. |
| `--client-key <file\|value>` | Client encryption key file or value (client-managed projects). |
| `--encrypt` | Encrypt before pushing (default `true`). |
| `--no-backup` | Do not record a backup version for this push. |

### export

Export secrets in a chosen format (a thin wrapper over `pull` with export defaults).

```bash
dotenv export myapp/production/api --format=json
dotenv export myapp --format=shell > exports.sh
```

| Flag | Description |
|------|-------------|
| `-f, --format <fmt>` | Export format (`env`, `json`, `yaml`, `shell`, `dockerfile`). |
| `-o, --output <file>` | Output file (default: stdout). |
| `--template <file>` | Custom template file (not implemented yet). |
| `-q, --quiet` | Suppress non-error output. |

### secret delete

Clear the encrypted secret blob stored at a level.

```bash
dotenv secret delete myapp/production/api
```

| Flag | Description |
|------|-------------|
| `-f, --force` | Skip the confirmation prompt. |
| `--no-backup` | Also purge version history and hard-delete (nothing recoverable). |

By default a backup version is kept, so a deleted secret can be restored with
[`dotenv secret restore`](./secret-versioning.md).

---

## Browse commands

### list

List resources in the current organization.

```bash
dotenv list projects
dotenv list targets myapp
dotenv list environments myapp/production
dotenv list accounts
dotenv list all
```

Resources: `organizations`, `projects`, `targets`, `environments`, `accounts`, `all`.

| Flag | Description |
|------|-------------|
| `--organization <org>` | Override the current account's organization. |
| `-f, --format <fmt>` | Output format: `table` (default), `json`, `yaml`. |
| `--json` | Output as JSON (deprecated — use `--format=json`). |
| `--paths` | Output full paths for copy/paste. |
| `--with-paths` | Include paths in JSON/YAML output. |

### tree

Show the project/target/environment hierarchy as a tree.

```bash
dotenv tree
dotenv tree myapp --full
```

| Flag | Description |
|------|-------------|
| `-p, --project <slug>` | Show the tree for a specific project. |
| `-t, --target <slug>` | Show the tree for a specific target (requires a project). |
| `--full` | Include environments. |
| `--max-depth <n>` | Maximum depth to display (0 = unlimited). |
| `--show-counts` | Show resource counts at each level. |
| `--format <fmt>` | Output format: `tree` (default), `json`, `yaml`. |

### explore

Interactively browse projects, targets, and environments.

```bash
dotenv explore
dotenv explore myapp/production
```

| Flag | Description |
|------|-------------|
| `--action <action>` | Default action when selecting a resource: `select`, `copy`, `pull`. |

### path

Find resources by name and print their paths.

```bash
dotenv path prod
dotenv path "prod.*api" --regex
```

| Flag | Description |
|------|-------------|
| `--exact` | Match exact names only. |
| `--regex` | Treat the search term as a regex pattern. |
| `--type <type>` | Restrict to `all` (default), `project`, `target`, or `environment`. |
| `--output <fmt>` | Output format: `list` (default), `first`, `count`. |

---

## Resource management

### project

Create, update, and delete projects.

```bash
dotenv project create my-app --description "Main application"
dotenv project update my-app --name "My App"
dotenv project delete my-app
```

`project create` flags:

| Flag | Description |
|------|-------------|
| `--description <text>` | Project description. |
| `--format <fmt>` | Secret format: `env`, `json`, `yaml`, `text`. |
| `--storage <mode>` | Encryption storage mode: `client` (default) or `server`. |
| `--client-key <file\|value>` | Client encryption key for a client-managed project (prompted if omitted). |

`project update` flags: `--name`, `--description`, `--slug`.
`project delete` flag: `-f, --force`.

### target

Create, update, and delete targets within a project.

```bash
dotenv target create my-app production
dotenv target update my-app production --name "Prod"
dotenv target delete my-app production
```

`target create` flag: `--description`.
`target update` flags: `--name`, `--description`, `--slug`.
`target delete` flag: `-f, --force`.

### environment

Create, update, and delete environments within a target. Aliased as `env`.

```bash
dotenv environment create my-app production api --status production
dotenv environment update my-app production api --name "API"
dotenv environment delete my-app production api
```

`environment create` flags: `--status` (production, staging, development, local), `--description`.
`environment update` flags: `--name`, `--status`, `--description`, `--slug`.
`environment delete` flag: `-f, --force`.

---

## Accounts and organizations

### account

Manage local accounts. See [Authentication](./authentication.md).

```bash
dotenv account list
dotenv account use work@example.com
dotenv account add
dotenv account remove old-account
dotenv account refresh
dotenv account rename old-name new-name
```

`account remove` is also available as `rm` / `delete`.

### org

Manage organizations for the current account. Aliased as `orgs`, `organization`,
`organizations`.

```bash
dotenv org list
dotenv org use acme-corp
dotenv org refresh
dotenv org show
```

| Flag | Description |
|------|-------------|
| `-f, --format <fmt>` | (`org list` only) Output format: `table`, `json`, `yaml`. |
| `--no-refresh` | Skip the automatic organization refresh. |

### refresh

Top-level alias for `dotenv account refresh`.

```bash
dotenv refresh
```

### auth

View authentication and user information.

```bash
dotenv auth info
dotenv auth info --verbose
```

`auth info` flag: `-v, --verbose` (show organization permissions and details).

---

## API keys

### apikeys

Manage organization API keys for programmatic access.

```bash
dotenv apikeys list
dotenv apikeys create "CI/CD Key" --abilities="secrets:read,projects:read"
dotenv apikeys update key-id --name="New Name"
dotenv apikeys delete key-id
dotenv apikeys rotate key-id
```

| Subcommand | Flags |
|------------|-------|
| `create [name]` | `--abilities` (required, comma-separated), `--expires` (RFC3339). |
| `update [key-id]` | `--name` (required). |
| `delete [key-id]` | `-f, --force`. |
| `rotate [key-id]` | `-f, --force`. |

Available abilities include `secrets:read`, `secrets:write`, `projects:read`,
`projects:write`, `targets:read`, `targets:write`, `environments:read`,
`environments:write`, `encryption:read`, `encryption:write`, `apikeys:read`,
`apikeys:write`. The token is shown only once at creation/rotation — save it.

---

## Encryption keys

### key

Inspect and manage a project's encryption keys. See
[Client-side encryption](./client-side-encryption.md).

```bash
dotenv key history my-app
dotenv key rotate my-app
dotenv key re-encrypt-history my-app
```

| Subcommand | Description / Flags |
|------------|---------------------|
| `history <project>` | List the encryption key version history. |
| `rotate <project>` | Rotate a **server-managed** key. Flags: `--history-policy` (`keep` \| `re_encrypt`), `-f, --force`. |
| `re-encrypt-history <project>` | Move backup versions under old **client-managed** keys onto the current key (decrypted/re-encrypted locally). Flags: `--old-key` (file or value, repeatable), `--client-key` (current key file or value). |

Client-managed keys cannot be rotated from the CLI — use the web dashboard.

---

## Secret versions

See [Secret versioning](./secret-versioning.md) for the full guide.

```bash
dotenv secret versions myapp/production/api
dotenv secret show myapp/production/api --version 42
dotenv secret diff myapp/production/api 42
dotenv secret restore 42
dotenv secret purge myapp/production/api
```

| Subcommand | Flags |
|------------|-------|
| `versions <path>` | `--limit` (default 50, max 100), `--page`. |
| `show <path> --version <id>` | `--version` (required), `-o, --output`, `--client-key`, `--old-key` (repeatable). |
| `diff <path> <version> [version-b]` | `--client-key`, `--old-key` (repeatable), `--show-values`. |
| `restore <version-id>` | `-f, --force`. |
| `purge <path>` | `--project-wide`, `-f, --force`. |
