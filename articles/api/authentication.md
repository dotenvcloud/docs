---
title: API Authentication
slug: authentication
order: 2
tags: [api, authentication, security]
---

# API Authentication

The DotEnv API uses bearer token authentication. This guide covers all authentication methods, token management, and security best practices.

## Authentication Methods

### API Keys

API keys are the primary authentication method for server-to-server communication:

```http
Authorization: Bearer dotenv_api_prod_abc123xyz
```

Example request:

```bash
curl -X GET https://api.dotenv.cloud/v1/projects \
  -H "Authorization: Bearer dotenv_api_prod_abc123xyz" \
  -H "Content-Type: application/json"
```

### OAuth 2.0

For user-delegated access and third-party integrations:

```http
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...
```

OAuth flow:

1. Redirect user to authorization URL
2. User approves access
3. Receive authorization code
4. Exchange code for access token
5. Use token for API requests

### Personal Access Tokens

For individual developer use:

```http
Authorization: Bearer dotenv_pat_usr_abc123xyz
```

These tokens:

- Tied to user account
- Inherit user permissions
- Can be revoked by user
- Ideal for CLI tools

## API Key Types

### Organization Keys

Full access to organization resources:

```json
{
    "type": "organization",
    "prefix": "dotenv_api_org_",
    "scope": "organization:*",
    "permissions": ["read", "write", "delete"]
}
```

### Project Keys

Limited to specific projects:

```json
{
    "type": "project",
    "prefix": "dotenv_api_proj_",
    "scope": "project:proj_123",
    "permissions": ["secrets:read", "secrets:write"]
}
```

### Service Account Keys

For automated systems:

```json
{
    "type": "service",
    "prefix": "dotenv_api_svc_",
    "scope": "service:ci-deployer",
    "permissions": ["secrets:read"],
    "expires_at": "2025-01-01T00:00:00Z"
}
```

### Read-Only Keys

Limited to read operations:

```json
{
    "type": "readonly",
    "prefix": "dotenv_api_ro_",
    "scope": "*",
    "permissions": ["read"]
}
```

## Creating API Keys

### Via Dashboard

1. Navigate to Settings → API Keys
2. Click "Create New Key"
3. Configure:
    - Name
    - Type
    - Permissions
    - Expiration
4. Copy key immediately (shown once)

### Via API

```http
POST /api-keys
Authorization: Bearer YOUR_ADMIN_KEY
Content-Type: application/json

{
  "name": "CI/CD Deployment Key",
  "type": "service",
  "permissions": [
    "projects:read",
    "secrets:read",
    "secrets:write"
  ],
  "scope": {
    "projects": ["proj_123", "proj_456"],
    "environments": ["staging", "production"]
  },
  "expires_in": "90d",
  "ip_whitelist": ["10.0.0.0/8"]
}
```

Response:

```json
{
    "data": {
        "id": "key_abc123",
        "name": "CI/CD Deployment Key",
        "key": "dotenv_api_svc_xyz789...",
        "type": "service",
        "created_at": "2024-01-15T10:00:00Z",
        "expires_at": "2024-04-15T10:00:00Z"
    }
}
```

## OAuth 2.0 Flow

### Authorization Request

```http
GET https://auth.dotenv.cloud/oauth/authorize?
  response_type=code&
  client_id=YOUR_CLIENT_ID&
  redirect_uri=https://your-app.com/callback&
  scope=projects:read secrets:read&
  state=random_state_value
```

### Token Exchange

```http
POST https://auth.dotenv.cloud/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=AUTH_CODE&
client_id=YOUR_CLIENT_ID&
client_secret=YOUR_CLIENT_SECRET&
redirect_uri=https://your-app.com/callback
```

Response:

```json
{
    "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
    "token_type": "Bearer",
    "expires_in": 3600,
    "refresh_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
    "scope": "projects:read secrets:read"
}
```

### Token Refresh

```http
POST https://auth.dotenv.cloud/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&
refresh_token=YOUR_REFRESH_TOKEN&
client_id=YOUR_CLIENT_ID&
client_secret=YOUR_CLIENT_SECRET
```

