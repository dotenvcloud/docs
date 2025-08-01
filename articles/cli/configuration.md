---
title: CLI Configuration
slug: configuration
order: 3
tags: [cli, configuration, settings]
---

# CLI Configuration

The DotEnv CLI can be configured through multiple methods to suit your workflow. This guide covers all configuration options and best practices.

## Configuration Hierarchy

Configuration is loaded in this order (later sources override earlier):

1. Default values
2. Global config file (`~/.dotenv/config.json`)
3. Project config file (`.dotenv.yml`)
4. Environment variables
5. Command-line flags

```bash
# Example hierarchy
Default: format=table
Global config: format=json
Project config: format=yaml
Environment: DOTENV_FORMAT=env
Command flag: --format shell
# Result: format=shell (command flag wins)
```

## Global Configuration

### Location

Global configuration is stored in:

- **macOS/Linux**: `~/.dotenv/config.json`
- **Windows**: `%USERPROFILE%\.dotenv\config.json`

### Configuration File

```json
{
    "version": "1.0",
    "api_endpoint": "https://api.dotenv.cloud",
    "organization": "acme-corp",
    "default_project": "my-app",
    "default_environment": "development",
    "format": "table",
    "color_output": true,
    "auto_update": true,
    "telemetry": true,
    "cache": {
        "enabled": true,
        "ttl": 300,
        "directory": "~/.dotenv/cache"
    },
    "security": {
        "keychain_enabled": true,
        "session_timeout": 3600,
        "require_mfa": false
    },
    "aliases": {
        "prod": "production",
        "dev": "development",
        "stg": "staging"
    }
}
```

### Managing Global Config

```bash
# View all config
dotenv config

# Get specific value
dotenv config get default_project

# Set value
dotenv config set default_project my-app

# Reset to defaults
dotenv config reset

# Edit directly
dotenv config edit
```

## Project Configuration

### .dotenv.yml

Create `.dotenv.yml` in your project root:

```yaml
# Project identification
project: my-app
organization: acme-corp

# Default environment
environment: development

# Required secrets
required:
    - DATABASE_URL
    - API_KEY
    - JWT_SECRET

# Optional secrets with defaults
optional:
    PORT: "3000"
    LOG_LEVEL: "info"
    NODE_ENV: "development"

# Environment-specific configs
environments:
    development:
        inherits: base
        required:
            - DEV_DATABASE_URL
        optional:
            DEBUG: "true"

    staging:
        inherits: development
        required:
            - STAGING_API_KEY

    production:
        required:
            - PROD_DATABASE_URL
            - PROD_API_KEY
        optional:
            SENTRY_DSN: ""

# Scripts
scripts:
    setup: |
        dotenv pull
        npm install

    deploy: |
        dotenv pull --environment production
        npm run build
        npm run deploy

# Hooks
hooks:
    pre_pull: "npm run pre-pull"
    post_pull: "npm run post-pull"
    pre_push: "npm test"

# Settings
settings:
    auto_pull: true
    merge_strategy: "override"
    validation: "strict"
```

### .dotenv.json Alternative

You can also use JSON format:

```json
{
    "project": "my-app",
    "environment": "development",
    "required": ["DATABASE_URL", "API_KEY"],
    "optional": {
        "PORT": "3000",
        "LOG_LEVEL": "info"
    }
}
```

## Environment Variables

Configure the CLI using environment variables:

### Authentication

```bash
# API Key
export DOTENV_API_KEY="dotenv_api_prod_abc123"

# OAuth Token
export DOTENV_TOKEN="eyJ0eXAiOiJKV1QiLCJhbGc..."
```

### Project Settings

```bash
# Default project
export DOTENV_PROJECT="my-app"

# Default environment
export DOTENV_ENVIRONMENT="production"

# Organization
export DOTENV_ORGANIZATION="acme-corp"
```

### Output Settings

```bash
# Output format (table|json|yaml|env|shell)
export DOTENV_FORMAT="json"

# Disable colors
export DOTENV_NO_COLOR="true"

# Quiet mode
export DOTENV_QUIET="true"

# Verbose output
export DOTENV_VERBOSE="true"
```

