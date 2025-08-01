---
title: Projects API
slug: projects
order: 4
tags: [api, projects, environments]
---

# Projects API

The Projects API allows you to manage projects and their environments. Projects are containers for secrets and provide logical separation between different applications or services.

## Project Object

```json
{
    "id": "proj_abc123def456",
    "organization_id": "org_xyz789",
    "name": "API Service",
    "slug": "api-service",
    "description": "Main API backend service",
    "color": "#3B82F6",
    "icon": "server",
    "tags": ["backend", "production", "critical"],
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-15T10:30:00Z",
    "created_by": {
        "id": "usr_123456",
        "email": "john@example.com"
    },
    "settings": {
        "auto_rotate_secrets": true,
        "rotation_period_days": 90,
        "require_approval_for_changes": true,
        "allowed_ips": ["10.0.0.0/8"],
        "webhook_url": "https://api.example.com/webhooks/dotenv"
    },
    "stats": {
        "environments_count": 3,
        "secrets_count": 45,
        "members_count": 12,
        "last_accessed": "2024-01-15T09:00:00Z"
    },
    "status": "active"
}
```

## Endpoints

### List Projects

Get all projects in an organization:

```http
GET /projects
GET /organizations/{organization_id}/projects
```

Query parameters:

- `page` (integer): Page number
- `per_page` (integer): Items per page (max 100)
- `search` (string): Search by name or description
- `tag` (string): Filter by tag
- `status` (string): Filter by status (active, archived)
- `sort` (string): Sort field (name, created_at, updated_at)
- `order` (string): Sort order (asc, desc)

Example request:

```bash
curl -X GET https://api.dotenv.cloud/v1/projects?tag=production \
  -H "Authorization: Bearer YOUR_API_KEY"
```

Example response:

```json
{
    "data": [
        {
            "id": "proj_abc123def456",
            "name": "API Service",
            "slug": "api-service",
            "organization_id": "org_xyz789",
            "environments_count": 3,
            "secrets_count": 45,
            "tags": ["backend", "production"],
            "created_at": "2024-01-01T00:00:00Z",
            "status": "active"
        }
    ],
    "pagination": {
        "current_page": 1,
        "per_page": 25,
        "total": 12,
        "total_pages": 1
    }
}
```

### Get Project

Get details of a specific project:

```http
GET /projects/{project_id}
```

Path parameters:

- `project_id` (string): Project ID or slug

Example request:

```bash
curl -X GET https://api.dotenv.cloud/v1/projects/proj_abc123def456 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### Create Project

Create a new project:

```http
POST /projects
```

Request body:

```json
{
    "organization_id": "org_xyz789",
    "name": "New Service",
    "slug": "new-service",
    "description": "Microservice for handling payments",
    "color": "#10B981",
    "icon": "credit-card",
    "tags": ["microservice", "payments"],
    "settings": {
        "auto_rotate_secrets": true,
        "rotation_period_days": 60
    }
}
```

Example request:

```bash
curl -X POST https://api.dotenv.cloud/v1/projects \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Payment Service",
    "slug": "payment-service",
    "organization_id": "org_xyz789"
  }'
```

### Update Project

Update project details:

```http
PATCH /projects/{project_id}
```

Request body (all fields optional):

```json
{
    "name": "Updated Name",
    "description": "Updated description",
    "color": "#EF4444",
    "tags": ["updated", "tags"],
    "settings": {
        "require_approval_for_changes": true
    }
}
```

### Delete Project

Delete a project:

```http
DELETE /projects/{project_id}
```

⚠️ **Warning**: This permanently deletes the project and all associated data.

Query parameters:

- `force` (boolean): Skip confirmation (default: false)

Example request:

```bash
curl -X DELETE https://api.dotenv.cloud/v1/projects/proj_abc123def456?force=true \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### Archive Project

Archive a project (soft delete):

```http
POST /projects/{project_id}/archive
```

### Restore Project

Restore an archived project:

```http
POST /projects/{project_id}/restore
```

## Environments

### List Environments

Get all environments for a project:

```http
GET /projects/{project_id}/environments
```

Example response:

```json
{
    "data": [
        {
            "id": "env_dev123",
            "name": "development",
            "slug": "development",
            "color": "#3B82F6",
            "is_default": true,
            "inherits_from": null,
            "secrets_count": 15,
            "created_at": "2024-01-01T00:00:00Z"
        },
        {
            "id": "env_stg456",
            "name": "staging",
            "slug": "staging",
            "color": "#F59E0B",
            "is_default": false,
            "inherits_from": "env_dev123",
            "secrets_count": 18,
            "created_at": "2024-01-01T00:00:00Z"
        },
        {
            "id": "env_prd789",
            "name": "production",
            "slug": "production",
            "color": "#EF4444",
            "is_default": false,
            "inherits_from": "env_stg456",
            "secrets_count": 20,
            "created_at": "2024-01-01T00:00:00Z"
        }
    ]
}
```

