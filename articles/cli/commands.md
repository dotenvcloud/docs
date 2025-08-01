---
title: Command Reference
slug: commands
order: 2
tags: [cli, commands, reference]
---

# Command Reference

Complete reference for all DotEnv CLI commands, organized by category.

## Global Options

These options work with all commands:

```bash
--help, -h              Show help
--version, -v           Show version
--config PATH           Config file path
--format FORMAT         Output format (table|json|yaml|env)
--no-color             Disable colored output
--quiet, -q            Suppress non-error output
--verbose              Show detailed output
--debug                Enable debug mode
```

## Authentication Commands

### login

Authenticate with DotEnv.

```bash
dotenv login [flags]
```

**Flags:**

- `--method METHOD` - Auth method (browser|token|sso)
- `--token TOKEN` - Use API token directly
- `--no-browser` - Don't open browser automatically
- `--sso` - Use SSO authentication

**Examples:**

```bash
# Interactive browser login
dotenv login

# Token-based login
dotenv login --token dotenv_api_prod_x9Kl3mN7pQ2vR8s

# SSO login
dotenv login --sso --org acme-corp
```

### logout

Logout from DotEnv.

```bash
dotenv logout [flags]
```

**Flags:**

- `--all` - Logout from all sessions
- `--force` - Don't prompt for confirmation

**Examples:**

```bash
# Standard logout
dotenv logout

# Logout everywhere
dotenv logout --all
```

### whoami

Show current authenticated user.

```bash
dotenv whoami [flags]
```

**Flags:**

- `--show-token` - Display API token (careful!)
- `--show-permissions` - List permissions

**Examples:**

```bash
# Basic info
dotenv whoami
# Output: alice@example.com (ACME Corp)

# Detailed info
dotenv whoami --show-permissions
```

## Project Commands

### projects

Alias for `projects list`.

```bash
dotenv projects [flags]
```

### projects list

List all accessible projects.

```bash
dotenv projects list [flags]
```

**Flags:**

- `--org ORG` - Filter by organization
- `--tag TAG` - Filter by tag
- `--archived` - Include archived projects
- `--sort FIELD` - Sort by field (name|created|updated)

**Examples:**

```bash
# List all projects
dotenv projects list

# Filter by organization
dotenv projects list --org acme-corp

# Include archived
dotenv projects list --archived
```

### projects create

Create a new project.

```bash
dotenv projects create [NAME] [flags]
```

**Flags:**

- `--display-name NAME` - Human-readable name
- `--description DESC` - Project description
- `--tags TAGS` - Comma-separated tags
- `--private` - Make project private

**Examples:**

```bash
# Interactive creation
dotenv projects create

# With options
dotenv projects create api-service \
  --display-name "API Service" \
  --description "Main REST API" \
  --tags "backend,production"
```

### projects delete

Delete a project.

```bash
dotenv projects delete [PROJECT] [flags]
```

**Flags:**

- `--force` - Skip confirmation
- `--archive` - Archive instead of delete

**Examples:**

```bash
# Delete with confirmation
dotenv projects delete old-project

# Archive instead
dotenv projects delete old-project --archive
```

### projects use

Set the active project.

```bash
dotenv projects use [PROJECT] [flags]
dotenv use [PROJECT] [flags]  # Alias
```

**Flags:**

- `--global` - Set as global default
- `--local` - Set for current directory only

**Examples:**

```bash
# Interactive selection
dotenv use

# Set specific project
dotenv use my-app

# Set global default
dotenv use my-app --global
```

## Secret Commands

### secrets

List secrets (alias for `secrets list`).

```bash
dotenv secrets [flags]
```

### secrets list

List all secrets in current project/environment.

```bash
dotenv secrets list [flags]
```

**Flags:**

- `--project PROJECT` - Specify project
- `--environment ENV` - Specify environment
- `--show-values` - Display actual values
- `--filter PATTERN` - Filter by key pattern
- `--tag TAG` - Filter by tag

**Examples:**

```bash
# List current project secrets
dotenv secrets

# Show values
dotenv secrets --show-values

# Filter by pattern
dotenv secrets --filter "API_*"

# Specific environment
dotenv secrets --environment production
```

### secrets get

Get a specific secret value.

```bash
dotenv secrets get KEY [flags]
dotenv get KEY [flags]  # Alias
```

**Flags:**

- `--project PROJECT` - Specify project
- `--environment ENV` - Specify environment
- `--raw` - Output raw value only
- `--copy` - Copy to clipboard

**Examples:**

```bash
# Get secret
dotenv get DATABASE_URL

# Copy to clipboard
dotenv get API_KEY --copy

# Raw output for scripts
export DB=$(dotenv get DATABASE_URL --raw)
```

### secrets set

Set a secret value.

```bash
dotenv secrets set KEY=VALUE [KEY2=VALUE2...] [flags]
dotenv set KEY=VALUE [flags]  # Alias
```

**Flags:**

- `--project PROJECT` - Specify project
- `--environment ENV` - Specify environment
- `--interactive, -i` - Interactive mode
- `--from-file FILE` - Read value from file
- `--expires DURATION` - Set expiration
- `--tag TAG` - Add tags