### Advanced Settings

```bash
# API endpoint (for self-hosted)
export DOTENV_API_ENDPOINT="https://dotenv.cloudpany.com"

# Config directory
export DOTENV_CONFIG_DIR="~/.config/dotenv"

# Cache directory
export DOTENV_CACHE_DIR="/tmp/dotenv-cache"

# Disable telemetry
export DOTENV_TELEMETRY_DISABLED="true"

# Debug mode
export DOTENV_DEBUG="true"

# HTTP timeout (seconds)
export DOTENV_TIMEOUT="30"

# Proxy settings
export HTTPS_PROXY="http://proxy:8080"
export NO_PROXY="localhost,127.0.0.1"
```

## Command Aliases

Create custom command aliases:

### In Global Config

```json
{
    "aliases": {
        "p": "projects",
        "s": "secrets",
        "e": "environments",
        "prod": "production",
        "dev": "development"
    }
}
```

### Shell Aliases

```bash
# .bashrc or .zshrc
alias denv="dotenv"
alias dpull="dotenv pull"
alias dpush="dotenv push"
alias drun="dotenv run --"
alias dset="dotenv set"
alias dget="dotenv get"

# Project-specific
alias myapp="dotenv use my-app && dotenv pull"
```

## Output Formats

### Table Format (Default)

```bash
dotenv secrets
┌─────────────┬─────────┬──────────────┐
│ KEY         │ VALUE   │ UPDATED      │
├─────────────┼─────────┼──────────────┤
│ API_KEY     │ •••••   │ 2 hours ago  │
│ DATABASE    │ •••••   │ 3 days ago   │
└─────────────┴─────────┴──────────────┘
```

### JSON Format

```bash
dotenv secrets --format json
{
  "API_KEY": "sk_live_abc123",
  "DATABASE": "postgresql://localhost/mydb"
}
```

### YAML Format

```bash
dotenv secrets --format yaml
API_KEY: sk_live_abc123
DATABASE: postgresql://localhost/mydb
```

### Environment Format

```bash
dotenv secrets --format env
API_KEY=sk_live_abc123
DATABASE=postgresql://localhost/mydb
```

### Shell Format

```bash
dotenv secrets --format shell
export API_KEY='sk_live_abc123'
export DATABASE='postgresql://localhost/mydb'
```

### Custom Format

```bash
# Using Go template syntax
dotenv secrets --format '{{range .}}{{.Key}}={{.Value}}{{"\n"}}{{end}}'
```

## Security Configuration

### Keychain Integration

```json
{
    "security": {
        "keychain_enabled": true,
        "keychain_service": "dotenv-cli",
        "keychain_account": "default"
    }
}
```

### Session Management

```json
{
    "security": {
        "session_timeout": 3600,
        "session_renewal": true,
        "require_mfa": true,
        "allowed_ips": ["10.0.0.0/8", "192.168.1.0/24"]
    }
}
```

### Certificate Pinning

```json
{
    "security": {
        "cert_pinning": true,
        "cert_pins": ["sha256/abc123...", "sha256/def456..."]
    }
}
```

## Cache Configuration

### Cache Settings

```json
{
    "cache": {
        "enabled": true,
        "directory": "~/.dotenv/cache",
        "ttl": 300,
        "max_size": "100MB",
        "compression": true
    }
}
```

### Cache Management

```bash
# Clear cache
dotenv cache clear

# Show cache info
dotenv cache info

# Disable cache for command
dotenv secrets --no-cache

# Force cache refresh
dotenv secrets --refresh-cache
```

## Proxy Configuration

### HTTP Proxy

```bash
# Set proxy
export HTTPS_PROXY="http://proxy.company.com:8080"
export HTTP_PROXY="http://proxy.company.com:8080"

# Proxy with auth
export HTTPS_PROXY="http://user:pass@proxy.company.com:8080"

# No proxy for internal
export NO_PROXY="localhost,127.0.0.1,.company.com"
```

### SOCKS Proxy