### Create Environment

Create a new environment:

```http
POST /projects/{project_id}/environments
```

Request body:

```json
{
    "name": "qa",
    "slug": "qa",
    "color": "#8B5CF6",
    "inherits_from": "env_dev123",
    "description": "QA testing environment"
}
```

### Update Environment

Update environment details:

```http
PATCH /projects/{project_id}/environments/{environment_id}
```

### Delete Environment

Delete an environment:

```http
DELETE /projects/{project_id}/environments/{environment_id}
```

### Clone Environment

Clone an existing environment:

```http
POST /projects/{project_id}/environments/{environment_id}/clone
```

Request body:

```json
{
    "name": "staging-copy",
    "slug": "staging-copy",
    "include_values": true
}
```

## Project Settings

### Get Settings

Get project settings:

```http
GET /projects/{project_id}/settings
```

Example response:

```json
{
    "data": {
        "general": {
            "auto_rotate_secrets": true,
            "rotation_period_days": 90,
            "notification_email": "team@example.com"
        },
        "security": {
            "require_approval_for_changes": true,
            "approval_required_environments": ["production"],
            "allowed_ips": ["10.0.0.0/8"],
            "enforce_encryption": true
        },
        "integrations": {
            "webhook_url": "https://api.example.com/webhooks/dotenv",
            "webhook_secret": "whsec_abc123",
            "slack_webhook": "https://hooks.slack.com/services/..."
        },
        "compliance": {
            "audit_retention_days": 365,
            "require_secret_scanning": true,
            "block_plaintext_secrets": true
        }
    }
}
```

### Update Settings

Update project settings:

```http
PATCH /projects/{project_id}/settings
```

## Project Members

### List Project Members

Get members with access to the project:

```http
GET /projects/{project_id}/members
```

Example response:

```json
{
    "data": [
        {
            "id": "usr_123456",
            "email": "john@example.com",
            "name": "John Doe",
            "role": "admin",
            "permissions": ["read", "write", "delete"],
            "added_at": "2024-01-05T00:00:00Z",
            "last_accessed": "2024-01-15T09:00:00Z"
        }
    ]
}
```

### Add Project Member

Grant access to a project:

```http
POST /projects/{project_id}/members
```

Request body:

```json
{
    "user_id": "usr_789012",
    "role": "developer",
    "permissions": ["read", "write"],
    "environments": ["development", "staging"]
}
```

### Update Member Permissions

Update member's project permissions:

```http
PATCH /projects/{project_id}/members/{member_id}
```

### Remove Project Member

Remove member from project:

```http
DELETE /projects/{project_id}/members/{member_id}
```

## Project Statistics

### Get Statistics

Get project usage statistics:

```http
GET /projects/{project_id}/stats
```

Query parameters:

- `period` (string): Time period (day, week, month, year)
- `start_date` (string): Start date (ISO 8601)
- `end_date` (string): End date (ISO 8601)

Example response:

```json
{
    "data": {
        "overview": {
            "total_secrets": 45,
            "total_environments": 3,
            "total_members": 12,
            "last_activity": "2024-01-15T10:00:00Z"
        },
        "activity": {
            "secret_reads": 1234,
            "secret_writes": 56,
            "secret_rotations": 8,
            "member_accesses": 89
        },
        "trends": {
            "secrets_growth": "+12%",
            "activity_change": "+34%",
            "rotation_compliance": "95%"
        },
        "by_environment": {
            "development": {
                "secrets": 15,
                "reads": 456,
                "writes": 34
            },
            "staging": {
                "secrets": 18,
                "reads": 345,
                "writes": 12
            },
            "production": {
                "secrets": 20,
                "reads": 433,
                "writes": 10
            }
        }
    }
}
```

## Project Activity

### Get Activity Log

Get project activity log:

```http
GET /projects/{project_id}/activity
```

Query parameters:

- `page` (integer): Page number
- `per_page` (integer): Items per page
- `action` (string): Filter by action type
- `actor` (string): Filter by actor ID
- `start_date` (string): Start date
- `end_date` (string): End date

Example response:

```json
{
    "data": [
        {
            "id": "act_xyz789",
            "action": "secret.updated",
            "actor": {
                "id": "usr_123456",
                "email": "john@example.com"
            },
            "resource": {
                "type": "secret",
                "id": "sec_abc123",
                "name": "DATABASE_URL"
            },
            "environment": "production",
            "timestamp": "2024-01-15T10:30:00Z",
            "ip_address": "192.168.1.100",
            "user_agent": "DotEnv-CLI/1.0.0"
        }
    ]
}
```

