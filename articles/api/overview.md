---
title: API Overview
slug: overview
order: 1
tags: [api, rest, overview]
---

# API Overview

The DotEnv API is a RESTful API that provides programmatic access to all DotEnv functionality. It's designed to be simple, secure, and scalable.

## Base URL

```
https://api.dotenv.cloud/v1
```

For self-hosted installations:

```
https://your-domain.com/api/v1
```

## API Design Principles

### RESTful Architecture

The API follows REST principles:

- **Resources**: Nouns (e.g., `/projects`, `/secrets`)
- **Methods**: Standard HTTP verbs (GET, POST, PUT, DELETE)
- **Stateless**: Each request contains all information needed
- **Uniform Interface**: Consistent patterns across endpoints

### JSON API

All requests and responses use JSON:

```http
Content-Type: application/json
Accept: application/json
```

### API Versioning

The API is versioned via the URL path:

```
https://api.dotenv.cloud/v1/...  # Current version
https://api.dotenv.cloud/v2/...  # Future version
```

## Authentication

All API requests require authentication using an API key:

```http
Authorization: Bearer dotenv_api_prod_abc123xyz
```

Example:

```bash
curl -H "Authorization: Bearer dotenv_api_prod_abc123xyz" \
  https://api.dotenv.cloud/v1/projects
```

## Request Format

### Headers

Required headers for all requests:

```http
Authorization: Bearer YOUR_API_KEY
Content-Type: application/json
Accept: application/json
```

Optional headers:

```http
X-Request-ID: unique-request-id
X-DotEnv-Version: 1.0.0
User-Agent: your-app/1.0
```

### Request Body

POST and PUT requests accept JSON:

```json
{
    "name": "my-project",
    "description": "Project description",
    "tags": ["production", "api"]
}
```

### Query Parameters

GET requests support query parameters:

```
GET /projects?page=2&limit=50&sort=created_at&order=desc
```

## Response Format

### Success Response

```json
{
    "data": {
        "id": "proj_abc123",
        "name": "my-project",
        "created_at": "2024-01-15T10:30:00Z",
        "updated_at": "2024-01-15T10:30:00Z"
    },
    "meta": {
        "request_id": "req_xyz789"
    }
}
```

### Collection Response

```json
{
    "data": [
        {
            "id": "proj_abc123",
            "name": "project-1"
        },
        {
            "id": "proj_def456",
            "name": "project-2"
        }
    ],
    "pagination": {
        "current_page": 1,
        "per_page": 25,
        "total": 100,
        "total_pages": 4,
        "next_page": 2,
        "prev_page": null
    },
    "meta": {
        "request_id": "req_xyz789"
    }
}
```

### Error Response

```json
{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "The request body is invalid",
        "details": {
            "name": ["The name field is required"],
            "email": ["The email must be a valid email address"]
        }
    },
    "meta": {
        "request_id": "req_xyz789",
        "documentation_url": "https://docs.dotenv.cloud/errors/validation"
    }
}
```

## HTTP Methods

### GET

Retrieve resources:

```http
GET /projects
GET /projects/{id}
GET /projects/{id}/secrets
```

### POST

Create new resources:

```http
POST /projects
POST /projects/{id}/secrets
```

### PUT

Update entire resources:

```http
PUT /projects/{id}
PUT /secrets/{id}
```

### PATCH

Partial updates:

```http
PATCH /projects/{id}
PATCH /secrets/{id}
```

### DELETE

Remove resources:

```http
DELETE /projects/{id}
DELETE /secrets/{id}
```

## Status Codes

### Success Codes

| Code | Meaning    | Usage                      |
| ---- | ---------- | -------------------------- |
| 200  | OK         | Successful GET, PUT, PATCH |
| 201  | Created    | Successful POST            |
| 204  | No Content | Successful DELETE          |
| 202  | Accepted   | Async operation started    |

### Client Error Codes

| Code | Meaning           | Usage                  |
| ---- | ----------------- | ---------------------- |
| 400  | Bad Request       | Invalid request format |
| 401  | Unauthorized      | Missing/invalid auth   |
| 403  | Forbidden         | No permission          |
| 404  | Not Found         | Resource doesn't exist |
| 409  | Conflict          | Duplicate resource     |
| 422  | Unprocessable     | Validation errors      |
| 429  | Too Many Requests | Rate limit exceeded    |

### Server Error Codes

| Code | Meaning             | Usage            |
| ---- | ------------------- | ---------------- |
| 500  | Internal Error      | Server error     |
| 503  | Service Unavailable | Maintenance mode |
| 504  | Gateway Timeout     | Request timeout  |

