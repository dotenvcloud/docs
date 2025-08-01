---
title: Error Reference
slug: errors
order: 10
tags: [troubleshooting, errors, reference]
---

# Error Reference

Complete reference of error codes and messages in the DotEnv platform.

## Error Code Format

DotEnv uses structured error codes in the format: `COMPONENT_ERROR_TYPE`

- **COMPONENT**: The system component (AUTH, API, ENC, etc.)
- **ERROR**: Literal "ERROR"
- **TYPE**: Specific error condition

## Authentication Errors

### AUTH_ERROR_INVALID_CREDENTIALS
**Message**: "Invalid email or password"
**Cause**: Incorrect login credentials
**Solution**: Verify email and password are correct

### AUTH_ERROR_TOKEN_EXPIRED
**Message**: "Authentication token has expired"
**Cause**: API token or session has expired
**Solution**: Re-authenticate or generate a new API token

### AUTH_ERROR_MFA_REQUIRED
**Message**: "Multi-factor authentication required"
**Cause**: Account requires MFA but code not provided
**Solution**: Provide valid MFA code with authentication request

### AUTH_ERROR_ACCOUNT_LOCKED
**Message**: "Account has been locked due to multiple failed attempts"
**Cause**: Too many failed login attempts
**Solution**: Wait 30 minutes or contact support

## API Errors

### API_ERROR_RATE_LIMIT
**Message**: "Rate limit exceeded"
**HTTP Status**: 429
**Cause**: Too many API requests in time window
**Solution**: Implement exponential backoff, check rate limit headers

### API_ERROR_INVALID_REQUEST
**Message**: "Invalid request format"
**HTTP Status**: 400
**Cause**: Malformed request body or parameters
**Solution**: Check API documentation for correct format

### API_ERROR_NOT_FOUND
**Message**: "Resource not found"
**HTTP Status**: 404
**Cause**: Requested resource doesn't exist
**Solution**: Verify resource ID and permissions

### API_ERROR_FORBIDDEN
**Message**: "Access forbidden"
**HTTP Status**: 403
**Cause**: Insufficient permissions
**Solution**: Check API token permissions and organization access

## Encryption Errors

### ENC_ERROR_INVALID_KEY
**Message**: "Invalid encryption key format"
**Cause**: Encryption key is malformed or wrong length
**Solution**: Ensure key is 32 bytes and properly base64 encoded

### ENC_ERROR_DECRYPTION_FAILED
**Message**: "Failed to decrypt secret"
**Cause**: Wrong key, corrupted data, or tampered ciphertext
**Solution**: Verify correct encryption key and data integrity

### ENC_ERROR_KEY_ROTATION_FAILED
**Message**: "Key rotation failed"
**Cause**: Unable to re-encrypt secrets with new key
**Solution**: Check permissions and retry operation

## Organization Errors

### ORG_ERROR_QUOTA_EXCEEDED
**Message**: "Organization quota exceeded"
**Cause**: Exceeded plan limits (projects, secrets, users)
**Solution**: Upgrade plan or remove unused resources

### ORG_ERROR_SUSPENDED
**Message**: "Organization has been suspended"
**Cause**: Billing issue or policy violation
**Solution**: Contact support or update billing information

### ORG_ERROR_INVALID_DOMAIN
**Message**: "Email domain not allowed for this organization"
**Cause**: Email domain restrictions are enabled
**Solution**: Use approved email domain or contact admin

## CLI Errors

### CLI_ERROR_NOT_INITIALIZED
**Message**: "Project not initialized"
**Cause**: No .dotenv file in current directory
**Solution**: Run `dotenv init` to initialize project

### CLI_ERROR_SYNC_FAILED
**Message**: "Failed to sync with remote"
**Cause**: Network issue or authentication problem
**Solution**: Check internet connection and re-authenticate

### CLI_ERROR_INVALID_COMMAND
**Message**: "Unknown command"
**Cause**: Typed command doesn't exist
**Solution**: Run `dotenv --help` for available commands

### CLI_ERROR_VERSION_MISMATCH
**Message**: "CLI version incompatible with API"
**Cause**: CLI version too old or too new
**Solution**: Update CLI to latest version

## Project Errors

### PROJ_ERROR_DUPLICATE_KEY
**Message**: "Secret key already exists"
**Cause**: Attempting to create secret with existing key
**Solution**: Use update command or choose different key

### PROJ_ERROR_INVALID_CHARACTERS
**Message**: "Invalid characters in secret key"
**Cause**: Key contains unsupported characters
**Solution**: Use only A-Z, 0-9, and underscore

### PROJ_ERROR_LOCKED
**Message**: "Project is locked"
**Cause**: Project locked for maintenance or security
**Solution**: Wait for unlock or contact project admin

## Network Errors

### NET_ERROR_CONNECTION_REFUSED
**Message**: "Connection refused"
**Cause**: API endpoint unreachable
**Solution**: Check network settings and firewall rules

### NET_ERROR_TIMEOUT
**Message**: "Request timeout"
**Cause**: Response took too long
**Solution**: Retry request or check network latency

### NET_ERROR_DNS_FAILED
**Message**: "DNS resolution failed"
**Cause**: Cannot resolve api.dotenv.cloud
**Solution**: Check DNS settings and internet connection

## SDK Errors

### SDK_ERROR_NOT_CONFIGURED
**Message**: "SDK not properly configured"
**Cause**: Missing API key or configuration
**Solution**: Initialize SDK with valid credentials

### SDK_ERROR_INVALID_RESPONSE
**Message**: "Invalid API response"
**Cause**: Unexpected response format
**Solution**: Update SDK to latest version

### SDK_ERROR_UNSUPPORTED_OPERATION
**Message**: "Operation not supported"
**Cause**: Feature not available in SDK version
**Solution**: Update SDK or use API directly

## Common Solutions

### Check Status Page
Visit https://status.dotenv.cloud for service status

### Verify API Key
```bash
dotenv auth:check
```

### Update CLI
```bash
# macOS/Linux
curl -fsSL https://dotenv.cloud/install.sh | sh

# Windows
iwr https://dotenv.cloud/install.ps1 -useb | iex
```

### Enable Debug Mode
```bash
# CLI
dotenv --debug [command]

# SDK
const client = new DotEnv({ debug: true })
```

### Contact Support
- Email: support@dotenv.cloud
- Include error code and full error message
- Provide request ID if available