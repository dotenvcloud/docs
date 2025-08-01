---
title: Common Issues
slug: common-issues
order: 20
tags: [troubleshooting, issues, solutions]
---

# Common Issues

Solutions to frequently encountered problems with DotEnv.

## Installation Issues

### CLI Installation Fails on macOS

**Problem**: Installation script fails with permission errors

**Solution**:
```bash
# Use sudo if installing to system directories
curl -fsSL https://dotenv.cloud/install.sh | sudo sh

# Or install to user directory
curl -fsSL https://dotenv.cloud/install.sh | sh -s -- --install-dir=$HOME/.local/bin
```

### CLI Installation Fails on Windows

**Problem**: PowerShell execution policy blocks installation

**Solution**:
```powershell
# Temporarily bypass execution policy
powershell -ExecutionPolicy Bypass -c "iwr https://dotenv.cloud/install.ps1 -useb | iex"

# Or permanently allow scripts
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

### CLI Not Found After Installation

**Problem**: `dotenv: command not found`

**Solution**:
```bash
# Add to PATH in ~/.bashrc or ~/.zshrc
export PATH="$HOME/.local/bin:$PATH"

# Reload shell configuration
source ~/.bashrc  # or ~/.zshrc
```

## Authentication Issues

### Cannot Login with CLI

**Problem**: Login fails even with correct credentials

**Possible Causes**:
1. MFA is enabled but code not provided
2. Account locked due to failed attempts
3. Network blocking API access

**Solutions**:
```bash
# Login with MFA
dotenv auth:login --mfa

# Check network connectivity
curl -I https://api.dotenv.cloud/health

# Use API token instead
dotenv auth:login --token YOUR_API_TOKEN
```

### API Token Not Working

**Problem**: "Invalid API token" error

**Solutions**:
1. Verify token hasn't expired
2. Check token has required permissions
3. Ensure using correct organization

```bash
# Verify token
dotenv auth:check

# List token permissions
dotenv auth:tokens:list
```

## Sync Issues

### Secrets Not Syncing

**Problem**: Local changes not appearing in remote

**Solutions**:
```bash
# Force sync
dotenv sync --force

# Check sync status
dotenv status

# Verify project connection
dotenv projects:info
```

### Merge Conflicts

**Problem**: "Conflict detected" during sync

**Solution**:
```bash
# Pull remote changes first
dotenv pull

# Resolve conflicts manually in .env file

# Push resolved changes
dotenv push --force
```

## Encryption Issues

### Cannot Decrypt Secrets

**Problem**: "Decryption failed" errors

**Common Causes**:
1. Using wrong encryption key
2. Key format is incorrect
3. Secret was encrypted with different key

**Solutions**:
```bash
# Verify key format (32 bytes, base64)
echo -n "your-key" | base64 -d | wc -c  # Should output 32

# Re-encrypt with correct key
dotenv secrets:rotate-keys

# Use server-managed encryption
dotenv projects:update --encryption-mode server
```

### Client-Side Encryption Not Working

**Problem**: Secrets visible in plaintext on server

**Solution**:
```bash
# Enable client-side encryption
dotenv projects:update --encryption-mode client

# Provide encryption key
export DOTENV_ENCRYPTION_KEY="your-base64-key"

# Re-encrypt existing secrets
dotenv secrets:rotate-keys
```

## Performance Issues

### Slow API Responses

**Problem**: Commands take long time to complete

**Solutions**:
1. Check network latency to API
2. Use pagination for large datasets
3. Enable caching

```bash
# Test API latency
time curl -I https://api.dotenv.cloud/health

# Use pagination
dotenv secrets:list --limit 50 --page 1

# Enable local caching
dotenv config:set cache.enabled true
```

### High Memory Usage

**Problem**: CLI using excessive memory

**Solutions**:
```bash
# Limit concurrent operations
dotenv config:set concurrency 1

# Clear local cache
dotenv cache:clear

# Use streaming for large files
dotenv export --stream > .env
```

## Integration Issues

### Docker Container Can't Access Secrets

**Problem**: Environment variables not available in container

**Solution**:
```dockerfile
# Option 1: Build-time secrets
ARG DOTENV_TOKEN
RUN --mount=type=secret,id=dotenv_token \
    dotenv export --token $(cat /run/secrets/dotenv_token) > .env

# Option 2: Runtime injection
FROM dotenv/cli as secrets
ARG DOTENV_TOKEN
RUN dotenv export --token $DOTENV_TOKEN > /secrets/.env

FROM your-app
COPY --from=secrets /secrets/.env .env
```

### CI/CD Pipeline Failures

**Problem**: Secrets not available in CI environment

**Solutions**:

**GitHub Actions**:
```yaml
- name: Load secrets
  run: |
    curl -fsSL https://dotenv.cloud/install.sh | sh
    dotenv export --token ${{ secrets.DOTENV_TOKEN }} > .env
```

**GitLab CI**:
```yaml
before_script:
  - curl -fsSL https://dotenv.cloud/install.sh | sh
  - dotenv export --token $DOTENV_TOKEN > .env
```

### SDK Initialization Errors

**Problem**: SDK throws initialization errors

**JavaScript/TypeScript**:
```javascript
// Ensure environment variables are loaded
import { DotEnv } from '@dotenv/sdk';

const client = new DotEnv({
  apiKey: process.env.DOTENV_API_KEY,
  // Add retry logic
  retry: {
    maxRetries: 3,
    retryDelay: 1000,
  },
});
```

**PHP**:
```php
use DotEnv\SDK\Client;

// Check for required extensions
if (!extension_loaded('curl')) {
    throw new Exception('CURL extension required');
}

$client = new Client([
    'apiKey' => $_ENV['DOTENV_API_KEY'],
    'timeout' => 30,
]);
```

## Organization Issues

### Cannot Access Organization

**Problem**: "Forbidden" errors when accessing organization

**Solutions**:
1. Verify organization membership
2. Check if organization is suspended
3. Ensure using correct API token

```bash
# List accessible organizations
dotenv orgs:list

# Switch organization context
dotenv config:set organization YOUR_ORG_ID
```

### Hitting Usage Limits

**Problem**: "Quota exceeded" errors

**Solutions**:
```bash
# Check current usage
dotenv orgs:usage

# Clean up unused resources
dotenv projects:list --unused
dotenv secrets:cleanup --dry-run
```

## Debug Mode

Enable debug mode for detailed error information:

```bash
# CLI debug mode
export DOTENV_DEBUG=true
dotenv [command]

# Or inline
dotenv --debug [command]
```

## Getting Help

If issues persist:

1. Check service status: https://status.dotenv.cloud
2. Search documentation: https://docs.dotenv.cloud
3. Contact support: support@dotenv.cloud

Include in support requests:
- Error messages and codes
- CLI version: `dotenv --version`
- Operating system and version
- Steps to reproduce issue
- Debug output if available