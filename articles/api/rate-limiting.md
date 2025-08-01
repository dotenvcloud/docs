---
title: Rate Limiting
slug: rate-limiting
order: 7
tags: [api, rate-limiting, throttling, performance]
---

# Rate Limiting

The DotEnv API implements rate limiting to ensure fair usage and maintain performance for all users. This guide covers rate limits, headers, strategies, and best practices.

## Overview

Rate limiting prevents API abuse and ensures service availability by restricting the number of requests a client can make within a specific time window.

### Default Limits

| Plan         | Requests per Hour | Burst Limit | Concurrent Requests |
| ------------ | ----------------- | ----------- | ------------------- |
| Free         | 1,000             | 100/min     | 10                  |
| Starter      | 5,000             | 500/min     | 25                  |
| Professional | 20,000            | 1,000/min   | 50                  |
| Enterprise   | Custom            | Custom      | Custom              |

### Endpoint-Specific Limits

Some endpoints have stricter limits due to their resource intensity:

| Endpoint                | Rate Limit | Reason                |
| ----------------------- | ---------- | --------------------- |
| `/secrets/decrypt`      | 1,000/hour | Computation intensive |
| `/projects/create`      | 100/hour   | Resource allocation   |
| `/organizations/create` | 10/hour    | Account creation      |
| `/api-keys/create`      | 50/hour    | Security sensitive    |
| `/secrets/bulk`         | 100/hour   | Batch operations      |
| `/webhooks/test`        | 100/hour   | External calls        |

## Rate Limit Headers

All API responses include rate limit information:

```http
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4999
X-RateLimit-Reset: 1642089600
X-RateLimit-Reset-After: 3600
X-RateLimit-Bucket: api_standard
```

### Header Definitions

- `X-RateLimit-Limit`: Maximum requests allowed in the current window
- `X-RateLimit-Remaining`: Requests remaining in the current window
- `X-RateLimit-Reset`: Unix timestamp when the window resets
- `X-RateLimit-Reset-After`: Seconds until the window resets
- `X-RateLimit-Bucket`: Rate limit bucket identifier

### Additional Headers

When approaching limits:

```http
X-RateLimit-Warning: Approaching limit
X-RateLimit-Suggested-Retry: 300
```

## Rate Limit Windows

### Sliding Window

The API uses a sliding window algorithm:

```
Timeline:  |-----|-----|-----|-----|
Requests:    50    75    60    40
Window:         [-----150-----]
                     [-----175-----]
                          [-----160-----]
```

This provides smoother rate limiting compared to fixed windows.

### Window Types

1. **Standard Window** (1 hour)

    - Most endpoints
    - Resets on a rolling basis

2. **Burst Window** (1 minute)

    - Prevents traffic spikes
    - Additional constraint on top of hourly limit

3. **Daily Window** (24 hours)
    - For resource-intensive operations
    - Separate from hourly limits

## Rate Limit Exceeded

### 429 Response

When rate limit is exceeded:

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1642089600
X-RateLimit-Reset-After: 1800
Retry-After: 1800
```

Response body:

```json
{
    "error": {
        "code": "RATE_LIMIT_EXCEEDED",
        "message": "API rate limit exceeded",
        "details": {
            "limit": 5000,
            "window": "1h",
            "reset_at": "2024-01-15T11:00:00Z",
            "reset_in": 1800
        }
    }
}
```

### Retry Strategy

Implement exponential backoff:

```javascript
async function makeRequestWithRetry(url, options, maxRetries = 3) {
    for (let i = 0; i < maxRetries; i++) {
        try {
            const response = await fetch(url, options);

            if (response.status === 429) {
                const resetAfter = response.headers.get(
                    "X-RateLimit-Reset-After",
                );
                const waitTime = resetAfter
                    ? parseInt(resetAfter) * 1000
                    : Math.pow(2, i) * 1000;

                console.log(
                    `Rate limited. Waiting ${waitTime}ms before retry...`,
                );
                await new Promise((resolve) => setTimeout(resolve, waitTime));
                continue;
            }

            return response;
        } catch (error) {
            if (i === maxRetries - 1) throw error;
        }
    }
}
```

## Rate Limit Strategies

### 1. Request Batching

Combine multiple operations:

```javascript
// Bad - Multiple requests
for (const secret of secrets) {
    await api.secrets.create(secret);
}

// Good - Batch request
await api.secrets.bulkCreate(secrets);
```

### 2. Caching

Cache responses when appropriate:

```javascript
const cache = new Map();

