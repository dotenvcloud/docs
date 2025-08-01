---
title: Webhooks
slug: webhooks
order: 6
tags: [api, webhooks, events, real-time]
---

# Webhooks

Webhooks allow you to receive real-time notifications when events occur in your DotEnv account. Instead of polling the API, webhooks push data to your endpoint as events happen.

## Overview

Webhooks are HTTP callbacks that DotEnv sends to your specified URL when certain events occur. They enable you to:

- React to changes in real-time
- Build integrations and automations
- Audit and monitor activity
- Sync data with external systems

## Webhook Object

```json
{
    "id": "whk_abc123def456",
    "url": "https://api.example.com/webhooks/dotenv",
    "name": "Production Webhook",
    "description": "Notifies production changes",
    "secret": "whsec_live_abc123xyz",
    "events": ["secret.created", "secret.updated", "secret.deleted"],
    "active": true,
    "filters": {
        "environments": ["production"],
        "projects": ["proj_api", "proj_web"]
    },
    "headers": {
        "X-Custom-Header": "custom-value"
    },
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-15T10:30:00Z",
    "last_triggered_at": "2024-01-15T10:00:00Z",
    "success_rate": 98.5,
    "status": "healthy"
}
```

## Webhook Events

### Secret Events

- `secret.created` - New secret created
- `secret.updated` - Secret value or metadata updated
- `secret.deleted` - Secret deleted
- `secret.rotated` - Secret rotated
- `secret.accessed` - Secret value accessed (read)
- `secret.bulk_import` - Multiple secrets imported
- `secret.validation_failed` - Secret validation failed

### Project Events

- `project.created` - New project created
- `project.updated` - Project settings updated
- `project.deleted` - Project deleted
- `project.archived` - Project archived
- `project.restored` - Project restored
- `project.member_added` - Member added to project
- `project.member_removed` - Member removed from project

### Environment Events

- `environment.created` - New environment created
- `environment.updated` - Environment updated
- `environment.deleted` - Environment deleted
- `environment.cloned` - Environment cloned

### Organization Events

- `organization.updated` - Organization settings updated
- `organization.member_invited` - Member invited
- `organization.member_joined` - Member accepted invitation
- `organization.member_removed` - Member removed
- `organization.subscription_updated` - Subscription changed

### Security Events

- `auth.login` - User logged in
- `auth.logout` - User logged out
- `auth.failed_login` - Failed login attempt
- `auth.mfa_enabled` - MFA enabled
- `auth.api_key_created` - API key created
- `auth.api_key_deleted` - API key deleted

## Creating Webhooks

### Create Webhook

```http
POST /webhooks
```

Request body:

```json
{
    "url": "https://api.example.com/webhooks/dotenv",
    "name": "Production Alerts",
    "description": "Notify on production secret changes",
    "events": ["secret.created", "secret.updated", "secret.deleted"],
    "filters": {
        "environments": ["production"],
        "projects": ["proj_api"]
    },
    "headers": {
        "X-Custom-Auth": "bearer-token"
    },
    "active": true
}
```

Example request:

```bash
curl -X POST https://api.dotenv.cloud/v1/webhooks \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://api.example.com/webhook",
    "events": ["secret.updated", "secret.deleted"]
  }'
```

Response:

```json
{
    "data": {
        "id": "whk_new123",
        "url": "https://api.example.com/webhook",
        "secret": "whsec_live_xyz789abc",
        "events": ["secret.updated", "secret.deleted"],
        "created_at": "2024-01-15T12:00:00Z"
    }
}
```

### Update Webhook

```http
PATCH /webhooks/{webhook_id}
```

Request body:

```json
{
    "url": "https://api.example.com/new-webhook-url",
    "events": ["secret.*", "project.*"],
    "active": true
}
```

### Delete Webhook

```http
DELETE /webhooks/{webhook_id}
```

## Webhook Payload

### Payload Structure

All webhook deliveries have a consistent structure:

```json
{
    "id": "evt_abc123def456",
    "webhook_id": "whk_xyz789",
    "event": "secret.updated",
    "created_at": "2024-01-15T10:30:00Z",
    "data": {
        // Event-specific data
    },
    "actor": {
        "id": "usr_123456",
        "email": "john@example.com",
        "type": "user"
    },
    "organization": {
        "id": "org_abc123",
        "name": "ACME Corp"
    }
}
```

### Event-Specific Payloads

#### Secret Created

```json
{
    "event": "secret.created",
    "data": {
        "secret": {
            "id": "sec_abc123",
            "key": "NEW_API_KEY",
            "project_id": "proj_xyz789",
            "environment": "production",
            "tags": ["api", "external"],
            "created_at": "2024-01-15T10:30:00Z"
        }
    }
}
```

#### Secret Updated

```json
{
    "event": "secret.updated",
    "data": {
        "secret": {
            "id": "sec_abc123",
            "key": "DATABASE_URL",
            "project_id": "proj_xyz789",
            "environment": "production"
        },
        "changes": {
            "value": true,
            "description": true,
            "tags": {
                "added": ["critical"],
                "removed": ["test"]
            }
        },
        "previous_version": 2,
        "current_version": 3
    }
}
```

