---
title: CLI Authentication
slug: authentication
order: 4
tags: [cli, authentication, security]
---

# CLI Authentication

The DotEnv CLI supports multiple authentication methods to accommodate different workflows and security requirements. This guide covers all authentication options and best practices.

## Authentication Methods

### Browser Authentication (Default)

The recommended authentication method for interactive use:

```bash
# Start authentication flow
dotenv login

# Opens browser automatically
# Complete OAuth flow
# CLI receives token automatically
```

**Process:**

1. CLI starts local server on port 8080
2. Opens browser to DotEnv login page
3. You authenticate via OAuth (Google, GitHub, etc.)
4. Browser redirects back to CLI
5. CLI stores credentials securely

**Options:**

```bash
# Use specific port
dotenv login --port 9090

# Don't open browser automatically
dotenv login --no-browser

# Use specific OAuth provider
dotenv login --provider github
```

### API Key Authentication

Best for automation and CI/CD:

```bash
# Using command flag
dotenv login --token dotenv_api_prod_abc123xyz

# Using environment variable
export DOTENV_API_KEY="dotenv_api_prod_abc123xyz"
dotenv projects list

# Using config file
echo '{"api_key": "dotenv_api_prod_abc123xyz"}' > ~/.dotenv/config.json
```

**API Key Types:**

- **Personal**: Tied to your user account
- **Service**: For automated systems
- **Project**: Limited to specific project
- **Organization**: Full org access

### SSO Authentication

For enterprise users with Single Sign-On:

```bash
# SSO login
dotenv login --sso

# With organization hint
dotenv login --sso --org acme-corp

# Direct SSO URL
dotenv login --sso-url https://acme.dotenv.cloud/sso
```

**Supported Providers:**

- Okta
- Azure AD
- Google Workspace
- SAML 2.0
- OpenID Connect

### Multi-Factor Authentication

Additional security layer for sensitive operations:

```bash
# Login with MFA
dotenv login
# Enter username/password
# Enter MFA code: 123456

# Backup codes
dotenv login --backup-code ABCD-EFGH-IJKL

# Hardware token
dotenv login --use-hardware-token
```

## Credential Storage

### macOS Keychain

On macOS, credentials are stored in the system keychain:

```bash
# View stored credentials
security find-generic-password -s "dotenv-cli"

# Remove credentials
security delete-generic-password -s "dotenv-cli"

# Disable keychain storage
export DOTENV_KEYCHAIN_DISABLED=true
```

### Windows Credential Manager

On Windows, credentials use the Windows Credential Manager:

```powershell
# View credentials
cmdkey /list:dotenv*

# Remove credentials
cmdkey /delete:dotenv-cli

# Use alternative storage
$env:DOTENV_CREDENTIAL_STORE = "file"
```

### Linux Secret Service

On Linux, credentials use the Secret Service API:

```bash
# Using GNOME Keyring
secret-tool search application dotenv-cli

# Using KWallet
kwallet-query kdewallet -r dotenv-cli

# Fallback to encrypted file
export DOTENV_USE_FILE_STORE=true
```

### Encrypted File Storage

Fallback option for all platforms:

```bash
# Force file storage
export DOTENV_CREDENTIAL_STORE=file

# Custom location
export DOTENV_CREDENTIALS_PATH=~/.config/dotenv/creds

# File location
~/.dotenv/credentials.enc
```

## Session Management

### Session Lifecycle

Sessions expire after a period of inactivity:

```bash
# Default session timeout: 8 hours
# View session info
dotenv whoami --session-info

# Extend session
dotenv refresh-session

# Set custom timeout
dotenv config set session_timeout 3600  # 1 hour
```

### Multiple Sessions

Manage multiple authentication contexts:

```bash
# List all sessions
dotenv sessions list

# Switch session
dotenv sessions use work-account

# Create named session
dotenv login --session personal-projects

# Remove session
dotenv sessions delete old-session
```

### Session Security

Best practices for session management:

```bash
# Require re-authentication for sensitive operations
dotenv config set require_fresh_auth true

# Lock session to IP
dotenv config set session_ip_lock true

# Automatic logout on idle
dotenv config set auto_logout_minutes 30
```

## Environment Variables

Configure authentication via environment:

### Authentication Variables

```bash
# API Key (highest priority)
export DOTENV_API_KEY="dotenv_api_prod_abc123"

# OAuth Token
export DOTENV_TOKEN="eyJ0eXAiOiJKV1QiLCJhbGc..."

# Session Token
export DOTENV_SESSION_TOKEN="session_abc123"

# Disable interactive auth
export DOTENV_NON_INTERACTIVE=true
```

### Security Variables

```bash
# Require MFA
export DOTENV_REQUIRE_MFA=true

# Certificate pinning
export DOTENV_CERT_PINS="sha256/abc123..."

# Custom auth endpoint
export DOTENV_AUTH_ENDPOINT="https://auth.company.com"

# Proxy authentication
export HTTPS_PROXY="http://user:pass@proxy:8080"
```

## Team Authentication

### Service Accounts

Create service accounts for automation:

```bash
# Create service account
dotenv service-accounts create \
  --name "ci-bot" \
  --description "CI/CD automation" \
  --expires 365d

# Output:
# Service Account: ci-bot
# API Key: dotenv_api_svc_xyz789...
# Expires: 2025-01-17

# Rotate service account key
dotenv service-accounts rotate ci-bot
```

### Delegated Authentication

Allow team members to authenticate on behalf of service:

```bash
# Setup delegation
dotenv auth delegate \
  --service "production-deployer" \
  --users "alice@example.com,bob@example.com" \
  --permissions "read,deploy"

# Use delegated auth
dotenv login --delegate production-deployer
```

### Role-Based Access

Configure role-based authentication:

```bash
# Require specific role
dotenv login --require-role admin

# Check current roles
dotenv whoami --show-roles

# Switch role context
dotenv auth use-role developer
```

## CI/CD Authentication

### GitHub Actions

```yaml
- name: Authenticate DotEnv CLI
  env:
      DOTENV_API_KEY: ${{ secrets.DOTENV_API_KEY }}
  run: |
      # API key from env automatically used
      dotenv projects list
```

### Jenkins

```groovy
pipeline {
  environment {
    DOTENV_API_KEY = credentials('dotenv-api-key')
  }
  stages {
    stage('Deploy') {
      steps {
        sh 'dotenv secrets pull --environment production'
      }
    }
  }
}
```

### GitLab CI

```yaml
deploy:
    variables:
        DOTENV_API_KEY: $DOTENV_API_KEY
    script:
        - dotenv secrets pull
        - ./deploy.sh
```

### CircleCI

```yaml
jobs:
    deploy:
        environment:
            DOTENV_API_KEY: << context.dotenv-api-key >>
        steps:
            - run: dotenv secrets pull
```

## Security Best Practices

### 1. Use Appropriate Method

```bash
# Development: Browser auth
dotenv login

# CI/CD: API keys
export DOTENV_API_KEY=$CI_SECRET

# Production: Service accounts
dotenv login --token $SERVICE_ACCOUNT_KEY
```

### 2. Rotate Credentials

```bash
# Rotate API key
dotenv auth rotate-key

# Schedule rotation
dotenv config set key_rotation_days 90

# Check key age
dotenv auth key-info
```

### 3. Limit Scope

```bash
# Create project-scoped key
dotenv api-keys create \
  --scope project:my-app \
  --permissions read,list

# Organization read-only key
dotenv api-keys create \
  --scope org:acme \
  --permissions read \
  --name "monitoring"
```

### 4. Monitor Usage

```bash
# View authentication logs
dotenv audit auth --days 7

# List active sessions
dotenv sessions list --active

# Revoke suspicious session
dotenv sessions revoke session_xyz
```

## Troubleshooting

### Common Issues

#### Authentication Failed

```bash
# Clear cached credentials
dotenv logout --clear-cache

# Try alternative method
dotenv login --method token

# Debug mode
DOTENV_DEBUG=true dotenv login
```

