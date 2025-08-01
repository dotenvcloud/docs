---
title: CLI Troubleshooting
slug: troubleshooting
order: 7
tags: [cli, troubleshooting, debug]
---

# CLI Troubleshooting

This guide helps you diagnose and resolve common issues with the DotEnv CLI.

## Common Issues

### Authentication Problems

#### "Authentication failed" Error

**Symptoms:**

```bash
$ dotenv login
✗ Error: Authentication failed: Invalid credentials
```

**Solutions:**

1. **Clear cached credentials:**

```bash
dotenv logout --clear-cache
dotenv login
```

2. **Check API endpoint:**

```bash
# Verify endpoint
dotenv config get api_endpoint

# Reset to default
dotenv config set api_endpoint "https://api.dotenv.cloud"
```

3. **Try alternative auth method:**

```bash
# API key authentication
dotenv login --token your-api-key

# Skip browser
dotenv login --no-browser
```

4. **Debug authentication:**

```bash
DOTENV_DEBUG=true dotenv login
```

#### "Session expired" Error

**Problem:** Your authentication session has timed out.

**Solution:**

```bash
# Refresh session
dotenv refresh-session

# Or login again
dotenv login --force

# Extend session timeout
dotenv config set session_timeout 28800  # 8 hours
```

#### "MFA required" But Can't Enter Code

**Problem:** Multi-factor authentication is required but the prompt doesn't appear.

**Solution:**

```bash
# Use backup codes
dotenv login --backup-code YOUR-BACKUP-CODE

# Disable MFA temporarily (if allowed)
dotenv login --skip-mfa

# Use non-interactive mode
echo "your-mfa-code" | dotenv login --mfa-stdin
```

### Network Issues

#### Connection Timeouts

**Symptoms:**

```bash
$ dotenv secrets list
✗ Error: Request timeout after 30s
```

**Solutions:**

1. **Increase timeout:**

```bash
# Temporary
dotenv secrets list --timeout 60

# Permanent
dotenv config set request_timeout 60
```

2. **Check connectivity:**

```bash
# Test connection
dotenv doctor --network

# Test specific endpoint
curl -I https://api.dotenv.cloud/health
```

3. **Use proxy:**

```bash
# HTTP proxy
export HTTPS_PROXY="http://proxy.company.com:8080"
export NO_PROXY="localhost,127.0.0.1"

# SOCKS proxy
export HTTPS_PROXY="socks5://proxy.company.com:1080"
```

#### SSL/TLS Errors

**Symptoms:**

```bash
✗ Error: SSL certificate problem: unable to verify the first certificate
```

**Solutions:**

1. **Update CA certificates:**

```bash
# macOS
brew install ca-certificates

# Linux
sudo update-ca-certificates

# Windows
# Update via Windows Update
```

2. **Specify CA bundle:**

```bash
export SSL_CERT_FILE=/path/to/ca-bundle.crt
dotenv secrets list
```

3. **Corporate proxy with self-signed cert:**

```bash
# Add corporate CA
export NODE_EXTRA_CA_CERTS=/path/to/corporate-ca.crt
```

#### Rate Limiting

**Symptoms:**

```bash
✗ Error: Rate limit exceeded. Try again in 60 seconds
```

**Solutions:**

1. **Check rate limit status:**

```bash
dotenv api rate-limit
```

2. **Use caching:**

```bash
# Enable cache
dotenv config set cache.enabled true

# Increase cache TTL
dotenv config set cache.ttl 600  # 10 minutes
```

3. **Batch operations:**

```bash
# Instead of multiple calls
dotenv set KEY1=value1
dotenv set KEY2=value2

# Use batch
dotenv set KEY1=value1 KEY2=value2
```

### Project/Environment Issues

#### "Project not found" Error

**Symptoms:**

```bash
$ dotenv secrets list
✗ Error: Project 'my-app' not found
```

**Solutions:**

1. **List available projects:**

```bash
dotenv projects list
```

2. **Set correct project:**

```bash
# Interactive selection
dotenv use

# Specific project
dotenv use correct-project-name
```

3. **Check organization:**

```bash
# List organizations
dotenv orgs list

# Switch organization
dotenv orgs use correct-org
```

#### "No active environment" Error

**Problem:** No environment is selected for the current operation.

**Solution:**

```bash
# List environments
dotenv env list

# Set environment
dotenv env use development

# Set default
dotenv config set default_environment development
```

#### Environment Variables Not Loading

**Problem:** Secrets aren't available in your application.

**Solutions:**

1. **Verify pull succeeded:**

```bash
# Check .env file
cat .env

# Pull with verbose output
dotenv pull -v
```

2. **Check format:**

```bash
# Ensure correct format
dotenv pull --format env

# Verify no quotes issues
dotenv pull --no-quotes
```