async function getCachedProject(projectId) {
    const cacheKey = `project:${projectId}`;
    const cached = cache.get(cacheKey);

    if (cached && cached.expires > Date.now()) {
        return cached.data;
    }

    const project = await api.projects.get(projectId);
    cache.set(cacheKey, {
        data: project,
        expires: Date.now() + 300000, // 5 minutes
    });

    return project;
}
```

### 3. Request Queuing

Queue requests to stay within limits:

```javascript
class RateLimitedQueue {
    constructor(limit, window) {
        this.limit = limit;
        this.window = window;
        this.queue = [];
        this.processing = false;
        this.requests = [];
    }

    async add(request) {
        return new Promise((resolve, reject) => {
            this.queue.push({ request, resolve, reject });
            this.process();
        });
    }

    async process() {
        if (this.processing) return;
        this.processing = true;

        while (this.queue.length > 0) {
            // Remove old requests from tracking
            const cutoff = Date.now() - this.window;
            this.requests = this.requests.filter((time) => time > cutoff);

            // Check if we can make a request
            if (this.requests.length < this.limit) {
                const { request, resolve, reject } = this.queue.shift();
                this.requests.push(Date.now());

                try {
                    const result = await request();
                    resolve(result);
                } catch (error) {
                    reject(error);
                }
            } else {
                // Wait before checking again
                await new Promise((resolve) => setTimeout(resolve, 1000));
            }
        }

        this.processing = false;
    }
}
```

### 4. Monitoring Usage

Track your API usage:

```javascript
class ApiClient {
    constructor(apiKey) {
        this.apiKey = apiKey;
        this.metrics = {
            requests: 0,
            remaining: null,
            resetAt: null,
        };
    }

    async request(method, path, data) {
        const response = await fetch(`${BASE_URL}${path}`, {
            method,
            headers: {
                Authorization: `Bearer ${this.apiKey}`,
                "Content-Type": "application/json",
            },
            body: data ? JSON.stringify(data) : undefined,
        });

        // Update metrics
        this.metrics.requests++;
        this.metrics.remaining = parseInt(
            response.headers.get("X-RateLimit-Remaining"),
        );
        this.metrics.resetAt = parseInt(
            response.headers.get("X-RateLimit-Reset"),
        );

        // Warn when approaching limit
        if (this.metrics.remaining < 100) {
            console.warn(
                `API rate limit warning: ${this.metrics.remaining} requests remaining`,
            );
        }

        return response;
    }

    getRateLimitStatus() {
        const resetIn = this.metrics.resetAt - Math.floor(Date.now() / 1000);
        return {
            used: this.metrics.requests,
            remaining: this.metrics.remaining,
            resetsIn: resetIn > 0 ? resetIn : 0,
        };
    }
}
```

## Bypass Options

### Enterprise Plans

Enterprise customers can request:

- Higher rate limits
- Dedicated rate limit pools
- Exemption for specific IPs
- Custom rate limit windows

### Development Mode

For development environments:

```http
X-DotEnv-Environment: development
```

This provides relaxed limits:

- 10,000 requests/hour
- 1,000 requests/minute burst

### IP Whitelisting

Whitelisted IPs get enhanced limits:

```bash
curl -X POST https://api.dotenv.cloud/v1/account/whitelist \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "ip_addresses": ["203.0.113.0/24"],
    "reason": "CI/CD servers"
  }'
```

## Best Practices

### 1. Handle Rate Limits Gracefully

```javascript
async function apiCall(endpoint, options) {
    try {
        const response = await fetch(endpoint, options);

        if (response.status === 429) {
            const resetAfter = response.headers.get("X-RateLimit-Reset-After");
            throw new RateLimitError(`Rate limited. Reset in ${resetAfter}s`);
        }

        return response;
    } catch (error) {
        if (error instanceof RateLimitError) {
            // Handle rate limit specifically
            await waitForReset(error.resetTime);
            return apiCall(endpoint, options); // Retry
        }
        throw error;
    }
}
```

### 2. Implement Circuit Breaker

```javascript
class CircuitBreaker {
    constructor(threshold = 5, timeout = 60000) {
        this.threshold = threshold;
        this.timeout = timeout;
        this.failures = 0;
        this.lastFailure = null;
        this.state = "CLOSED"; // CLOSED, OPEN, HALF_OPEN
    }

