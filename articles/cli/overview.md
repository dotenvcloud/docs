---
title: CLI Overview
slug: overview
order: 1
tags: [cli, overview, introduction]
---

# DotEnv CLI Overview

The DotEnv CLI is a powerful command-line tool for managing environment variables and secrets across your development workflow. It provides secure access to your secrets while maintaining a developer-friendly experience.

## Key Features

- 🔐 **Secure by Design**: All secrets encrypted with AES-256-GCM
- 🚀 **Developer Friendly**: Intuitive commands and helpful output
- 🌍 **Cross-Platform**: Works on Windows, macOS, and Linux
- 🔄 **Real-time Sync**: Always get the latest secrets
- 📦 **Zero Dependencies**: Single binary, no runtime required
- 🎯 **Smart Defaults**: Works out of the box with sensible defaults

## Installation

### Quick Install

#### macOS/Linux

```bash
curl -sSL https://dotenv.cloud/install.sh | sh
```

#### Windows

```powershell
iwr -useb https://dotenv.cloud/install.ps1 | iex
```

#### Package Managers

```bash
# Homebrew (macOS)
brew install dotenv/tap/dotenv-cli

# npm
npm install -g @dotenv/cli

# yarn
yarn global add @dotenv/cli
```

### Verify Installation

```bash
dotenv --version
# Output: dotenv version 1.0.0
```

## Getting Started

### 1. Authentication

First, authenticate with your DotEnv account:

```bash
dotenv login
```

This opens your browser for secure authentication. Once complete, you're ready to use the CLI.

### 2. Basic Workflow

```bash
# Select a project
dotenv use my-app

# List secrets
dotenv secrets

# Set a secret
dotenv set DATABASE_URL="postgresql://localhost/mydb"

# Pull secrets to .env file
dotenv pull

# Run command with secrets
dotenv run -- npm start
```

## Command Structure

DotEnv CLI follows a consistent command structure:

```bash
dotenv [resource] [action] [arguments] [flags]
```

Examples:

```bash
dotenv secrets list --project my-app
dotenv projects create new-app
dotenv environments clone staging production
```

## Core Commands

### Authentication

```bash
dotenv login              # Interactive login
dotenv logout             # Logout
dotenv whoami            # Show current user
```

### Projects

```bash
dotenv projects list     # List all projects
dotenv projects create   # Create new project
dotenv use <project>     # Set active project
```

### Secrets

```bash
dotenv secrets           # List secrets (alias for 'list')
dotenv set KEY=value     # Set a secret
dotenv get KEY           # Get a secret value
dotenv delete KEY        # Delete a secret
dotenv pull              # Download secrets to .env
dotenv push              # Upload .env to DotEnv
```

### Environments

```bash
dotenv env               # Show current environment
dotenv env list          # List environments
dotenv env use <env>     # Switch environment
```

### Running Commands

```bash
dotenv run -- command    # Run command with secrets
dotenv exec -- command   # Alias for run
```

## Configuration

### Global Configuration

The CLI stores configuration in `~/.dotenv/config.json`:

```json
{
    "organization": "acme-corp",
    "default_project": "my-app",
    "default_environment": "development",
    "api_endpoint": "https://api.dotenv.cloud",
    "color_output": true,
    "format": "table"
}
```

### Project Configuration

Create `.dotenv.yml` in your project root:

```yaml
# .dotenv.yml
project: my-app
environment: development
required:
    - DATABASE_URL
    - API_KEY
    - REDIS_URL
optional:
    - SENTRY_DSN
    - ANALYTICS_KEY
```

### Environment Variables

Configure CLI behavior with environment variables:

```bash
DOTENV_API_KEY=your-key          # API authentication
DOTENV_PROJECT=my-app           # Default project
DOTENV_ENVIRONMENT=production   # Default environment
DOTENV_CONFIG_DIR=~/.config     # Config directory
DOTENV_NO_COLOR=true           # Disable colors
DOTENV_FORMAT=json             # Output format
```

## Output Formats

The CLI supports multiple output formats:

### Table (Default)

```bash
dotenv secrets
┌─────────────────┬─────────┬──────────────┐
│ KEY             │ VALUE   │ UPDATED      │
├─────────────────┼─────────┼──────────────┤
│ DATABASE_URL    │ •••••   │ 2 hours ago  │
│ API_KEY         │ •••••   │ 3 days ago   │
│ REDIS_URL       │ •••••   │ 1 week ago   │
└─────────────────┴─────────┴──────────────┘
```

### JSON

```bash
dotenv secrets --format json
{
  "DATABASE_URL": "postgresql://...",
  "API_KEY": "sk_live_...",
  "REDIS_URL": "redis://..."
}
```

### Shell