#### Session Expired

```bash
# Refresh session
dotenv refresh-session

# Force new login
dotenv login --force

# Extend timeout
dotenv config set session_timeout 28800  # 8 hours
```

#### MFA Issues

```bash
# Use backup code
dotenv login --backup-code XXXX-XXXX

# Skip MFA (if allowed)
dotenv login --skip-mfa

# Reset MFA
dotenv auth reset-mfa
```

#### Proxy Authentication

```bash
# Basic proxy auth
export HTTPS_PROXY="http://user:pass@proxy:8080"

# NTLM proxy
export HTTPS_PROXY="http://domain\\user:pass@proxy:8080"

# Proxy with cert
export HTTPS_PROXY="http://proxy:8080"
export PROXY_CERT="/path/to/cert.pem"
```

### Debug Authentication

Enable detailed logging:

```bash
# Verbose output
dotenv login -v

# Debug mode
export DOTENV_DEBUG=true
export DOTENV_LOG_LEVEL=debug
dotenv login

# Trace HTTP
export DOTENV_TRACE=true
dotenv login
```

### Reset Authentication

Complete authentication reset:

```bash
# 1. Logout everywhere
dotenv logout --all

# 2. Clear local data
rm -rf ~/.dotenv/credentials*
rm -rf ~/.dotenv/sessions*

# 3. Clear keychain (macOS)
security delete-generic-password -s "dotenv-cli"

# 4. Fresh login
dotenv login
```

## Advanced Authentication

### Custom OAuth Provider

Configure custom OAuth provider:

```json
{
    "auth": {
        "oauth_provider": "custom",
        "oauth_endpoint": "https://auth.company.com",
        "oauth_client_id": "dotenv-cli",
        "oauth_scopes": ["read", "write"]
    }
}
```

### Hardware Token Support

Use hardware security keys:

```bash
# Configure FIDO2
dotenv config set auth_hardware_token true

# Register key
dotenv auth register-hardware-token

# Login with key
dotenv login --use-hardware-token
```

### Certificate Authentication

Use client certificates:

```bash
# Configure cert auth
dotenv config set \
  auth_client_cert /path/to/cert.pem \
  auth_client_key /path/to/key.pem

# Login with cert
dotenv login --method certificate
```

### Kerberos/SPNEGO

Enterprise Kerberos authentication:

```bash
# Enable Kerberos
export DOTENV_USE_KERBEROS=true
export KRB5_CONFIG=/etc/krb5.conf

# Login with Kerberos ticket
kinit user@REALM
dotenv login --method kerberos
```

## Integration Examples

### Shell Function

```bash
# ~/.bashrc or ~/.zshrc
dotenv-auth() {
  if [[ -z "$DOTENV_API_KEY" ]]; then
    echo "Authenticating with DotEnv..."
    dotenv login
  else
    echo "Already authenticated"
  fi
}

# Auto-auth before commands
alias dotenv='dotenv-auth && command dotenv'
```

### Docker Authentication

```dockerfile
# Multi-stage auth
FROM alpine AS auth
RUN apk add --no-cache curl
ARG DOTENV_API_KEY
RUN curl -H "Authorization: Bearer $DOTENV_API_KEY" \
    https://api.dotenv.cloud/v1/verify

FROM node:18
COPY --from=auth /verification /
```

### Kubernetes Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
    name: dotenv-auth
type: Opaque
stringData:
    api-key: "dotenv_api_prod_abc123"
---
apiVersion: v1
kind: Pod
spec:
    containers:
        - name: app
          env:
              - name: DOTENV_API_KEY
                valueFrom:
                    secretKeyRef:
                        name: dotenv-auth
                        key: api-key
```

## Next Steps

- [Advanced Usage](./advanced-usage) - Power user features
- [Configuration](./configuration) - Detailed configuration options
- [Troubleshooting](./troubleshooting) - Common issues
- [Security Guide](/documentation/v1/security/overview) - Security best practices