#### Project Member Added

```json
{
    "event": "project.member_added",
    "data": {
        "project": {
            "id": "proj_xyz789",
            "name": "API Service"
        },
        "member": {
            "id": "usr_456789",
            "email": "jane@example.com",
            "role": "developer",
            "permissions": ["read", "write"]
        }
    }
}
```

## Webhook Security

### Signature Verification

All webhooks are signed using HMAC-SHA256. Verify the signature to ensure authenticity:

```javascript
const crypto = require("crypto");

function verifyWebhookSignature(payload, signature, secret) {
    const expectedSignature = crypto
        .createHmac("sha256", secret)
        .update(payload)
        .digest("hex");

    return crypto.timingSafeEqual(
        Buffer.from(signature),
        Buffer.from(expectedSignature),
    );
}

// Express.js example
app.post("/webhook", (req, res) => {
    const signature = req.headers["x-dotenv-signature"];
    const timestamp = req.headers["x-dotenv-timestamp"];
    const payload = JSON.stringify(req.body);

    // Verify timestamp (prevent replay attacks)
    const currentTime = Math.floor(Date.now() / 1000);
    if (Math.abs(currentTime - parseInt(timestamp)) > 300) {
        return res.status(400).send("Timestamp too old");
    }

    // Verify signature
    const signedPayload = `${timestamp}.${payload}`;
    if (!verifyWebhookSignature(signedPayload, signature, webhookSecret)) {
        return res.status(401).send("Invalid signature");
    }

    // Process webhook
    console.log("Webhook verified:", req.body);
    res.status(200).send("OK");
});
```

### Python Example

```python
import hmac
import hashlib
import time
from flask import Flask, request, abort

app = Flask(__name__)

def verify_webhook_signature(payload, signature, secret):
    expected_signature = hmac.new(
        secret.encode('utf-8'),
        payload.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(signature, expected_signature)

@app.route('/webhook', methods=['POST'])
def handle_webhook():
    signature = request.headers.get('X-DotEnv-Signature')
    timestamp = request.headers.get('X-DotEnv-Timestamp')

    # Verify timestamp
    current_time = int(time.time())
    if abs(current_time - int(timestamp)) > 300:
        abort(400, 'Timestamp too old')

    # Verify signature
    payload = request.get_data(as_text=True)
    signed_payload = f"{timestamp}.{payload}"

    if not verify_webhook_signature(signed_payload, signature, webhook_secret):
        abort(401, 'Invalid signature')

    # Process webhook
    data = request.get_json()
    print(f"Received {data['event']} event")

    return 'OK', 200
```

### Headers

Webhook requests include these headers:

```http
X-DotEnv-Signature: sha256=abc123...
X-DotEnv-Timestamp: 1642089600
X-DotEnv-Event: secret.updated
X-DotEnv-Webhook-ID: whk_abc123
X-DotEnv-Delivery-ID: del_xyz789
Content-Type: application/json
User-Agent: DotEnv-Webhooks/1.0
```

## Webhook Management

### List Webhooks

```http
GET /webhooks
```

Query parameters:

- `page` (integer): Page number
- `per_page` (integer): Items per page
- `active` (boolean): Filter by status
- `event` (string): Filter by event type

### Get Webhook

```http
GET /webhooks/{webhook_id}
```

### Test Webhook

Send a test event to verify configuration:

```http
POST /webhooks/{webhook_id}/test
```

Request body:

```json
{
    "event": "test.ping",
    "data": {
        "message": "Test webhook delivery"
    }
}
```

### Get Webhook Deliveries

View delivery history:

```http
GET /webhooks/{webhook_id}/deliveries
```

Response:

```json
{
    "data": [
        {
            "id": "del_abc123",
            "event": "secret.created",
            "status": "success",
            "status_code": 200,
            "duration_ms": 145,
            "attempted_at": "2024-01-15T10:30:00Z",
            "response_headers": {
                "content-type": "application/json"
            },
            "response_body": "{\"status\":\"ok\"}"
        }
    ]
}
```

### Retry Failed Delivery

```http
POST /webhooks/{webhook_id}/deliveries/{delivery_id}/retry
```

## Webhook Filters

### Environment Filters

Only receive events from specific environments:

```json
{
    "filters": {
        "environments": ["production", "staging"]
    }
}
```

### Project Filters

Limit events to certain projects:

```json
{
    "filters": {
        "projects": ["proj_api", "proj_web"]
    }
}
```

### Event Patterns

Use wildcards to subscribe to event groups:

```json
{
    "events": [
        "secret.*", // All secret events
        "*.created", // All creation events
        "project.member_*" // All project member events
    ]
}
```

### Custom Filters

Advanced filtering with conditions:

```json
{
    "filters": {
        "conditions": [
            {
                "field": "data.secret.tags",
                "operator": "contains",
                "value": "critical"
            },
            {
                "field": "actor.type",
                "operator": "equals",
                "value": "user"
            }
        ]
    }
}
```

