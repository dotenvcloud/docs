---
title: Organizations API
slug: organizations
order: 3
tags: [api, organizations, teams]
---

# Organizations API

The Organizations API allows you to manage organizations, teams, and members. Organizations are the top-level container for all resources in DotEnv.

## Organization Object

```json
{
    "id": "org_abc123def456",
    "name": "ACME Corporation",
    "slug": "acme-corp",
    "description": "Leading provider of innovative solutions",
    "logo_url": "https://assets.dotenv.cloud/orgs/acme-corp/logo.png",
    "website": "https://acme-corp.com",
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-15T10:30:00Z",
    "settings": {
        "require_mfa": true,
        "allowed_auth_methods": ["password", "google", "github"],
        "ip_whitelist": ["10.0.0.0/8"],
        "session_timeout": 3600
    },
    "subscription": {
        "plan": "enterprise",
        "status": "active",
        "seats": 50,
        "expires_at": "2025-01-01T00:00:00Z"
    },
    "stats": {
        "projects_count": 12,
        "members_count": 34,
        "secrets_count": 567
    }
}
```

## Endpoints

### List Organizations

Get all organizations the authenticated user has access to:

```http
GET /organizations
```

Query parameters:

- `page` (integer): Page number
- `per_page` (integer): Items per page (max 100)
- `search` (string): Search by name
- `sort` (string): Sort field (name, created_at)
- `order` (string): Sort order (asc, desc)

Example request:

```bash
curl -X GET https://api.dotenv.cloud/v1/organizations?page=1&per_page=25 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

Example response:

```json
{
    "data": [
        {
            "id": "org_abc123def456",
            "name": "ACME Corporation",
            "slug": "acme-corp",
            "role": "owner",
            "created_at": "2024-01-01T00:00:00Z"
        },
        {
            "id": "org_xyz789ghi012",
            "name": "Startup Inc",
            "slug": "startup-inc",
            "role": "member",
            "created_at": "2024-01-10T00:00:00Z"
        }
    ],
    "pagination": {
        "current_page": 1,
        "per_page": 25,
        "total": 2,
        "total_pages": 1
    }
}
```

### Get Organization

Get details of a specific organization:

```http
GET /organizations/{organization_id}
```

Path parameters:

- `organization_id` (string): Organization ID or slug

Example request:

```bash
curl -X GET https://api.dotenv.cloud/v1/organizations/org_abc123def456 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

Example response:

```json
{
    "data": {
        "id": "org_abc123def456",
        "name": "ACME Corporation",
        "slug": "acme-corp",
        "description": "Leading provider of innovative solutions",
        "created_at": "2024-01-01T00:00:00Z",
        "updated_at": "2024-01-15T10:30:00Z",
        "settings": {
            "require_mfa": true,
            "allowed_auth_methods": ["password", "google", "github"]
        },
        "subscription": {
            "plan": "enterprise",
            "status": "active"
        }
    }
}
```

### Create Organization

Create a new organization:

```http
POST /organizations
```

Request body:

```json
{
    "name": "New Organization",
    "slug": "new-org",
    "description": "Description of the organization",
    "website": "https://new-org.com"
}
```

Example request:

```bash
curl -X POST https://api.dotenv.cloud/v1/organizations \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "New Organization",
    "slug": "new-org"
  }'
```

Example response:

```json
{
    "data": {
        "id": "org_new123abc456",
        "name": "New Organization",
        "slug": "new-org",
        "created_at": "2024-01-15T12:00:00Z",
        "role": "owner"
    }
}
```

### Update Organization

Update organization details:

```http
PATCH /organizations/{organization_id}
```

Request body (all fields optional):

```json
{
    "name": "Updated Name",
    "description": "Updated description",
    "website": "https://updated-site.com",
    "logo_url": "https://assets.example.com/logo.png"
}
```

Example request:

```bash
curl -X PATCH https://api.dotenv.cloud/v1/organizations/org_abc123def456 \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "ACME Corp International",
    "description": "Global leader in innovative solutions"
  }'
```

### Delete Organization

Delete an organization (requires owner role):

```http
DELETE /organizations/{organization_id}
```

⚠️ **Warning**: This permanently deletes the organization and all associated data.

Example request:

```bash
curl -X DELETE https://api.dotenv.cloud/v1/organizations/org_abc123def456 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

## Organization Settings

### Get Settings

Get organization settings:

```http
GET /organizations/{organization_id}/settings
```

Example response:

```json
{
    "data": {
        "security": {
            "require_mfa": true,
            "allowed_auth_methods": ["password", "google", "github"],
            "password_policy": {
                "min_length": 12,
                "require_uppercase": true,
                "require_numbers": true,
                "require_symbols": true
            },
            "session_timeout": 3600,
            "ip_whitelist": ["10.0.0.0/8", "192.168.1.0/24"]
        },
        "notifications": {
            "email_notifications": true,
            "webhook_url": "https://example.com/webhook",
            "notification_events": [
                "member.added",
                "project.created",
                "secret.rotated"
            ]
        },
        "compliance": {
            "audit_retention_days": 365,
            "require_approval_for_production": true,
            "enforce_secret_rotation": true,
            "rotation_period_days": 90
        }
    }
}
```

### Update Settings

Update organization settings:

```http
PATCH /organizations/{organization_id}/settings
```

Request body:

```json
{
    "security": {
        "require_mfa": true,
        "session_timeout": 7200
    },
    "compliance": {
        "enforce_secret_rotation": true,
        "rotation_period_days": 60
    }
}
```

## Team Members

### List Members

Get organization members:

```http
GET /organizations/{organization_id}/members
```

Query parameters:

- `page` (integer): Page number
- `per_page` (integer): Items per page
- `role` (string): Filter by role (owner, admin, member)
- `status` (string): Filter by status (active, invited, suspended)
- `search` (string): Search by name or email

Example response:

```json
{
    "data": [
        {
            "id": "usr_123456",
            "email": "john@example.com",
            "name": "John Doe",
            "role": "admin",
            "status": "active",
            "joined_at": "2024-01-05T00:00:00Z",
            "last_active_at": "2024-01-15T09:30:00Z",
            "mfa_enabled": true
        }
    ],
    "pagination": {
        "current_page": 1,
        "per_page": 25,
        "total": 34
    }
}
```

### Invite Member

Invite a new member:

```http
POST /organizations/{organization_id}/members
```

Request body:

```json
{
    "email": "newuser@example.com",
    "role": "member",
    "projects": ["proj_123", "proj_456"],
    "message": "Welcome to our organization!"
}
```

Example response:

```json
{
    "data": {
        "id": "inv_abc123",
        "email": "newuser@example.com",
        "role": "member",
        "status": "invited",
        "invited_by": "usr_123456",
        "invited_at": "2024-01-15T12:00:00Z",
        "expires_at": "2024-01-22T12:00:00Z"
    }
}
```

### Update Member

Update member role or status:

```http
PATCH /organizations/{organization_id}/members/{member_id}
```

Request body:

```json
{
    "role": "admin",
    "status": "active"
}
```

### Remove Member

Remove a member from the organization:

```http
DELETE /organizations/{organization_id}/members/{member_id}
```

## Teams

### List Teams

Get organization teams:

```http
GET /organizations/{organization_id}/teams
```

Example response:

```json
{
    "data": [
        {
            "id": "team_frontend",
            "name": "Frontend Team",
            "description": "Frontend developers",
            "members_count": 8,
            "projects_count": 3,
            "created_at": "2024-01-10T00:00:00Z"
        }
    ]
}
```

### Create Team

Create a new team:

```http
POST /organizations/{organization_id}/teams
```

Request body:

```json
{
    "name": "Backend Team",
    "description": "Backend developers and DevOps",
    "members": ["usr_123", "usr_456"],
    "projects": ["proj_api", "proj_workers"]
}
```

### Team Permissions

Manage team permissions:

```http
PUT /organizations/{organization_id}/teams/{team_id}/permissions
```

Request body:

```json
{
    "permissions": [
        {
            "resource": "projects",
            "actions": ["read", "write"]
        },
        {
            "resource": "secrets",
            "actions": ["read"],
            "conditions": {
                "environments": ["development", "staging"]
            }
        }
    ]
}
```

## Projects

### List Organization Projects

Get all projects in an organization:

```http
GET /organizations/{organization_id}/projects
```

Query parameters:

- `page` (integer): Page number
- `per_page` (integer): Items per page
- `search` (string): Search by name
- `tag` (string): Filter by tag
- `archived` (boolean): Include archived projects

Example response:

```json
{
    "data": [
        {
            "id": "proj_abc123",
            "name": "API Service",
            "slug": "api-service",
            "description": "Main API backend",
            "environments_count": 3,
            "secrets_count": 45,
            "team": "team_backend",
            "created_at": "2024-01-05T00:00:00Z"
        }
    ]
}
```

## Audit Logs

### Get Audit Logs

Retrieve organization audit logs:

```http
GET /organizations/{organization_id}/audit-logs
```

Query parameters:

- `page` (integer): Page number
- `per_page` (integer): Items per page
- `start_date` (string): Start date (ISO 8601)
- `end_date` (string): End date (ISO 8601)
- `actor` (string): Filter by actor ID
- `action` (string): Filter by action type
- `resource_type` (string): Filter by resource type

Example response:

```json
{
    "data": [
        {
            "id": "log_xyz789",
            "action": "secret.created",
            "actor": {
                "id": "usr_123456",
                "email": "john@example.com",
                "ip_address": "192.168.1.100"
            },
            "resource": {
                "type": "secret",
                "id": "sec_abc123",
                "name": "API_KEY"
            },
            "metadata": {
                "project_id": "proj_123",
                "environment": "production"
            },
            "timestamp": "2024-01-15T10:30:00Z"
        }
    ]
}
```

## Billing

### Get Subscription

Get organization subscription details:

```http
GET /organizations/{organization_id}/subscription
```

Example response:

```json
{
    "data": {
        "plan": {
            "id": "plan_enterprise",
            "name": "Enterprise",
            "price_monthly": 500,
            "price_yearly": 5000
        },
        "status": "active",
        "current_period_start": "2024-01-01T00:00:00Z",
        "current_period_end": "2024-02-01T00:00:00Z",
        "seats": {
            "included": 50,
            "used": 34,
            "overage": 0
        },
        "usage": {
            "api_calls": 45678,
            "secrets_stored": 567,
            "audit_events": 12345
        }
    }
}
```

### Update Subscription

Update subscription plan:

```http
PUT /organizations/{organization_id}/subscription
```

Request body:

```json
{
    "plan_id": "plan_enterprise_plus",
    "seats": 100,
    "billing_cycle": "yearly"
}
```

## Webhooks

### Organization Webhooks

Manage organization-level webhooks:

```http
GET /organizations/{organization_id}/webhooks
POST /organizations/{organization_id}/webhooks
```

Webhook events:

- `organization.updated`
- `member.added`
- `member.removed`
- `member.role_changed`
- `team.created`
- `team.deleted`
- `project.created`
- `subscription.updated`

## Rate Limits

Organization endpoints have specific rate limits:

| Endpoint            | Rate Limit |
| ------------------- | ---------- |
| List organizations  | 100/hour   |
| Create organization | 10/hour    |
| Update organization | 50/hour    |
| Delete organization | 5/hour     |
| Invite members      | 100/hour   |

## Error Responses

### 404 Not Found

```json
{
    "error": {
        "code": "ORGANIZATION_NOT_FOUND",
        "message": "Organization not found or you don't have access"
    }
}
```

### 403 Forbidden

```json
{
    "error": {
        "code": "INSUFFICIENT_PERMISSIONS",
        "message": "You need 'admin' role to perform this action",
        "details": {
            "required_role": "admin",
            "your_role": "member"
        }
    }
}
```

### 422 Validation Error

```json
{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Invalid input data",
        "details": {
            "slug": ["The slug has already been taken"],
            "name": ["The name field is required"]
        }
    }
}
```

## Best Practices

1. **Cache organization data**: Organizations don't change frequently
2. **Use webhooks**: For real-time updates on organization changes
3. **Implement retry logic**: For transient failures
4. **Respect rate limits**: Use exponential backoff
5. **Minimize API calls**: Use batch operations when available

## Examples

### JavaScript SDK

```javascript
// Initialize client
const dotenv = new DotEnvAPI({ apiKey: "YOUR_API_KEY" });

// List organizations
const orgs = await dotenv.organizations.list();

// Create organization
const newOrg = await dotenv.organizations.create({
    name: "New Startup",
    slug: "new-startup",
});

// Invite member
await dotenv.organizations.inviteMember(orgId, {
    email: "developer@example.com",
    role: "member",
});
```

### Python SDK

```python
from dotenv_api import DotEnvAPI

# Initialize client
client = DotEnvAPI(api_key='YOUR_API_KEY')

# Get organization
org = client.organizations.get('org_abc123')

# Update settings
client.organizations.update_settings(
    org_id=org.id,
    settings={
        'security': {
            'require_mfa': True
        }
    }
)
```

## Next Steps

- [Projects API](./projects) - Project management
- [Teams Guide](/documentation/v1/core-concepts/access-control) - Team permissions
- [Billing Guide](/documentation/v1/administration/billing) - Subscription management
- [Audit Logs](/documentation/v1/administration/audit-logs) - Audit log details