## Project Webhooks

### List Webhooks

Get project webhooks:

```http
GET /projects/{project_id}/webhooks
```

### Create Webhook

Create a project webhook:

```http
POST /projects/{project_id}/webhooks
```

Request body:

```json
{
    "url": "https://api.example.com/webhooks/dotenv",
    "events": ["secret.created", "secret.updated", "secret.deleted"],
    "environments": ["production"],
    "secret": "your-webhook-secret",
    "active": true
}
```

### Test Webhook

Test webhook configuration:

```http
POST /projects/{project_id}/webhooks/{webhook_id}/test
```

## Project Templates

### Create from Template

Create a project from a template:

```http
POST /projects/from-template
```

Request body:

```json
{
    "template_id": "tmpl_nodejs_api",
    "organization_id": "org_xyz789",
    "name": "New API",
    "slug": "new-api",
    "environments": ["development", "staging", "production"]
}
```

### Export as Template

Export project as a template:

```http
POST /projects/{project_id}/export-template
```

Request body:

```json
{
    "name": "Node.js API Template",
    "description": "Standard Node.js API setup",
    "include_secrets": false,
    "include_settings": true
}
```

## Bulk Operations

### Bulk Create Projects

Create multiple projects:

```http
POST /projects/bulk
```

Request body:

```json
{
    "organization_id": "org_xyz789",
    "projects": [
        {
            "name": "Service A",
            "slug": "service-a"
        },
        {
            "name": "Service B",
            "slug": "service-b"
        }
    ],
    "template_id": "tmpl_microservice"
}
```

### Bulk Update Projects

Update multiple projects:

```http
PATCH /projects/bulk
```

Request body:

```json
{
    "project_ids": ["proj_123", "proj_456"],
    "updates": {
        "settings": {
            "auto_rotate_secrets": true
        },
        "tags": ["updated"]
    }
}
```

## Import/Export

### Export Project

Export project configuration:

```http
GET /projects/{project_id}/export
```

Query parameters:

- `format` (string): Export format (json, yaml)
- `include_secrets` (boolean): Include secret values
- `include_members` (boolean): Include member list

### Import Project

Import project configuration:

```http
POST /projects/import
```

Request body:

```json
{
  "organization_id": "org_xyz789",
  "format": "json",
  "data": {
    "name": "Imported Project",
    "environments": [...],
    "secrets": [...]
  }
}
```

## Error Responses

### 404 Not Found

```json
{
    "error": {
        "code": "PROJECT_NOT_FOUND",
        "message": "Project not found or you don't have access"
    }
}
```

### 409 Conflict

```json
{
    "error": {
        "code": "PROJECT_EXISTS",
        "message": "A project with this slug already exists",
        "details": {
            "slug": "api-service"
        }
    }
}
```

### 422 Validation Error

```json
{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Invalid project configuration",
        "details": {
            "name": ["The name field is required"],
            "slug": ["The slug must be unique within the organization"]
        }
    }
}
```

## Best Practices

1. **Use meaningful slugs**: They're used in CLI and URLs
2. **Tag projects**: Makes filtering and organization easier
3. **Set up environments early**: Define your environment strategy
4. **Configure webhooks**: For real-time integrations
5. **Regular backups**: Export configurations periodically
6. **Monitor activity**: Review access logs regularly

## SDK Examples

### JavaScript

```javascript
const dotenv = new DotEnvAPI({ apiKey: "YOUR_API_KEY" });

// List projects
const projects = await dotenv.projects.list({
    tag: "production",
    sort: "updated_at",
    order: "desc",
});

// Create project
const project = await dotenv.projects.create({
    name: "New Service",
    slug: "new-service",
    organization_id: "org_xyz789",
});

// Add environment
await dotenv.projects.createEnvironment(project.id, {
    name: "staging",
    inherits_from: "development",
});
```

### Python

```python
from dotenv_api import DotEnvAPI

client = DotEnvAPI(api_key='YOUR_API_KEY')

# Get project
project = client.projects.get('proj_abc123')

# Update settings
client.projects.update_settings(
    project_id=project.id,
    settings={
        'security': {
            'require_approval_for_changes': True
        }
    }
)

# Export project
export_data = client.projects.export(
    project_id=project.id,
    include_secrets=False
)
```

## Next Steps

- [Secrets API](./secrets) - Managing secrets
- [Environments Guide](/documentation/v1/core-concepts/environments-concept) - Environment strategies
- [Webhooks](/documentation/v1/drafts/api/webhooks) *(Coming Soon)* - Setting up webhooks
- [Access Control](/documentation/v1/core-concepts/access-control) - Permission management