## Pagination

List endpoints support pagination:

```http
GET /projects?page=2&per_page=50
```

Parameters:

- `page`: Page number (default: 1)
- `per_page`: Items per page (default: 25, max: 100)
- `cursor`: Cursor-based pagination token

Response includes pagination metadata:

```json
{
  "data": [...],
  "pagination": {
    "current_page": 2,
    "per_page": 50,
    "total": 234,
    "total_pages": 5,
    "next_page": 3,
    "prev_page": 1,
    "next_cursor": "eyJpZCI6MTAwfQ==",
    "prev_cursor": "eyJpZCI6NTB9"
  }
}
```

## Filtering

Filter results using query parameters:

```http
GET /secrets?environment=production&tag=database
GET /projects?organization=org_123&archived=false
```

### Operators

- `eq`: Equals (default)
- `ne`: Not equals
- `gt`: Greater than
- `gte`: Greater than or equal
- `lt`: Less than
- `lte`: Less than or equal
- `like`: Contains
- `in`: In array

Example:

```http
GET /secrets?created_at[gte]=2024-01-01&key[like]=API
```

## Sorting

Sort results using the `sort` parameter:

```http
GET /projects?sort=created_at&order=desc
GET /secrets?sort=key&order=asc
```

Multiple sort fields:

```http
GET /projects?sort=organization,name&order=asc,desc
```

## Field Selection

Select specific fields to reduce payload:

```http
GET /projects?fields=id,name,created_at
GET /secrets?fields=key,environment,updated_at
```

## Relationships

Include related resources:

```http
GET /projects?include=organization,environments
GET /secrets?include=project,created_by
```

## Search

Full-text search on supported endpoints:

```http
GET /projects?q=api
GET /secrets?search=database
```

## Batch Operations

Some endpoints support batch operations:

```http
POST /secrets/batch
{
  "operations": [
    {"method": "create", "data": {"key": "API_KEY", "value": "..."}},
    {"method": "update", "id": "sec_123", "data": {"value": "..."}},
    {"method": "delete", "id": "sec_456"}
  ]
}
```

## Webhooks

Register webhooks for real-time updates:

```http
POST /webhooks
{
  "url": "https://your-app.com/webhook",
  "events": ["secret.created", "secret.updated"],
  "secret": "webhook_secret_key"
}
```

## Rate Limiting

API requests are rate limited:

- **Anonymous**: 100 requests/hour
- **Authenticated**: 1,000 requests/hour
- **Enterprise**: Custom limits

Rate limit headers:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1642089600
```

## CORS

The API supports CORS for browser-based applications:

```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Authorization, Content-Type
```

## Compression

Responses are compressed when requested:

```http
Accept-Encoding: gzip, deflate
```

## Caching

Some responses include cache headers:

```http
Cache-Control: private, max-age=300
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
```

Conditional requests:

```http
If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"
```

## SDK Support

Official SDKs available:

- JavaScript/TypeScript
- Python
- Go
- PHP
- Ruby
- Java
- .NET

Example (JavaScript):

```javascript
import { DotEnvAPI } from "@dotenv/api";

const api = new DotEnvAPI({
    apiKey: "dotenv_api_prod_abc123",
});

const projects = await api.projects.list();
```

## API Explorer

Interactive API documentation:
https://api.dotenv.cloud/explorer

Features:

- Try endpoints directly
- Generate code examples
- View request/response schemas
- Test authentication

## Getting Started

1. **Get API Key**: Generate from dashboard
2. **Make First Request**:
    ```bash
    curl -H "Authorization: Bearer YOUR_API_KEY" \
      https://api.dotenv.cloud/v1/projects
    ```
3. **Explore Endpoints**: Use API Explorer
4. **Integrate**: Use SDKs or direct HTTP

## Best Practices

1. **Use SDKs**: Easier than direct HTTP
2. **Handle Errors**: Implement retry logic
3. **Respect Limits**: Monitor rate limits
4. **Secure Keys**: Never expose API keys
5. **Use Webhooks**: For real-time updates
6. **Cache Responses**: Reduce API calls
7. **Batch Operations**: For bulk changes

## Next Steps

- [Authentication](./authentication) - Detailed auth guide
- [Organizations API](./organizations) - Organization management
- [Projects API](./projects) - Project endpoints
- [Secrets API](./secrets) - Secret management
- [Webhooks](/documentation/v1/drafts/api/webhooks) *(Coming Soon)* - Real-time events
- [Rate Limiting](./rate-limiting) - Limits and strategies
- [Error Handling](./error-handling) - Error codes and handling