**Examples:**

```bash
# Set single secret
dotenv set DATABASE_URL="postgresql://localhost/mydb"

# Set multiple secrets
dotenv set API_KEY="sk_123" REDIS_URL="redis://localhost"

# Interactive mode (hides input)
dotenv set DATABASE_PASSWORD --interactive

# From file
dotenv set SSL_CERT --from-file cert.pem

# With expiration
dotenv set TEMP_TOKEN="abc123" --expires 24h
```

### secrets delete

Delete secrets.

```bash
dotenv secrets delete KEY [KEY2...] [flags]
dotenv delete KEY [flags]  # Alias
```

**Flags:**

- `--project PROJECT` - Specify project
- `--environment ENV` - Specify environment
- `--force` - Skip confirmation
- `--all` - Delete all secrets (dangerous!)

**Examples:**

```bash
# Delete single secret
dotenv delete OLD_API_KEY

# Delete multiple
dotenv delete KEY1 KEY2 KEY3

# Delete all (with confirmation)
dotenv delete --all
```

### secrets pull

Download secrets to local .env file.

```bash
dotenv secrets pull [flags]
dotenv pull [flags]  # Alias
```

**Flags:**

- `--project PROJECT` - Specify project
- `--environment ENV` - Specify environment
- `--output FILE` - Output file (default: .env)
- `--format FORMAT` - Output format (env|json|yaml|shell)
- `--overwrite` - Overwrite existing file
- `--merge` - Merge with existing file

**Examples:**

```bash
# Pull to .env
dotenv pull

# Pull to specific file
dotenv pull --output .env.production

# Pull as JSON
dotenv pull --format json --output secrets.json

# Merge with existing
dotenv pull --merge

# Shell format for sourcing
dotenv pull --format shell > env.sh
source env.sh
```

### secrets push

Upload local .env file to DotEnv.

```bash
dotenv secrets push [flags]
dotenv push [flags]  # Alias
```

**Flags:**

- `--project PROJECT` - Specify project
- `--environment ENV` - Specify environment
- `--input FILE` - Input file (default: .env)
- `--replace` - Replace all existing secrets
- `--dry-run` - Preview changes without applying

**Examples:**

```bash
# Push .env file
dotenv push

# Push specific file
dotenv push --input .env.production

# Preview changes
dotenv push --dry-run

# Replace all secrets
dotenv push --replace
```

### secrets rotate

Rotate secret values.

```bash
dotenv secrets rotate KEY [flags]
```

**Flags:**

- `--project PROJECT` - Specify project
- `--environment ENV` - Specify environment
- `--all` - Rotate all secrets
- `--tag TAG` - Rotate by tag
- `--generate` - Auto-generate new value

**Examples:**

```bash
# Rotate specific secret
dotenv secrets rotate API_KEY

# Auto-generate new value
dotenv secrets rotate DATABASE_PASSWORD --generate

# Rotate all secrets with tag
dotenv secrets rotate --tag "rotation:monthly"
```

## Environment Commands

### environments

Alias for `environments list`.

```bash
dotenv environments [flags]
dotenv env [flags]  # Alias
```

### environments list

List all environments.

```bash
dotenv environments list [flags]
dotenv env list [flags]  # Alias
```

**Flags:**

- `--project PROJECT` - Specify project
- `--show-inheritance` - Show inheritance chain

**Examples:**

```bash
# List environments
dotenv env list

# Show inheritance
dotenv env list --show-inheritance
```

### environments create

Create a new environment.

```bash
dotenv environments create NAME [flags]
dotenv env create NAME [flags]  # Alias
```

**Flags:**

- `--project PROJECT` - Specify project
- `--inherits-from ENV` - Inherit from environment
- `--description DESC` - Environment description
- `--clone-from ENV` - Clone existing environment

**Examples:**

```bash
# Create environment
dotenv env create qa

# With inheritance
dotenv env create staging --inherits-from development

# Clone environment
dotenv env create production-backup --clone-from production
```

### environments use

Switch active environment.

```bash
dotenv environments use NAME [flags]
dotenv env use NAME [flags]  # Alias
```

**Flags:**

- `--project PROJECT` - Specify project
- `--global` - Set as global default

**Examples:**

```bash
# Interactive selection
dotenv env use

# Switch environment
dotenv env use production

# Set global default
dotenv env use development --global
```

### environments delete

Delete an environment.

```bash
dotenv environments delete NAME [flags]
dotenv env delete NAME [flags]  # Alias
```

**Flags:**

- `--project PROJECT` - Specify project
- `--force` - Skip confirmation

**Examples:**

```bash
# Delete environment
dotenv env delete old-env

# Force delete
dotenv env delete temp-env --force
```

## Run Commands

### run

Run a command with secrets loaded.

```bash
dotenv run [flags] -- COMMAND [ARGS...]
```

**Flags:**

- `--project PROJECT` - Specify project
- `--environment ENV` - Specify environment
- `--inherit` - Inherit current environment
- `--override` - Override existing env vars

