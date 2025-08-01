---
title: Environment Variables
slug: environment-variables
order: 8
tags: [cli, environment, variables]
---

# Environment Variables

The DotEnv CLI can be configured using environment variables. This guide covers all available options and their usage.

## Configuration Variables

### Authentication

Control CLI authentication behavior:

```bash
# API Key - Highest priority authentication method
export DOTENV_API_KEY="dotenv_api_prod_abc123xyz"

# OAuth Token - Alternative to API key
export DOTENV_TOKEN="eyJ0eXAiOiJKV1QiLCJhbGc..."

# Session Token - For session persistence
export DOTENV_SESSION_TOKEN="session_abc123..."

# Disable interactive authentication
export DOTENV_NON_INTERACTIVE=true

# Skip browser opening during login
export DOTENV_NO_BROWSER=true
```

### Project Configuration

Set default project and organization:

```bash
# Default project
export DOTENV_PROJECT="my-app"

# Default organization
export DOTENV_ORGANIZATION="acme-corp"

# Default environment
export DOTENV_ENVIRONMENT="development"

# Project path (for auto-detection)
export DOTENV_PROJECT_PATH="/path/to/project"
```

### API Configuration

Configure API endpoints and behavior:

```bash
# API Endpoint (for self-hosted or regional endpoints)
export DOTENV_API_ENDPOINT="https://api.dotenv.cloud"

# API Version
export DOTENV_API_VERSION="v1"

# Request timeout (seconds)
export DOTENV_TIMEOUT="30"

# Retry configuration
export DOTENV_MAX_RETRIES="3"
export DOTENV_RETRY_DELAY="1000"  # milliseconds
```

### Output Configuration

Control CLI output format and appearance:

```bash
# Output format (table|json|yaml|env|shell)
export DOTENV_FORMAT="json"

# Disable colored output
export DOTENV_NO_COLOR="true"
export NO_COLOR="true"  # Standard convention

# Quiet mode (suppress non-error output)
export DOTENV_QUIET="true"

# Verbose output
export DOTENV_VERBOSE="true"

# Machine-readable output
export DOTENV_MACHINE_READABLE="true"
```

### Debugging

Enable debugging and diagnostics:

```bash
# General debug mode
export DOTENV_DEBUG="true"

# Log level (trace|debug|info|warn|error)
export DOTENV_LOG_LEVEL="debug"

# HTTP request tracing
export DOTENV_TRACE="true"

# Performance profiling
export DOTENV_PROFILE="true"

# Debug specific components
export DOTENV_DEBUG_AUTH="true"
export DOTENV_DEBUG_CACHE="true"
export DOTENV_DEBUG_CRYPTO="true"
```

### Security

Configure security-related settings:

```bash
# Require MFA for all operations
export DOTENV_REQUIRE_MFA="true"

# Certificate pinning
export DOTENV_CERT_PINS="sha256/abc123...,sha256/def456..."

# Disable certificate verification (NOT RECOMMENDED)
export DOTENV_INSECURE="true"

# Custom CA certificate
export DOTENV_CA_CERT="/path/to/ca.crt"
export NODE_EXTRA_CA_CERTS="/path/to/ca.crt"

# Client certificate authentication
export DOTENV_CLIENT_CERT="/path/to/client.crt"
export DOTENV_CLIENT_KEY="/path/to/client.key"
```

### Cache Configuration

Control caching behavior:

```bash
# Disable cache entirely
export DOTENV_NO_CACHE="true"

# Cache directory
export DOTENV_CACHE_DIR="~/.cache/dotenv"

# Cache TTL (seconds)
export DOTENV_CACHE_TTL="300"

# Maximum cache size
export DOTENV_CACHE_MAX_SIZE="100MB"

# Force cache refresh
export DOTENV_REFRESH_CACHE="true"
```

### File Locations

Override default file locations:

```bash
# Config directory
export DOTENV_CONFIG_DIR="~/.config/dotenv"

# Config file path
export DOTENV_CONFIG_FILE="~/.dotenv/config.json"

# Project config file
export DOTENV_PROJECT_CONFIG=".dotenv.yml"

# Credentials storage
export DOTENV_CREDENTIALS_PATH="~/.dotenv/credentials"

# Log file location
export DOTENV_LOG_FILE="/var/log/dotenv.log"
```

## Network Configuration

### Proxy Settings

Configure proxy servers:

```bash
# HTTP/HTTPS proxy
export HTTP_PROXY="http://proxy.company.com:8080"
export HTTPS_PROXY="http://proxy.company.com:8080"
export http_proxy="http://proxy.company.com:8080"
export https_proxy="http://proxy.company.com:8080"

# Proxy with authentication
export HTTPS_PROXY="http://user:pass@proxy.company.com:8080"

# SOCKS proxy
export HTTPS_PROXY="socks5://proxy.company.com:1080"

# No proxy for specific domains
export NO_PROXY="localhost,127.0.0.1,.company.com,192.168.0.0/16"
export no_proxy="localhost,127.0.0.1,.company.com,192.168.0.0/16"

# Proxy for specific protocols
export FTP_PROXY="http://proxy.company.com:8080"
export ALL_PROXY="http://proxy.company.com:8080"
```

### DNS Configuration

Custom DNS resolution:

```bash
# Custom DNS servers
export DOTENV_DNS_SERVERS="8.8.8.8,8.8.4.4"

# DNS timeout
export DOTENV_DNS_TIMEOUT="5"

# Use system DNS
export DOTENV_USE_SYSTEM_DNS="true"
```

## Behavioral Variables

### Update Behavior

Control auto-update functionality:

```bash
# Disable auto-update checks
export DOTENV_NO_UPDATE_CHECK="true"

# Auto-update without prompting
export DOTENV_AUTO_UPDATE="true"

# Update channel (stable|beta|nightly)
export DOTENV_UPDATE_CHANNEL="stable"

# Update check interval (hours)
export DOTENV_UPDATE_CHECK_INTERVAL="24"
```

### Telemetry

Control telemetry and analytics:

```bash
# Disable all telemetry
export DOTENV_TELEMETRY_DISABLED="true"
export DO_NOT_TRACK="1"

# Telemetry endpoint
export DOTENV_TELEMETRY_ENDPOINT="https://telemetry.dotenv.cloud"

# Anonymous usage stats only
export DOTENV_TELEMETRY_ANONYMOUS="true"
```

### Editor Integration

Configure editor behavior:

```bash
# Default editor
export EDITOR="vim"
export VISUAL="code"

# Editor for config files
export DOTENV_EDITOR="code --wait"

# Editor arguments
export DOTENV_EDITOR_ARGS="--new-window"
```

## Platform-Specific Variables

### macOS

macOS-specific settings:

```bash
# Keychain usage
export DOTENV_KEYCHAIN_ENABLED="true"
export DOTENV_KEYCHAIN_SERVICE="dotenv-cli"
export DOTENV_KEYCHAIN_ACCOUNT="default"

# macOS notification center
export DOTENV_NOTIFICATIONS="true"
```

### Windows

Windows-specific settings:

```bash
# Windows credential store
export DOTENV_CREDENTIAL_STORE="wincred"

# PowerShell integration
export DOTENV_POWERSHELL_MODULE="true"

# Windows registry usage
export DOTENV_USE_REGISTRY="false"
```

### Linux

Linux-specific settings:

```bash
# Secret service provider
export DOTENV_SECRET_SERVICE="gnome-keyring"
export DOTENV_SECRET_SERVICE="kwallet"

# D-Bus configuration
export DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/1000/bus"

# Use kernel keyring
export DOTENV_USE_KERNEL_KEYRING="true"
```

## Special Purpose Variables

### CI/CD Detection

Automatically detected CI environments:

```bash
# Generic CI indicator
export CI="true"
export CONTINUOUS_INTEGRATION="true"

# GitHub Actions
export GITHUB_ACTIONS="true"
export GITHUB_TOKEN="..."

# GitLab CI
export GITLAB_CI="true"
export CI_PROJECT_ID="..."

# Jenkins
export JENKINS_HOME="/var/jenkins"
export BUILD_ID="..."

# CircleCI
export CIRCLECI="true"
export CIRCLE_BUILD_NUM="..."
```

### Container Environments

Container-specific settings:

```bash
# Docker detection
export DOCKER_CONTAINER="true"

# Kubernetes pod
export KUBERNETES_SERVICE_HOST="..."
export KUBERNETES_SERVICE_PORT="..."

# Container runtime
export CONTAINER_RUNTIME="docker"
```

### Development Mode

Development-specific settings:

```bash
# Development mode
export DOTENV_DEV_MODE="true"
export NODE_ENV="development"

# Local development server
export DOTENV_LOCAL_API="http://localhost:3000"

# Mock mode
export DOTENV_MOCK_MODE="true"

# Fixture data
export DOTENV_USE_FIXTURES="true"
```

## Usage Examples

### Development Workflow

```bash
# .env.local
export DOTENV_PROJECT="my-app"
export DOTENV_ENVIRONMENT="development"
export DOTENV_FORMAT="table"
export DOTENV_NO_UPDATE_CHECK="true"

# Source before work
source .env.local
dotenv pull  # Uses configured defaults
```

### CI/CD Pipeline

```bash
# GitHub Actions
env:
  DOTENV_API_KEY: ${{ secrets.DOTENV_API_KEY }}
  DOTENV_PROJECT: "my-app"
  DOTENV_ENVIRONMENT: ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}
  DOTENV_NON_INTERACTIVE: "true"
  DOTENV_NO_COLOR: "true"
  DOTENV_QUIET: "true"
```

### Docker Container

```dockerfile
FROM node:18

# Set DotEnv environment
ENV DOTENV_API_KEY=${DOTENV_API_KEY}
ENV DOTENV_PROJECT="my-app"
ENV DOTENV_ENVIRONMENT="production"
ENV DOTENV_NO_CACHE="true"
ENV DOTENV_NON_INTERACTIVE="true"

# Install CLI
RUN curl -sSL https://dotenv.cloud/install.sh | sh

# Use in container
RUN dotenv pull
CMD ["dotenv", "run", "--", "node", "server.js"]
```

### Security-Hardened Setup

```bash
# High-security environment
export DOTENV_REQUIRE_MFA="true"
export DOTENV_CERT_PINS="sha256/abc123..."
export DOTENV_TELEMETRY_DISABLED="true"
export DOTENV_AUDIT_ALL="true"
export DOTENV_SESSION_TIMEOUT="900"  # 15 minutes
export DOTENV_IP_WHITELIST="10.0.0.0/8"
```

## Precedence Order

Environment variables follow this precedence (highest to lowest):

1. Command-line flags
2. Environment variables
3. Project config file (.dotenv.yml)
4. Global config file (~/.dotenv/config.json)
5. Default values

Example:

```bash
# Default: format=table
# Config file: format=json
# Environment: DOTENV_FORMAT=yaml
# Command: --format env

# Result: format=env (command flag wins)
```

## Best Practices

### Security

1. **Never commit environment variables**

    ```bash
    # .gitignore
    .env*
    !.env.example
    ```

2. **Use specific scopes**

    ```bash
    # Too broad
    export DOTENV_API_KEY="key-with-all-permissions"

    # Better
    export DOTENV_READ_ONLY_KEY="key-with-read-permission"
    ```

3. **Rotate regularly**
    ```bash
    # Set expiry reminder
    export DOTENV_KEY_EXPIRES="2024-12-31"
    ```

### Organization

1. **Use .envrc for project-specific vars**

    ```bash
    # .envrc (with direnv)
    export DOTENV_PROJECT="my-app"
    export DOTENV_ENVIRONMENT="development"
    ```

2. **Document your variables**

    ```bash
    # .env.example
    # DotEnv CLI Configuration
    DOTENV_PROJECT=your-project-name
    DOTENV_ENVIRONMENT=development
    DOTENV_FORMAT=json  # Options: table|json|yaml|env
    ```

3. **Group related variables**

    ```bash
    # Authentication
    export DOTENV_API_KEY="..."

    # Project Settings
    export DOTENV_PROJECT="..."
    export DOTENV_ENVIRONMENT="..."

    # Output Preferences
    export DOTENV_FORMAT="..."
    export DOTENV_NO_COLOR="..."
    ```

### Debugging

Create debug profiles:

```bash
# debug.env
export DOTENV_DEBUG="true"
export DOTENV_TRACE="true"
export DOTENV_LOG_LEVEL="trace"
export DOTENV_LOG_FILE="/tmp/dotenv-debug.log"

# Usage
source debug.env
dotenv secrets list
```

## Next Steps

- [Configuration](./configuration) - File-based configuration
- [Advanced Usage](./advanced-usage) - Power user features
- [Scripting](./scripting) - Using environment variables in scripts
- [CI/CD Usage](./ci-cd-usage) - CI/CD-specific patterns