```bash
dotenv secrets --format shell
export DATABASE_URL="postgresql://..."
export API_KEY="sk_live_..."
export REDIS_URL="redis://..."
```

### .env

```bash
dotenv secrets --format env
DATABASE_URL=postgresql://...
API_KEY=sk_live_...
REDIS_URL=redis://...
```

## Interactive Mode

Many commands support interactive mode for better UX:

```bash
# Interactive project selection
dotenv use
? Select a project:
  ▸ api-service
    web-app
    mobile-app

# Interactive secret setting
dotenv set --interactive
? Enter key: API_KEY
? Enter value (hidden): ****
✓ Secret 'API_KEY' set successfully

# Interactive environment selection
dotenv env use
? Select environment:
    development
  ▸ staging
    production
```

## Error Handling

The CLI provides clear error messages and suggestions:

```bash
$ dotenv get MISSING_KEY
✗ Error: Secret 'MISSING_KEY' not found

Did you mean one of these?
  - MISSING_API_KEY
  - MESSAGING_KEY

Run 'dotenv secrets' to see all available secrets.
```

## Shell Completion

Enable tab completion for your shell:

### Bash

```bash
echo 'eval "$(dotenv completion bash)"' >> ~/.bashrc
source ~/.bashrc
```

### Zsh

```bash
echo 'eval "$(dotenv completion zsh)"' >> ~/.zshrc
source ~/.zshrc
```

### Fish

```bash
dotenv completion fish > ~/.config/fish/completions/dotenv.fish
```

### PowerShell

```powershell
dotenv completion powershell | Out-String | Invoke-Expression
```

## Debugging

Enable debug mode for troubleshooting:

```bash
# Verbose output
dotenv secrets -v

# Debug mode
DOTENV_DEBUG=true dotenv secrets

# Trace HTTP requests
DOTENV_TRACE=true dotenv pull
```

## Offline Mode

Some commands work offline using cached data:

```bash
# List cached secrets
dotenv secrets --offline

# Use cached project list
dotenv projects list --offline

# Force online mode
dotenv secrets --no-cache
```

## CI/CD Integration

The CLI works great in CI/CD pipelines:

```bash
#!/bin/bash
# ci-deploy.sh

# Use API key authentication
export DOTENV_API_KEY=$CI_DOTENV_API_KEY

# Pull production secrets
dotenv pull --project my-app --environment production

# Run tests with secrets
dotenv run -- npm test

# Deploy application
dotenv run -- ./deploy.sh
```

## Docker Integration

Use the CLI in Docker containers:

```dockerfile
FROM node:18-alpine

# Install DotEnv CLI
RUN apk add --no-cache curl && \
    curl -sSL https://dotenv.cloud/install.sh | sh

# Copy application
WORKDIR /app
COPY . .

# Run with secrets
CMD ["dotenv", "run", "--", "node", "server.js"]
```

## Security Features

### Secure Storage

- Credentials stored in OS keychain (macOS/Windows)
- Encrypted file storage on Linux
- No plaintext secrets on disk

### Session Management

- Automatic session timeout
- Explicit logout command
- Revocable API keys

### Audit Trail

- All CLI actions logged
- Available in audit logs
- Track who accessed what

## Performance

The CLI is optimized for speed:

- **Binary size**: ~15MB
- **Startup time**: <50ms
- **API calls**: Cached and batched
- **Memory usage**: <20MB typical

## Troubleshooting

### Common Issues

#### Authentication Failed

```bash
# Clear credentials and re-login
dotenv logout
dotenv login
```

#### Project Not Found

```bash
# List available projects
dotenv projects list

# Set project explicitly
dotenv use my-app
```

#### Network Issues

```bash
# Check connectivity
dotenv doctor

# Use proxy
export HTTPS_PROXY=http://proxy:8080
dotenv pull
```

### Getting Help

```bash
# General help
dotenv --help

# Command-specific help
dotenv secrets --help

# Version info
dotenv --version

# System diagnostics
dotenv doctor
```

## Best Practices

1. **Use Project Config**: Create `.dotenv.yml` for project settings
2. **Don't Commit Secrets**: Add `.env` to `.gitignore`
3. **Use Interactive Mode**: For sensitive values
4. **Enable Completion**: For better productivity
5. **Regular Updates**: Keep CLI updated

```bash
# Check for updates
dotenv update

# Auto-update
dotenv config set auto_update true
```

## Next Steps

- [Command Reference](./commands) - Detailed command documentation
- [Configuration](./configuration) - Advanced configuration options
- [Authentication](./authentication) - Auth methods and security
- [Advanced Usage](./advanced-usage) - Power user features
- [Troubleshooting](./troubleshooting) - Common issues and solutions