```bash
# SOCKS5 proxy
export HTTPS_PROXY="socks5://proxy.company.com:1080"

# SOCKS5 with auth
export HTTPS_PROXY="socks5://user:pass@proxy.company.com:1080"
```

## Logging Configuration

### Log Levels

```json
{
    "logging": {
        "level": "info",
        "file": "~/.dotenv/cli.log",
        "max_size": "10MB",
        "max_files": 5,
        "format": "json"
    }
}
```

### Debug Logging

```bash
# Enable debug logging
export DOTENV_LOG_LEVEL="debug"

# Log to file
export DOTENV_LOG_FILE="/tmp/dotenv.log"

# Trace HTTP requests
export DOTENV_TRACE="true"
```

## Hooks and Scripts

### Pre/Post Hooks

```yaml
# .dotenv.yml
hooks:
    pre_pull: |
        echo "Pulling secrets..."
        git stash

    post_pull: |
        echo "Secrets updated!"
        npm run validate-env

    pre_push: |
        npm test

    post_push: |
        git commit -m "Update secrets"
```

### Custom Scripts

```yaml
scripts:
    # Setup development environment
    dev:setup: |
        dotenv env use development
        dotenv pull
        npm install
        npm run db:migrate

    # Deploy to production
    prod:deploy: |
        dotenv env use production
        dotenv pull --output .env.production
        npm run build
        npm run deploy

    # Rotate all secrets
    rotate:all: |
        dotenv secrets list --format json | \
        jq -r '.[] | .key' | \
        xargs -I {} dotenv secrets rotate {}
```

Run scripts:

```bash
dotenv run-script dev:setup
dotenv run-script prod:deploy
```

## Integration Configuration

### Editor Integration

```json
{
    "editor": {
        "command": "code",
        "wait": true,
        "args": ["--wait"]
    }
}
```

### Git Integration

```yaml
# .dotenv.yml
git:
    auto_commit: true
    commit_message: "Update environment configuration"
    branch: "env-updates"

    hooks:
        pre_commit: "dotenv validate"
        post_merge: "dotenv pull"
```

### CI/CD Integration

```yaml
ci:
    provider: "github"

    environments:
        - branch: main
          environment: production

        - branch: develop
          environment: staging

        - branch: "feature/*"
          environment: development
```

## Performance Tuning

### Connection Settings

```json
{
    "performance": {
        "connection_timeout": 10,
        "request_timeout": 30,
        "max_retries": 3,
        "retry_delay": 1000,
        "connection_pool_size": 10
    }
}
```

### Batch Operations

```json
{
    "performance": {
        "batch_size": 50,
        "parallel_operations": 4,
        "rate_limit": 100
    }
}
```

## Troubleshooting Config

### Reset Configuration

```bash
# Reset global config
dotenv config reset

# Reset project config
rm .dotenv.yml

# Clear all data
dotenv reset --all
```

### Validate Configuration

```bash
# Validate project config
dotenv validate

# Check config syntax
dotenv config validate

# Test configuration
dotenv doctor
```

### Common Issues

#### Config Not Loading

```bash
# Check config location
dotenv config path

# Verify syntax
dotenv config validate

# Use debug mode
DOTENV_DEBUG=true dotenv config
```

#### Permission Issues

```bash
# Fix permissions
chmod 600 ~/.dotenv/config.json

# Check ownership
ls -la ~/.dotenv/
```

## Best Practices

1. **Use Project Config**: Keep `.dotenv.yml` in version control
2. **Secure Global Config**: Set appropriate file permissions
3. **Environment Variables**: Use for CI/CD and automation
4. **Regular Backups**: Backup global configuration
5. **Validate Changes**: Test configuration changes

```bash
# Backup configuration
cp ~/.dotenv/config.json ~/.dotenv/config.backup.json

# Test new config
DOTENV_CONFIG_DIR=/tmp/test dotenv config
```

## Next Steps

- [Authentication](./authentication) - Auth configuration
- [Advanced Usage](./advanced-usage) - Power user features
- [Scripting](./scripting) - Automation with CLI
- [Troubleshooting](./troubleshooting) - Common issues