**Examples:**

```bash
# Run npm start with secrets
dotenv run -- npm start

# Run with specific environment
dotenv run --environment production -- node server.js

# Run shell command
dotenv run -- bash -c 'echo $DATABASE_URL'

# Override existing vars
dotenv run --override -- python app.py
```

### exec

Alias for `run`.

```bash
dotenv exec [flags] -- COMMAND [ARGS...]
```

## Organization Commands

### orgs list

List organizations.

```bash
dotenv orgs list [flags]
```

**Flags:**

- `--show-role` - Show your role

**Examples:**

```bash
# List organizations
dotenv orgs list

# With roles
dotenv orgs list --show-role
```

### orgs use

Switch active organization.

```bash
dotenv orgs use [ORG] [flags]
```

**Examples:**

```bash
# Interactive selection
dotenv orgs use

# Switch organization
dotenv orgs use acme-corp
```

## Config Commands

### config

Show current configuration.

```bash
dotenv config [flags]
```

**Examples:**

```bash
# Show all config
dotenv config

# Show as JSON
dotenv config --format json
```

### config get

Get a config value.

```bash
dotenv config get KEY [flags]
```

**Examples:**

```bash
# Get config value
dotenv config get default_project

# Get API endpoint
dotenv config get api_endpoint
```

### config set

Set a config value.

```bash
dotenv config set KEY VALUE [flags]
```

**Examples:**

```bash
# Set default project
dotenv config set default_project my-app

# Set output format
dotenv config set format json

# Disable colors
dotenv config set color_output false
```

## Utility Commands

### doctor

Diagnose issues with DotEnv CLI.

```bash
dotenv doctor [flags]
```

**Flags:**

- `--fix` - Attempt to fix issues

**Examples:**

```bash
# Run diagnostics
dotenv doctor

# Auto-fix issues
dotenv doctor --fix
```

### update

Update DotEnv CLI to latest version.

```bash
dotenv update [flags]
```

**Flags:**

- `--check` - Check for updates only
- `--force` - Force reinstall

**Examples:**

```bash
# Update CLI
dotenv update

# Check only
dotenv update --check
```

### completion

Generate shell completion scripts.

```bash
dotenv completion SHELL [flags]
```

**Supported shells:**

- bash
- zsh
- fish
- powershell

**Examples:**

```bash
# Bash
dotenv completion bash > /etc/bash_completion.d/dotenv

# Zsh
dotenv completion zsh > "${fpath[1]}/_dotenv"

# Fish
dotenv completion fish > ~/.config/fish/completions/dotenv.fish
```

## Import/Export Commands

### import

Import secrets from various formats.

```bash
dotenv import [FILE] [flags]
```

**Flags:**

- `--format FORMAT` - Input format (env|json|yaml)
- `--project PROJECT` - Target project
- `--environment ENV` - Target environment

**Examples:**

```bash
# Import .env file
dotenv import .env

# Import JSON
dotenv import secrets.json --format json

# Import from stdin
cat secrets.yaml | dotenv import - --format yaml
```

### export

Export secrets to various formats.

```bash
dotenv export [flags]
```

**Flags:**

- `--format FORMAT` - Output format (env|json|yaml|shell)
- `--output FILE` - Output file
- `--project PROJECT` - Source project
- `--environment ENV` - Source environment

**Examples:**

```bash
# Export as JSON
dotenv export --format json > secrets.json

# Export for shell
dotenv export --format shell > env.sh

# Export specific environment
dotenv export --environment production --output prod.env
```

## Team Commands

### members list

List organization members.

```bash
dotenv members list [flags]
```

**Flags:**

- `--org ORG` - Specify organization
- `--project PROJECT` - Show project members

**Examples:**

```bash
# List org members
dotenv members list

# List project members
dotenv members list --project my-app
```

### members invite

Invite new members.

```bash
dotenv members invite EMAIL [flags]
```

**Flags:**

- `--role ROLE` - Assign role
- `--project PROJECT` - Add to project
- `--message MSG` - Custom message

**Examples:**

```bash
# Invite to organization
dotenv members invite alice@example.com --role developer

# Invite to project
dotenv members invite bob@example.com --project my-app
```

## Advanced Examples

### Complex Workflows

```bash
# Development setup
dotenv use my-app && \
dotenv env use development && \
dotenv pull && \
dotenv run -- npm install && \
dotenv run -- npm start

# Production deployment
dotenv pull \
  --project my-app \
  --environment production \
  --output .env.production && \
docker build --secret id=env,src=.env.production .

# Batch operations
for env in development staging production; do
  dotenv secrets list \
    --environment $env \
    --format json > ${env}-secrets.json
done

# Secret sync
dotenv export --environment staging | \
  dotenv import - --environment qa
```

## Next Steps

- [Configuration](./configuration) - Detailed configuration options
- [Authentication](./authentication) - Auth methods and security
- [Advanced Usage](./advanced-usage) - Power user features
- [Troubleshooting](./troubleshooting) - Common issues
- [Scripting](./scripting) - Automation examples
