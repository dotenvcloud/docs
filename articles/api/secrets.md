---
title: Secrets API
slug: secrets
order: 5
tags: [api, secrets, encryption]
---

# Secrets API

The Secrets API provides secure management of sensitive data with encryption, versioning, and access control. All secret values are encrypted using AES-256-GCM encryption.

## Secret Object

```json
{
    "id": "sec_abc123def456",
    "key": "DATABASE_URL",
    "value": "encrypted_value_base64...",
    "project_id": "proj_xyz789",
    "environment": "production",
    "description": "PostgreSQL database connection string",
    "tags": ["database", "critical"],
    "version": 3,
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-15T10:30:00Z",
    "created_by": {
        "id": "usr_123456",
        "email": "john@example.com"
    },
    "updated_by": {
        "id": "usr_123456",
        "email": "john@example.com"
    },
    "metadata": {
        "rotation_period": 90,
        "last_rotated": "2024-01-01T00:00:00Z",
        "expires_at": "2024-04-01T00:00:00Z",
        "references": ["deployment.yaml", "docker-compose.yml"]
    },
    "encrypted": true,
    "client_encrypted": false
}
```

## Endpoints

### List Secrets

Get all secrets in a project/environment:

```http
GET /projects/{project_id}/secrets
GET /projects/{project_id}/environments/{environment}/secrets
```

Query parameters:

- `environment` (string): Filter by environment
- `search` (string): Search in keys and descriptions
- `tag` (string): Filter by tag
- `page` (integer): Page number
- `per_page` (integer): Items per page (max 100)
- `include_values` (boolean): Include decrypted values (default: false)
- `version` (string): Get specific version (latest, all)

Example request:

```bash
curl -X GET https://api.dotenv.cloud/v1/projects/proj_xyz789/secrets?environment=production \
  -H "Authorization: Bearer YOUR_API_KEY"
```

Example response:

```json
{
    "data": [
        {
            "id": "sec_abc123",
            "key": "DATABASE_URL",
            "project_id": "proj_xyz789",
            "environment": "production",
            "description": "PostgreSQL connection",
            "tags": ["database"],
            "version": 3,
            "updated_at": "2024-01-15T10:30:00Z",
            "metadata": {
                "rotation_period": 90
            }
        }
    ],
    "pagination": {
        "current_page": 1,
        "per_page": 25,
        "total": 45
    }
}
```

### Get Secret

Get a specific secret:

```http
GET /secrets/{secret_id}
GET /projects/{project_id}/secrets/{key}
```

Query parameters:

- `include_value` (boolean): Include decrypted value
- `version` (integer): Get specific version

Example request:

```bash
curl -X GET https://api.dotenv.cloud/v1/secrets/sec_abc123?include_value=true \
  -H "Authorization: Bearer YOUR_API_KEY"
```

Example response:

```json
{
    "data": {
        "id": "sec_abc123",
        "key": "DATABASE_URL",
        "value": "postgresql://user:pass@host:5432/db",
        "environment": "production",
        "version": 3,
        "created_at": "2024-01-01T00:00:00Z",
        "updated_at": "2024-01-15T10:30:00Z"
    }
}
```

### Create Secret

Create a new secret:

```http
POST /projects/{project_id}/secrets
```

Request body:

```json
{
    "key": "NEW_API_KEY",
    "value": "sk_live_abc123xyz",
    "environment": "production",
    "description": "Stripe API key",
    "tags": ["payment", "critical"],
    "encrypt_client_side": false,
    "metadata": {
        "rotation_period": 30,
        "expires_at": "2024-06-01T00:00:00Z"
    }
}
```

Example request:

```bash
curl -X POST https://api.dotenv.cloud/v1/projects/proj_xyz789/secrets \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "STRIPE_API_KEY",
    "value": "sk_live_abc123",
    "environment": "production"
  }'
```

### Update Secret

Update an existing secret:

```http
PUT /secrets/{secret_id}
PATCH /secrets/{secret_id}
PUT /projects/{project_id}/secrets/{key}
```

Request body:

```json
{
    "value": "new_secret_value",
    "description": "Updated description",
    "tags": ["updated", "tags"],
    "metadata": {
        "reason": "Regular rotation"
    }
}
```

### Delete Secret

Delete a secret:

```http
DELETE /secrets/{secret_id}
DELETE /projects/{project_id}/secrets/{key}
```

Query parameters:

- `environment` (string): Required when using key-based deletion

Example request:

```bash
curl -X DELETE https://api.dotenv.cloud/v1/projects/proj_xyz789/secrets/OLD_KEY?environment=production \
  -H "Authorization: Bearer YOUR_API_KEY"
```

## Bulk Operations

### Bulk Create/Update

Create or update multiple secrets:

```http
POST /projects/{project_id}/secrets/bulk
```

Request body:

```json
{
    "environment": "production",
    "secrets": [
        {
            "key": "API_KEY",
            "value": "new_value_1",
            "description": "API key"
        },
        {
            "key": "DATABASE_URL",
            "value": "new_value_2",
            "description": "Database connection"
        }
    ],
    "options": {
        "overwrite": true,
        "validate": true
    }
}
```