## Reliability

### Delivery Guarantees

- **At-least-once delivery**: Events may be delivered multiple times
- **Ordered delivery**: Events are delivered in order per webhook
- **Retry policy**: Failed deliveries are retried with exponential backoff

### Retry Schedule

1. Immediate retry
2. 5 seconds
3. 30 seconds
4. 2 minutes
5. 10 minutes
6. 30 minutes
7. 1 hour
8. 2 hours
9. 4 hours
10. 8 hours

After 10 failed attempts, the webhook is marked as failed.

### Handling Failures

Webhooks expect a 2xx status code. Other responses trigger retries:

```javascript
// Good response
res.status(200).json({ status: "processed" });

// Will trigger retry
res.status(500).json({ error: "Internal error" });
```

### Idempotency

Handle duplicate deliveries using the event ID:

```javascript
const processedEvents = new Set();

app.post("/webhook", async (req, res) => {
    const eventId = req.body.id;

    // Check if already processed
    if (processedEvents.has(eventId)) {
        return res.status(200).json({ status: "already_processed" });
    }

    // Process event
    await processEvent(req.body);
    processedEvents.add(eventId);

    res.status(200).json({ status: "processed" });
});
```

## Best Practices

### 1. Verify Signatures

Always verify webhook signatures to ensure authenticity.

### 2. Respond Quickly

Return a 200 status immediately, process asynchronously:

```javascript
app.post("/webhook", (req, res) => {
    // Respond immediately
    res.status(200).send("OK");

    // Process asynchronously
    setImmediate(() => {
        processWebhookEvent(req.body);
    });
});
```

### 3. Handle Retries

Implement idempotency to handle duplicate deliveries.

### 4. Monitor Health

Track webhook success rates and response times.

### 5. Use HTTPS

Always use HTTPS endpoints for security.

### 6. Implement Timeouts

Set reasonable timeouts to prevent hanging connections.

### 7. Log Everything

Log all webhook activity for debugging.

## Debugging Webhooks

### Webhook Inspector

Use the webhook inspector in the dashboard to:

- View recent deliveries
- Inspect request/response data
- Retry failed deliveries
- Test webhook configuration

### Common Issues

1. **401 Unauthorized**

    - Check signature verification
    - Verify webhook secret

2. **Connection Timeout**

    - Ensure endpoint is publicly accessible
    - Check firewall rules

3. **SSL Errors**

    - Use valid SSL certificate
    - Support TLS 1.2+

4. **Response Too Slow**
    - Respond quickly, process async
    - Optimize endpoint performance

## Examples

### Node.js Express

```javascript
const express = require("express");
const crypto = require("crypto");

const app = express();
app.use(express.json());

app.post("/webhook", (req, res) => {
    // Verify webhook
    const signature = req.headers["x-dotenv-signature"];
    const timestamp = req.headers["x-dotenv-timestamp"];

    const payload = `${timestamp}.${JSON.stringify(req.body)}`;
    const expectedSignature = crypto
        .createHmac("sha256", process.env.WEBHOOK_SECRET)
        .update(payload)
        .digest("hex");

    if (signature !== expectedSignature) {
        return res.status(401).send("Unauthorized");
    }

    // Handle events
    const { event, data } = req.body;

    switch (event) {
        case "secret.created":
            console.log("New secret:", data.secret.key);
            break;
        case "secret.updated":
            console.log("Updated secret:", data.secret.key);
            break;
        case "secret.deleted":
            console.log("Deleted secret:", data.secret.key);
            break;
    }

    res.status(200).send("OK");
});

app.listen(3000);
```

### AWS Lambda

```javascript
exports.handler = async (event) => {
    const body = JSON.parse(event.body);
    const signature = event.headers["x-dotenv-signature"];
    const timestamp = event.headers["x-dotenv-timestamp"];

    // Verify signature
    const payload = `${timestamp}.${event.body}`;
    const expectedSignature = crypto
        .createHmac("sha256", process.env.WEBHOOK_SECRET)
        .update(payload)
        .digest("hex");

    if (signature !== expectedSignature) {
        return {
            statusCode: 401,
            body: JSON.stringify({ error: "Unauthorized" }),
        };
    }

    // Process event
    console.log(`Received ${body.event} event`);

    // Store in DynamoDB, trigger SNS, etc.

    return {
        statusCode: 200,
        body: JSON.stringify({ status: "processed" }),
    };
};
```

## Rate Limits

Webhook endpoints have specific limits:

| Action          | Rate Limit |
| --------------- | ---------- |
| Create webhook  | 100/hour   |
| Update webhook  | 500/hour   |
| Test webhook    | 100/hour   |
| List deliveries | 1000/hour  |

## Next Steps

- [API Overview](./overview) - General API information
- [Authentication](./authentication) - API authentication
- [Rate Limiting](./rate-limiting) - Rate limit details
- [Error Handling](./error-handling) - Error responses
- [Security Best Practices](/documentation/v1/security/best-practices) - Security guide