3. **Run command correctly:**

```bash
# Wrong - runs after shell processes .env
dotenv pull && node app.js

# Correct - dotenv loads vars
dotenv run -- node app.js
```

### Secret Management Issues

#### Can't Set Secret Value

**Symptoms:**

```bash
$ dotenv set API_KEY="value"
✗ Error: Failed to set secret
```

**Solutions:**

1. **Check permissions:**

```bash
# Verify permissions
dotenv whoami --show-permissions

# Check project role
dotenv members list --project my-app
```

2. **Validate secret name:**

```bash
# Check naming rules
# - Start with letter
# - Only letters, numbers, underscores
# - No spaces or special characters

# Valid
dotenv set API_KEY="value"
dotenv set DATABASE_URL_2="value"

# Invalid
dotenv set "API KEY"="value"  # Space
dotenv set "2FA_CODE"="value"  # Starts with number
```

3. **Check value size:**

```bash
# Check if value too large
echo -n "$SECRET_VALUE" | wc -c

# Use file for large values
dotenv set CERTIFICATE --from-file cert.pem
```

#### Secrets Not Updating

**Problem:** Changes to secrets aren't reflected.

**Solutions:**

1. **Clear cache:**

```bash
# Clear CLI cache
dotenv cache clear

# Force fresh pull
dotenv pull --no-cache
```

2. **Check environment:**

```bash
# Verify correct environment
dotenv env current

# Pull from specific environment
dotenv pull --environment production
```

3. **Verify push succeeded:**

```bash
# Push with confirmation
dotenv push --verbose

# Check specific secret
dotenv get MY_SECRET --show-metadata
```

### CLI Installation Issues

#### "Command not found" After Installation

**Problem:** CLI isn't in PATH after installation.

**Solutions:**

1. **Add to PATH manually:**

```bash
# Find installation location
which dotenv || find / -name dotenv 2>/dev/null

# Add to PATH (.bashrc/.zshrc)
export PATH="$HOME/.dotenv/bin:$PATH"

# Reload shell
source ~/.bashrc
```

2. **Reinstall with correct method:**

```bash
# Homebrew (macOS)
brew uninstall dotenv-cli
brew install dotenv/tap/dotenv-cli

# npm global
npm uninstall -g @dotenv/cli
npm install -g @dotenv/cli
```

#### Permission Denied Errors

**Problem:** Can't execute CLI or access config files.

**Solutions:**

1. **Fix executable permissions:**

```bash
chmod +x $(which dotenv)
```

2. **Fix config directory:**

```bash
# Fix ownership
sudo chown -R $(whoami) ~/.dotenv

# Fix permissions
chmod -R 700 ~/.dotenv
```

3. **Install in user directory:**

```bash
# Install locally
curl -sSL https://dotenv.cloud/install.sh | sh -s -- --prefix="$HOME/.local"
export PATH="$HOME/.local/bin:$PATH"
```

### Performance Issues

#### Slow Command Execution

**Problem:** Commands take too long to execute.

**Solutions:**

1. **Enable caching:**

```bash
dotenv config set cache.enabled true
dotenv config set cache.ttl 300
```

2. **Use offline mode when possible:**

```bash
# List cached data
dotenv projects list --offline
dotenv secrets list --offline
```

3. **Profile performance:**

```bash
# Time commands
time dotenv secrets list

# Enable profiling
DOTENV_PROFILE=true dotenv secrets list
```

#### High Memory Usage

**Problem:** CLI consuming too much memory.

**Solutions:**

1. **Limit output size:**

```bash
# Paginate results
dotenv secrets list --limit 50 --page 1

# Export in chunks
dotenv export --chunk-size 100
```

2. **Clear cache:**

```bash
dotenv cache clear
dotenv config set cache.max_size "50MB"
```

## Debugging Techniques

### Enable Debug Mode

Get detailed output for troubleshooting:

```bash
# Basic debug
DOTENV_DEBUG=true dotenv secrets list

# Verbose logging
DOTENV_LOG_LEVEL=debug dotenv pull

# Trace HTTP requests
DOTENV_TRACE=true dotenv login

# All debugging
DOTENV_DEBUG=true DOTENV_TRACE=true DOTENV_LOG_LEVEL=trace dotenv doctor
```

### Use Doctor Command

Comprehensive system check:

```bash
# Full diagnostic
dotenv doctor

# Specific checks
dotenv doctor --network
dotenv doctor --auth
dotenv doctor --config

# Auto-fix issues
dotenv doctor --fix
```

### Check Logs

Find detailed error information:

```bash
# View CLI logs
tail -f ~/.dotenv/logs/cli.log

# Enable file logging
dotenv config set logging.file ~/.dotenv/debug.log
dotenv config set logging.level debug

# View structured logs
dotenv logs --format json | jq '.[] | select(.level=="error")'
```

### Version Information

Check compatibility:

```bash
# CLI version
dotenv --version

# Detailed version info
dotenv version --detailed

# Check for updates
dotenv update --check

# API version
dotenv api version
```

## Platform-Specific Issues

### macOS Issues

#### Keychain Access Denied

**Solution:**

```bash
# Reset keychain access
security delete-generic-password -s "dotenv-cli" 2>/dev/null

# Grant access manually
security add-generic-password -s "dotenv-cli" -a "dotenv" -w "temp"
security delete-generic-password -s "dotenv-cli"
```

#### Homebrew Installation Issues

**Solution:**

```bash
# Clean reinstall
brew uninstall --force dotenv-cli
brew cleanup
brew install dotenv/tap/dotenv-cli

# Link manually if needed
brew link --overwrite dotenv-cli
```

### Windows Issues

#### PowerShell Execution Policy

**Solution:**

```powershell
# Check policy
Get-ExecutionPolicy

# Allow scripts
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser

# Run installer
iwr -useb https://dotenv.cloud/install.ps1 | iex
```

#### Path Length Limitations

**Solution:**

```batch
# Use short paths
dotenv config set cache.directory C:\tmp\dotenv

# Enable long path support (Windows 10+)
# Run as Administrator:
reg add HKLM\SYSTEM\CurrentControlSet\Control\FileSystem /v LongPathsEnabled /t REG_DWORD /d 1
```

### Linux Issues

#### Snap Installation Restrictions

**Solution:**

```bash
# Use classic confinement
sudo snap install dotenv-cli --classic

# Or use alternative installation
curl -sSL https://dotenv.cloud/install.sh | sh
```

#### SELinux Conflicts

**Solution:**

```bash
# Check SELinux status
getenforce

# Add exception
sudo semanage fcontext -a -t bin_t "$HOME/.dotenv/bin/dotenv"
sudo restorecon -v "$HOME/.dotenv/bin/dotenv"
```

## Getting Help

### Self-Service Resources

1. **Built-in help:**

```bash
dotenv --help
dotenv secrets --help
dotenv help secrets
```

2. **Man pages:**

```bash
man dotenv
```

3. **Online documentation:**

```bash
dotenv docs
dotenv docs --search "authentication"
```

### Community Support

1. **GitHub Issues:**

    - Search existing issues: https://github.com/dotenv/cli/issues
    - Create new issue with debug info

2. **Discord Community:**

    - Join: https://discord.gg/dotenv
    - #cli-help channel

3. **Stack Overflow:**
    - Tag: `dotenv-cli`
    - Include version and OS info

### Reporting Bugs

When reporting issues, include:

```bash
# Generate bug report
dotenv doctor --report > bug-report.txt

# Manual collection
echo "=== System Info ===" > report.txt
uname -a >> report.txt
echo -e "\n=== CLI Version ===" >> report.txt
dotenv --version >> report.txt
echo -e "\n=== Config ===" >> report.txt
dotenv config >> report.txt
echo -e "\n=== Debug Output ===" >> report.txt
DOTENV_DEBUG=true dotenv [failing-command] >> report.txt 2>&1
```

### Enterprise Support

For enterprise customers:

```bash
# Open support ticket
dotenv support create \
  --title "Issue with secret rotation" \
  --priority high \
  --attach debug.log

# Check ticket status
dotenv support status TICKET-123

# Live chat
dotenv support chat
```

## Preventive Measures

### Regular Maintenance

```bash
# Weekly tasks
dotenv update --check
dotenv doctor
dotenv cache clear

# Monthly tasks
dotenv audit review
dotenv tokens list --check-expiry
dotenv keys check-rotation
```

### Monitoring Script

```bash
#!/bin/bash
# monitor-dotenv.sh

# Check CLI health
if ! dotenv doctor --quiet; then
  echo "DotEnv CLI health check failed"
  # Send alert
fi

# Check authentication
if ! dotenv whoami >/dev/null 2>&1; then
  echo "DotEnv authentication check failed"
  # Re-authenticate or alert
fi

# Check rate limits
RATE_LIMIT=$(dotenv api rate-limit --format json | jq '.remaining')
if [ "$RATE_LIMIT" -lt 100 ]; then
  echo "Low API rate limit: $RATE_LIMIT"
  # Adjust usage or alert
fi
```

## Next Steps

- [Advanced Usage](./advanced-usage) - Power user features
- [Configuration](./configuration) - Detailed configuration
- [Security Guide](/documentation/v1/security/cli-security) - Security best practices
- [FAQ](/documentation/v1/faq) - Frequently asked questions