## Token Management

### Listing Keys

```http
GET /api-keys
Authorization: Bearer YOUR_API_KEY
```

Response:

```json
{
    "data": [
        {
            "id": "key_abc123",
            "name": "Production API",
            "type": "organization",
            "last_used": "2024-01-15T09:30:00Z",
            "expires_at": null,
            "status": "active"
        }
    ]
}
```

### Revoking Keys

```http
DELETE /api-keys/{key_id}
Authorization: Bearer YOUR_API_KEY
```

Or revoke by value:

```http
POST /api-keys/revoke
Authorization: Bearer YOUR_API_KEY

{
  "key": "dotenv_api_prod_xyz789..."
}
```

### Key Rotation

```http
POST /api-keys/{key_id}/rotate
Authorization: Bearer YOUR_API_KEY

{
  "grace_period": "24h"
}
```

Response:

```json
{
    "data": {
        "old_key": {
            "id": "key_abc123",
            "expires_at": "2024-01-16T10:00:00Z"
        },
        "new_key": {
            "id": "key_def456",
            "key": "dotenv_api_prod_new789...",
            "created_at": "2024-01-15T10:00:00Z"
        }
    }
}
```

## Security Features

### IP Whitelisting

Restrict key usage by IP:

```http
PATCH /api-keys/{key_id}
Authorization: Bearer YOUR_API_KEY

{
  "ip_whitelist": [
    "192.168.1.0/24",
    "10.0.0.0/8",
    "203.0.113.0/24"
  ]
}
```

### Request Signing

For additional security, sign requests:

```http
Authorization: Bearer YOUR_API_KEY
X-DotEnv-Signature: sha256=HMAC_SIGNATURE
X-DotEnv-Timestamp: 1642089600
```

Signature calculation:

```javascript
const crypto = require("crypto");

const timestamp = Math.floor(Date.now() / 1000);
const payload = `${timestamp}.${method}.${path}.${body}`;
const signature = crypto
    .createHmac("sha256", signingSecret)
    .update(payload)
    .digest("hex");
```

### Multi-Factor Authentication

Require MFA for sensitive operations:

```http
POST /projects/{id}/secrets
Authorization: Bearer YOUR_API_KEY
X-DotEnv-MFA-Code: 123456

{
  "key": "PRODUCTION_SECRET",
  "value": "sensitive_value"
}
```

### Temporary Tokens

Create short-lived tokens:

```http
POST /api-keys/temporary
Authorization: Bearer YOUR_API_KEY

{
  "expires_in": "1h",
  "permissions": ["secrets:read"],
  "scope": {
    "projects": ["proj_123"],
    "environments": ["production"]
  }
}
```

## Permission Scopes

### Resource Scopes

- `*` - All resources
- `organizations:*` - All organizations
- `organizations:read` - Read organizations
- `organizations:write` - Create/update organizations
- `projects:*` - All project operations
- `projects:read` - Read projects
- `projects:write` - Create/update projects
- `projects:delete` - Delete projects
- `secrets:*` - All secret operations
- `secrets:read` - Read secrets
- `secrets:write` - Create/update secrets
- `secrets:delete` - Delete secrets
- `secrets:decrypt` - Decrypt secret values

### Action Scopes

- `audit:read` - View audit logs
- `members:*` - Manage team members
- `api-keys:*` - Manage API keys
- `webhooks:*` - Manage webhooks
- `billing:*` - Manage billing

### Combining Scopes

```json
{
    "permissions": [
        "projects:read",
        "secrets:read",
        "secrets:write",
        "audit:read"
    ],
    "scope": {
        "organizations": ["org_123"],
        "projects": ["proj_456", "proj_789"],
        "environments": ["production"]
    }
}
```

## Authentication Errors

### 401 Unauthorized

Missing or invalid authentication:

```json
{
    "error": {
        "code": "UNAUTHORIZED",
        "message": "Invalid or missing authentication token"
    }
}
```

Common causes:

- Missing `Authorization` header
- Malformed token
- Expired token
- Revoked token

### 403 Forbidden