    async call(fn) {
        if (this.state === "OPEN") {
            if (Date.now() - this.lastFailure > this.timeout) {
                this.state = "HALF_OPEN";
            } else {
                throw new Error("Circuit breaker is OPEN");
            }
        }

        try {
            const result = await fn();
            if (this.state === "HALF_OPEN") {
                this.state = "CLOSED";
                this.failures = 0;
            }
            return result;
        } catch (error) {
            this.failures++;
            this.lastFailure = Date.now();

            if (this.failures >= this.threshold) {
                this.state = "OPEN";
            }

            throw error;
        }
    }
}
```

### 3. Use Webhooks Instead of Polling

Instead of:

```javascript
// Bad - Polling
setInterval(async () => {
    const secrets = await api.secrets.list();
    checkForChanges(secrets);
}, 5000);
```

Use webhooks:

```javascript
// Good - Webhooks
api.webhooks.create({
    url: "https://app.example.com/webhook",
    events: ["secret.updated", "secret.created"],
});
```

### 4. Optimize API Calls

```javascript
// Bad - Multiple calls
const project = await api.projects.get(projectId);
const secrets = await api.secrets.list(projectId);
const members = await api.projects.getMembers(projectId);

// Good - Use includes
const project = await api.projects.get(projectId, {
    include: ["secrets", "members"],
});
```

### 5. Monitor and Alert

```javascript
class RateLimitMonitor {
    constructor(alertThreshold = 0.8) {
        this.alertThreshold = alertThreshold;
    }

    checkHeaders(headers) {
        const limit = parseInt(headers.get("X-RateLimit-Limit"));
        const remaining = parseInt(headers.get("X-RateLimit-Remaining"));
        const usage = (limit - remaining) / limit;

        if (usage > this.alertThreshold) {
            this.sendAlert({
                message: "High API usage detected",
                usage: `${Math.round(usage * 100)}%`,
                remaining,
                limit,
            });
        }
    }

    sendAlert(data) {
        // Send to monitoring service
        console.warn("Rate limit alert:", data);
    }
}
```

## Rate Limit Types

### 1. User Rate Limits

Based on authenticated user:

```http
X-RateLimit-Scope: user
X-RateLimit-Identity: usr_123456
```

### 2. API Key Rate Limits

Based on API key:

```http
X-RateLimit-Scope: api_key
X-RateLimit-Identity: key_abc123
```

### 3. IP Rate Limits

For unauthenticated requests:

```http
X-RateLimit-Scope: ip
X-RateLimit-Identity: 203.0.113.1
```

### 4. Organization Rate Limits

Shared across organization:

```http
X-RateLimit-Scope: organization
X-RateLimit-Identity: org_xyz789
```

## Advanced Features

### Burst Credits

Accumulate unused requests:

```http
X-RateLimit-Burst-Credits: 500
X-RateLimit-Burst-Max: 1000
```

### Priority Queuing

High-priority requests get preferential treatment:

```http
X-DotEnv-Priority: high
```

### Rate Limit Reservation

Reserve capacity for critical operations:

```javascript
const reservation = await api.rateLimits.reserve({
    requests: 100,
    duration: "5m",
    operation: "bulk_import",
});

// Use reservation
await api.secrets.bulkImport(data, {
    headers: {
        "X-RateLimit-Reservation": reservation.token,
    },
});
```

## SDK Support

### JavaScript SDK

```javascript
const client = new DotEnvAPI({
    apiKey: "YOUR_API_KEY",
    rateLimitStrategy: "queue", // 'queue', 'throw', 'wait'
    maxRetries: 3,
    onRateLimit: (error, retryAfter) => {
        console.log(`Rate limited. Retry after ${retryAfter}s`);
    },
});
```

### Python SDK

```python
from dotenv_api import DotEnvAPI

client = DotEnvAPI(
    api_key='YOUR_API_KEY',
    rate_limit_handler='auto',  # 'auto', 'manual', 'raise'
    max_retries=3
)

# Manual handling
try:
    secrets = client.secrets.list()
except RateLimitError as e:
    print(f"Rate limited. Reset at: {e.reset_at}")
    time.sleep(e.reset_after)
    secrets = client.secrets.list()
```

## Troubleshooting

### Common Issues

1. **Sudden 429 errors**

    - Check for loops or repeated calls
    - Verify rate limit headers
    - Check if multiple instances are running

2. **Inconsistent limits**

    - Different endpoints have different limits
    - Burst limits may be hit before hourly limits
    - Organization limits may affect individual users

3. **Reset time confusion**
    - Use `X-RateLimit-Reset-After` for wait time
    - `X-RateLimit-Reset` is Unix timestamp
    - Times are in UTC

### Debugging

Enable verbose logging:

```javascript
const client = new DotEnvAPI({
    apiKey: "YOUR_API_KEY",
    debug: true,
    logRateLimit: true,
});
```

## Next Steps

- [API Overview](./overview) - General API information
- [Error Handling](./error-handling) - Error responses
- [Webhooks](/documentation/v1/drafts/api/webhooks) *(Coming Soon)* - Real-time updates
- [Best Practices](/documentation/v1/best-practices) - API best practices