### Bulk Delete

Delete multiple secrets:

```http
DELETE /projects/{project_id}/secrets/bulk
```

Request body:

```json
{
    "environment": "development",
    "keys": ["OLD_KEY_1", "OLD_KEY_2", "OLD_KEY_3"]
}
```

### Bulk Retrieve

Retrieve multiple specific secrets:

```http
POST /projects/{project_id}/secrets/retrieve
```

Request body:

```json
{
    "environment": "production",
    "keys": ["API_KEY", "DATABASE_URL", "REDIS_URL"],
    "include_values": true
}
```

## Secret Versions

### List Versions

Get version history:

```http
GET /secrets/{secret_id}/versions
```

Example response:

```json
{
    "data": [
        {
            "version": 3,
            "value": "encrypted_value_v3",
            "created_at": "2024-01-15T10:30:00Z",
            "created_by": {
                "id": "usr_123456",
                "email": "john@example.com"
            },
            "metadata": {
                "reason": "Regular rotation"
            }
        },
        {
            "version": 2,
            "value": "encrypted_value_v2",
            "created_at": "2024-01-01T00:00:00Z"
        }
    ]
}
```

### Restore Version

Restore a previous version:

```http
POST /secrets/{secret_id}/versions/{version}/restore
```

## Secret Rotation

### Rotate Secret

Rotate a secret value:

```http
POST /secrets/{secret_id}/rotate
```

Request body:

```json
{
    "strategy": "generate",
    "options": {
        "length": 32,
        "include_symbols": true
    },
    "grace_period": "24h",
    "notify": ["team@example.com"]
}
```

### Bulk Rotation

Rotate multiple secrets:

```http
POST /projects/{project_id}/secrets/rotate
```

Request body:

```json
{
    "environment": "production",
    "filter": {
        "tag": "database",
        "older_than": "90d"
    },
    "strategy": "generate"
}
```

## Encryption

### Client-Side Encryption

For zero-knowledge encryption:

```http
POST /projects/{project_id}/secrets
```

Request body:

```json
{
    "key": "SENSITIVE_DATA",
    "value": "base64_encrypted_value",
    "environment": "production",
    "encrypted": true,
    "encryption": {
        "algorithm": "aes-256-gcm",
        "key_id": "key_abc123",
        "iv": "base64_iv",
        "tag": "base64_tag"
    }
}
```

### Encryption Keys

Get project encryption keys:

```http
GET /projects/{project_id}/encryption-keys
```

Example response:

```json
{
    "data": [
        {
            "id": "key_abc123",
            "algorithm": "aes-256-gcm",
            "public_key": "base64_public_key",
            "created_at": "2024-01-01T00:00:00Z",
            "is_active": true,
            "client_managed": false
        }
    ]
}
```

### Re-encrypt Secrets

Re-encrypt with new key:

```http
POST /projects/{project_id}/secrets/reencrypt
```

Request body:

```json
{
    "new_key_id": "key_def456",
    "environments": ["production", "staging"]
}
```

## Import/Export

### Export Secrets

Export secrets in various formats:

```http
GET /projects/{project_id}/secrets/export
```

Query parameters:

- `environment` (string): Environment to export
- `format` (string): Export format (env, json, yaml, toml)
- `include_descriptions` (boolean): Include descriptions
- `mask_values` (boolean): Mask sensitive values

Example response (.env format):

```
# Database Configuration
DATABASE_URL=postgresql://user:pass@host:5432/db

# API Keys
STRIPE_API_KEY=sk_live_***
SENDGRID_API_KEY=SG.***

# Redis Configuration
REDIS_URL=redis://localhost:6379
```

### Import Secrets

Import from file:

```http
POST /projects/{project_id}/secrets/import
```

Request body:

```json
{
    "environment": "development",
    "format": "env",
    "content": "DATABASE_URL=postgresql://...\nAPI_KEY=sk_test_...",
    "options": {
        "overwrite": false,
        "validate": true
    }
}
```

## Search and Filter

### Search Secrets

Full-text search across secrets:

```http
GET /projects/{project_id}/secrets/search
```

Query parameters:

- `q` (string): Search query
- `environment` (string): Filter by environment
- `include_values` (boolean): Search in values (requires permission)
- `include_descriptions` (boolean): Search in descriptions

### Filter by Metadata

Filter secrets by metadata:

```http
GET /projects/{project_id}/secrets?metadata[expires_at][lte]=2024-12-31
```

## Validation

### Validate Secrets

Validate secret values:

```http
POST /projects/{project_id}/secrets/validate
```

Request body:

```json
{
    "environment": "production",
    "validations": [
        {
            "key": "DATABASE_URL",
            "type": "postgres_uri"
        },
        {
            "key": "EMAIL",
            "type": "email"
        },
        {
            "key": "PORT",
            "type": "integer",
            "min": 1,
            "max": 65535
        }
    ]
}
```

### Connection Testing

Test secret connections:

```http
POST /projects/{project_id}/secrets/test
```

Request body:

```json
{
    "environment": "production",
    "tests": [
        {
            "key": "DATABASE_URL",
            "type": "database"
        },
        {
            "key": "REDIS_URL",
            "type": "redis"
        }
    ]
}
```

## References

### Find References

Find where secrets are referenced:

```http
GET /secrets/{secret_id}/references
```

Example response:

```json
{
    "data": {
        "internal": [
            {
                "type": "environment",
                "id": "env_staging",
                "name": "staging",
                "inherits": true
            }
        ],
        "external": [
            {
                "file": "docker-compose.yml",
                "line": 15,
                "context": "DATABASE_URL: ${DATABASE_URL}"
            }
        ]
    }
}
```

## Webhooks

Secret-specific webhook events:

- `secret.created`
- `secret.updated`
- `secret.deleted`
- `secret.rotated`
- `secret.accessed`
- `secret.validation_failed`

Example webhook payload:

```json
{
    "event": "secret.updated",
    "timestamp": "2024-01-15T10:30:00Z",
    "data": {
        "secret": {
            "id": "sec_abc123",
            "key": "DATABASE_URL",
            "environment": "production",
            "project_id": "proj_xyz789"
        },
        "changes": {
            "value": true,
            "description": true
        },
        "actor": {
            "id": "usr_123456",
            "email": "john@example.com"
        }
    }
}
```

## Rate Limits

Secret endpoints have specific rate limits:

| Endpoint             | Rate Limit |
| -------------------- | ---------- |
| List secrets         | 1000/hour  |
| Get secret value     | 5000/hour  |
| Create/Update secret | 500/hour   |
| Delete secret        | 100/hour   |
| Bulk operations      | 100/hour   |

## Error Responses

### 404 Not Found

```json
{
    "error": {
        "code": "SECRET_NOT_FOUND",
        "message": "Secret not found",
        "details": {
            "key": "MISSING_KEY",
            "environment": "production"
        }
    }
}
```

### 409 Conflict

```json
{
    "error": {
        "code": "SECRET_EXISTS",
        "message": "Secret already exists",
        "details": {
            "key": "DATABASE_URL",
            "environment": "production",
            "suggestion": "Use PUT to update existing secret"
        }
    }
}
```

### 422 Validation Error

```json
{
    "error": {
        "code": "INVALID_SECRET_VALUE",
        "message": "Secret value validation failed",
        "details": {
            "key": "DATABASE_URL",
            "validation": "Invalid PostgreSQL connection string"
        }
    }
}
```

## Best Practices

### Security

1. **Use client-side encryption** for highly sensitive data
2. **Rotate secrets regularly** - Set rotation reminders
3. **Limit access** - Use environment-specific permissions
4. **Audit access** - Monitor who accesses secrets
5. **Never log values** - Only log secret keys, not values

### Performance

1. **Batch operations** - Use bulk endpoints for multiple secrets
2. **Cache carefully** - Secrets can change, use short TTLs
3. **Use webhooks** - Instead of polling for changes
4. **Paginate results** - Don't fetch all secrets at once

### Organization

1. **Use consistent naming** - Establish naming conventions
2. **Tag secrets** - Makes filtering and management easier
3. **Document secrets** - Use descriptions field
4. **Group related secrets** - Use prefixes (DB*, API*, AWS\_)

## SDK Examples

### JavaScript

```javascript
const dotenv = new DotEnvAPI({ apiKey: "YOUR_API_KEY" });

// Get secret
const secret = await dotenv.secrets.get("DATABASE_URL", {
    project: "proj_xyz789",
    environment: "production",
    includeValue: true,
});

// Bulk update
await dotenv.secrets.bulkUpdate("proj_xyz789", {
    environment: "staging",
    secrets: [
        { key: "API_KEY", value: "new_key" },
        { key: "API_SECRET", value: "new_secret" },
    ],
});

// Rotate secret
await dotenv.secrets.rotate("sec_abc123", {
    strategy: "generate",
    notifyEmail: "team@example.com",
});
```

### Python

```python
from dotenv_api import DotEnvAPI

client = DotEnvAPI(api_key='YOUR_API_KEY')

# List secrets
secrets = client.secrets.list(
    project_id='proj_xyz789',
    environment='production',
    include_values=False
)

# Create secret with client encryption
encrypted_value = client.crypto.encrypt('sensitive_data')
client.secrets.create(
    project_id='proj_xyz789',
    key='SENSITIVE_KEY',
    value=encrypted_value,
    encrypted=True
)

# Export secrets
env_file = client.secrets.export(
    project_id='proj_xyz789',
    environment='development',
    format='env'
)
```

## Next Steps

- [Encryption Guide](/documentation/v1/security/encryption) - Encryption details
- [Secret Management](/documentation/v1/core-concepts/secret-management) - Best practices
- [Webhooks](/documentation/v1/drafts/api/webhooks) *(Coming Soon)* - Real-time updates
- [Rate Limiting](./rate-limiting) - API limits