Valid auth but insufficient permissions:

```json
{
    "error": {
        "code": "FORBIDDEN",
        "message": "You don't have permission to access this resource",
        "details": {
            "required_permission": "secrets:write",
            "your_permissions": ["secrets:read"]
        }
    }
}
```

### Token Expiration

```json
{
    "error": {
        "code": "TOKEN_EXPIRED",
        "message": "Your authentication token has expired",
        "details": {
            "expired_at": "2024-01-15T10:00:00Z"
        }
    }
}
```

## Best Practices

### Key Security

1. **Never commit keys**: Use environment variables
2. **Rotate regularly**: Set expiration dates
3. **Use least privilege**: Minimal permissions
4. **Restrict by IP**: When possible
5. **Monitor usage**: Check audit logs

### Token Storage

```javascript
// Bad - Hardcoded
const apiKey = "dotenv_api_prod_abc123";

// Good - Environment variable
const apiKey = process.env.DOTENV_API_KEY;

// Better - Secure key management
const apiKey = await keyVault.getSecret("dotenv-api-key");
```

### Request Authentication

```javascript
// Example secure client
class DotEnvClient {
    constructor(apiKey, options = {}) {
        this.apiKey = apiKey;
        this.baseURL = options.baseURL || "https://api.dotenv.cloud/v1";
        this.timeout = options.timeout || 30000;
        this.retries = options.retries || 3;
    }

    async request(method, path, data = null) {
        const url = `${this.baseURL}${path}`;
        const headers = {
            Authorization: `Bearer ${this.apiKey}`,
            "Content-Type": "application/json",
            "User-Agent": "DotEnv-Client/1.0",
        };

        try {
            const response = await fetch(url, {
                method,
                headers,
                body: data ? JSON.stringify(data) : null,
                timeout: this.timeout,
            });

            if (!response.ok) {
                throw new Error(`API error: ${response.status}`);
            }

            return await response.json();
        } catch (error) {
            // Implement retry logic
            throw error;
        }
    }
}
```

### Rate Limit Handling

```javascript
async function makeAPICall(client, method, path, data) {
    try {
        return await client.request(method, path, data);
    } catch (error) {
        if (error.status === 429) {
            const retryAfter = error.headers["X-RateLimit-Reset"];
            const waitTime = retryAfter - Date.now() / 1000;
            await sleep(waitTime * 1000);
            return makeAPICall(client, method, path, data);
        }
        throw error;
    }
}
```

## Testing Authentication

### cURL Examples

```bash
# Test authentication
curl -I https://api.dotenv.cloud/v1/projects \
  -H "Authorization: Bearer YOUR_API_KEY"

# Verbose output
curl -v https://api.dotenv.cloud/v1/projects \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### Postman Collection

Import our Postman collection:
https://api.dotenv.cloud/postman-collection.json

Features:

- Pre-configured auth
- Environment variables
- Example requests
- Test scripts

### API Key Validation

```http
POST /auth/validate
Authorization: Bearer YOUR_API_KEY
```

Response:

```json
{
    "data": {
        "valid": true,
        "type": "organization",
        "permissions": ["projects:*", "secrets:*"],
        "expires_at": null,
        "organization": {
            "id": "org_123",
            "name": "ACME Corp"
        }
    }
}
```

## Migration Guide

### From Basic Auth

```bash
# Old (deprecated)
curl -u username:password https://api.dotenv.cloud/v1/projects

# New
curl -H "Authorization: Bearer YOUR_API_KEY" https://api.dotenv.cloud/v1/projects
```

### From API v0

```javascript
// Old
const client = new DotEnv({
    username: "user",
    password: "pass",
});

// New
const client = new DotEnv({
    apiKey: process.env.DOTENV_API_KEY,
});
```

## Next Steps

- [API Overview](./overview) - General API information
- [Organizations API](./organizations) - Organization endpoints
- [Projects API](./projects) - Project management
- [Secrets API](./secrets) - Secret operations
- [Rate Limiting](./rate-limiting) - Rate limit details
- [Error Handling](./error-handling) - Error responses
