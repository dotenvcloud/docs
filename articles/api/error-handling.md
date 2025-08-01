---
title: Error Handling
slug: error-handling
order: 8
tags: [api, errors, troubleshooting, debugging]
---

# Error Handling

Understanding and properly handling API errors is crucial for building robust applications. This guide covers error formats, common errors, handling strategies, and debugging techniques.

## Error Response Format

All API errors follow a consistent JSON structure:

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
        "request_id": "req_xyz789abc123",
        "timestamp": "2024-01-15T10:30:00Z",
        "documentation_url": "https://docs.dotenv.cloud/errors/validation"
    }
}
```

### Error Object Fields

- `code`: Machine-readable error code
- `message`: Human-readable error description
- `details`: Additional context or field-specific errors
- `meta.request_id`: Unique identifier for debugging
- `meta.timestamp`: When the error occurred
- `meta.documentation_url`: Link to relevant documentation

## HTTP Status Codes

### 4xx Client Errors

| Status | Meaning              | Common Causes                            |
| ------ | -------------------- | ---------------------------------------- |
| 400    | Bad Request          | Malformed JSON, invalid parameters       |
| 401    | Unauthorized         | Missing or invalid authentication        |
| 403    | Forbidden            | Valid auth but insufficient permissions  |
| 404    | Not Found            | Resource doesn't exist or no access      |
| 409    | Conflict             | Duplicate resource, constraint violation |
| 422    | Unprocessable Entity | Validation errors                        |
| 429    | Too Many Requests    | Rate limit exceeded                      |

### 5xx Server Errors

| Status | Meaning               | Common Causes           |
| ------ | --------------------- | ----------------------- |
| 500    | Internal Server Error | Unexpected server error |
| 502    | Bad Gateway           | Upstream service error  |
| 503    | Service Unavailable   | Maintenance or overload |
| 504    | Gateway Timeout       | Request took too long   |

## Common Error Codes

### Authentication Errors

#### INVALID_API_KEY

```json
{
    "error": {
        "code": "INVALID_API_KEY",
        "message": "The provided API key is invalid",
        "details": {
            "hint": "Check that your API key is correct and active"
        }
    }
}
```

#### EXPIRED_TOKEN

```json
{
    "error": {
        "code": "EXPIRED_TOKEN",
        "message": "Your authentication token has expired",
        "details": {
            "expired_at": "2024-01-15T10:00:00Z",
            "hint": "Refresh your token or authenticate again"
        }
    }
}
```

#### INSUFFICIENT_PERMISSIONS

```json
{
    "error": {
        "code": "INSUFFICIENT_PERMISSIONS",
        "message": "You don't have permission to perform this action",
        "details": {
            "required_permission": "secrets:write",
            "your_permissions": ["secrets:read"],
            "resource": "project:proj_123"
        }
    }
}
```

### Validation Errors

#### VALIDATION_ERROR

```json
{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "The request contains invalid data",
        "details": {
            "key": [
                "The key field is required",
                "The key must be uppercase",
                "The key must not contain spaces"
            ],
            "value": [
                "The value must be encrypted when client_encrypted is true"
            ]
        }
    }
}
```

#### INVALID_INPUT_FORMAT

```json
{
    "error": {
        "code": "INVALID_INPUT_FORMAT",
        "message": "The request body must be valid JSON",
        "details": {
            "received": "text/plain",
            "expected": "application/json",
            "parse_error": "Unexpected token < in JSON at position 0"
        }
    }
}
```

### Resource Errors

#### RESOURCE_NOT_FOUND

```json
{
    "error": {
        "code": "RESOURCE_NOT_FOUND",
        "message": "The requested resource was not found",
        "details": {
            "resource_type": "project",
            "resource_id": "proj_xyz789",
            "hint": "Check that the ID is correct and you have access"
        }
    }
}
```

#### RESOURCE_ALREADY_EXISTS

```json
{
    "error": {
        "code": "RESOURCE_ALREADY_EXISTS",
        "message": "A resource with this identifier already exists",
        "details": {
            "resource_type": "secret",
            "conflicting_field": "key",
            "conflicting_value": "DATABASE_URL",
            "environment": "production",
            "hint": "Use PUT to update existing resources"
        }
    }
}
```

### Business Logic Errors

#### QUOTA_EXCEEDED

```json
{
    "error": {
        "code": "QUOTA_EXCEEDED",
        "message": "Your plan's quota has been exceeded",
        "details": {
            "quota_type": "projects",
            "limit": 10,
            "current": 10,
            "upgrade_url": "https://dotenv.cloud/pricing"
        }
    }
}
```

#### OPERATION_NOT_ALLOWED

```json
{
    "error": {
        "code": "OPERATION_NOT_ALLOWED",
        "message": "This operation is not allowed in the current context",
        "details": {
            "reason": "Cannot delete the last owner of an organization",
            "hint": "Add another owner before removing this one"
        }
    }
}
```

## Error Handling Strategies

### 1. Comprehensive Error Handling

```javascript
class ApiClient {
    async request(method, path, data) {
        try {
            const response = await fetch(`${BASE_URL}${path}`, {
                method,
                headers: {
                    Authorization: `Bearer ${this.apiKey}`,
                    "Content-Type": "application/json",
                },
                body: data ? JSON.stringify(data) : undefined,
            });

            if (!response.ok) {
                await this.handleErrorResponse(response);
            }

            return await response.json();
        } catch (error) {
            if (error instanceof ApiError) {
                throw error;
            }

            // Network or other errors
            throw new NetworkError("Failed to connect to API", error);
        }
    }

    async handleErrorResponse(response) {
        const contentType = response.headers.get("content-type");

        if (contentType?.includes("application/json")) {
            const errorData = await response.json();
            throw new ApiError(
                errorData.error.code,
                errorData.error.message,
                errorData.error.details,
                response.status,
            );
        }

        // Non-JSON error response
        const text = await response.text();
        throw new ApiError(
            "UNKNOWN_ERROR",
            `Server returned ${response.status}: ${text}`,
            null,
            response.status,
        );
    }
}
```

### 2. Error Classes

```javascript
class ApiError extends Error {
    constructor(code, message, details, status) {
        super(message);
        this.name = "ApiError";
        this.code = code;
        this.details = details;
        this.status = status;
    }

    isValidationError() {
        return this.code === "VALIDATION_ERROR";
    }

    isAuthError() {
        return [401, 403].includes(this.status);
    }

    isRateLimitError() {
        return this.status === 429;
    }

    isServerError() {
        return this.status >= 500;
    }
}

class NetworkError extends Error {
    constructor(message, cause) {
        super(message);
        this.name = "NetworkError";
        this.cause = cause;
    }
}
```

### 3. Retry Logic

```javascript
async function retryableRequest(fn, options = {}) {
    const {
        maxRetries = 3,
        retryDelay = 1000,
        shouldRetry = defaultShouldRetry,
        onRetry = () => {},
    } = options;

    let lastError;

    for (let attempt = 0; attempt <= maxRetries; attempt++) {
        try {
            return await fn();
        } catch (error) {
            lastError = error;

            if (attempt === maxRetries || !shouldRetry(error, attempt)) {
                throw error;
            }

            const delay = retryDelay * Math.pow(2, attempt);
            onRetry(error, attempt + 1, delay);

            await new Promise((resolve) => setTimeout(resolve, delay));
        }
    }

    throw lastError;
}

function defaultShouldRetry(error, attempt) {
    // Retry on network errors
    if (error instanceof NetworkError) return true;

    // Retry on 5xx errors
    if (error instanceof ApiError && error.isServerError()) return true;

    // Retry on specific 429 rate limit errors
    if (error instanceof ApiError && error.isRateLimitError()) {
        return attempt < 2; // Only retry rate limits twice
    }

    // Don't retry client errors
    return false;
}
```

### 4. Error Recovery

```javascript
class ResilientApiClient {
    async getSecretWithFallback(projectId, key, environment) {
        try {
            // Try primary method
            return await this.getSecret(projectId, key, environment);
        } catch (error) {
            if (error.code === "RESOURCE_NOT_FOUND") {
                // Try fallback environment
                if (environment === "production") {
                    console.warn("Production secret not found, trying staging");
                    return await this.getSecret(projectId, key, "staging");
                }
            }

            if (error.isServerError()) {
                // Return cached value if available
                const cached = this.cache.get(
                    `${projectId}:${key}:${environment}`,
                );
                if (cached) {
                    console.warn("API error, using cached value");
                    return cached;
                }
            }

            throw error;
        }
    }
}
```

## Validation Error Handling

### Field-Level Errors

```javascript
function handleValidationErrors(error) {
    if (!error.isValidationError()) return;

    const fieldErrors = error.details;

    // Display errors next to form fields
    Object.entries(fieldErrors).forEach(([field, messages]) => {
        const fieldElement = document.querySelector(`[name="${field}"]`);
        const errorElement = document.querySelector(
            `[data-error-for="${field}"]`,
        );

        if (fieldElement) {
            fieldElement.classList.add("error");
        }

        if (errorElement) {
            errorElement.textContent = messages.join(", ");
            errorElement.classList.remove("hidden");
        }
    });
}
```

### React Example

```jsx
function SecretForm() {
    const [errors, setErrors] = useState({});

    const handleSubmit = async (data) => {
        try {
            setErrors({});
            await api.secrets.create(data);
        } catch (error) {
            if (error.isValidationError()) {
                setErrors(error.details);
            } else {
                // Handle other errors
                showNotification({
                    type: "error",
                    message: error.message,
                });
            }
        }
    };

    return (
        <form onSubmit={handleSubmit}>
            <div>
                <input name="key" />
                {errors.key && (
                    <span className="error">{errors.key.join(", ")}</span>
                )}
            </div>
        </form>
    );
}
```

## Error Monitoring

### Logging Errors

```javascript
class ErrorLogger {
    log(error, context = {}) {
        const errorData = {
            timestamp: new Date().toISOString(),
            error: {
                name: error.name,
                message: error.message,
                code: error.code,
                status: error.status,
                stack: error.stack,
            },
            context,
            user: this.getCurrentUser(),
            request: this.getRequestContext(),
        };

        // Log locally
        console.error("API Error:", errorData);

        // Send to monitoring service
        if (this.shouldReportError(error)) {
            this.reportToSentry(errorData);
        }
    }

    shouldReportError(error) {
        // Don't report client errors except 403s
        if (error.status >= 400 && error.status < 500 && error.status !== 403) {
            return false;
        }

        // Always report server errors
        if (error.status >= 500) {
            return true;
        }

        // Report unexpected errors
        return true;
    }
}
```

### Error Metrics

```javascript
class ErrorMetrics {
    constructor() {
        this.errors = new Map();
    }

    track(error) {
        const key = `${error.code}:${error.status}`;
        const current = this.errors.get(key) || {
            count: 0,
            firstSeen: new Date(),
            lastSeen: null,
        };

        current.count++;
        current.lastSeen = new Date();

        this.errors.set(key, current);
    }

    getReport() {
        const report = {};

        this.errors.forEach((data, key) => {
            const [code, status] = key.split(":");
            report[key] = {
                code,
                status: parseInt(status),
                ...data,
                rate:
                    data.count / ((data.lastSeen - data.firstSeen) / 1000 / 60),
            };
        });

        return report;
    }
}
```

## Debugging Errors

### Request ID Tracking

```javascript
// Include request ID in all requests
const requestId = generateRequestId();

const response = await fetch(url, {
    headers: {
        "X-Request-ID": requestId,
        Authorization: `Bearer ${apiKey}`,
    },
});

// Log with request ID
console.log(`Request ${requestId} failed:`, error);
```

### Detailed Error Information

```javascript
class DetailedApiError extends ApiError {
    constructor(response, errorData) {
        super(
            errorData.error.code,
            errorData.error.message,
            errorData.error.details,
            response.status,
        );

        this.requestId = errorData.meta?.request_id;
        this.timestamp = errorData.meta?.timestamp;
        this.documentationUrl = errorData.meta?.documentation_url;
        this.responseHeaders = Object.fromEntries(response.headers);
    }

    getDebugInfo() {
        return {
            requestId: this.requestId,
            timestamp: this.timestamp,
            status: this.status,
            code: this.code,
            message: this.message,
            details: this.details,
            headers: this.responseHeaders,
            documentation: this.documentationUrl,
        };
    }
}
```

## Common Patterns

### Global Error Handler

```javascript
// Axios example
axios.interceptors.response.use(
    (response) => response,
    (error) => {
        if (error.response) {
            // API returned an error
            const apiError = new ApiError(
                error.response.data.error.code,
                error.response.data.error.message,
                error.response.data.error.details,
                error.response.status,
            );

            // Handle specific errors globally
            if (apiError.status === 401) {
                // Redirect to login
                window.location.href = "/login";
            }

            throw apiError;
        }

        // Network error
        throw new NetworkError("Network request failed", error);
    },
);
```

### Error Boundaries (React)

```jsx
class ApiErrorBoundary extends React.Component {
    state = { hasError: false, error: null };

    static getDerivedStateFromError(error) {
        return { hasError: true, error };
    }

    componentDidCatch(error, errorInfo) {
        if (error instanceof ApiError) {
            console.error("API Error:", error.getDebugInfo());

            // Log to error reporting service
            errorReporter.logError(error, {
                componentStack: errorInfo.componentStack,
            });
        }
    }

    render() {
        if (this.state.hasError) {
            const error = this.state.error;

            if (error instanceof ApiError) {
                if (error.status === 503) {
                    return <MaintenancePage />;
                }

                if (error.isServerError()) {
                    return <ServerErrorPage error={error} />;
                }
            }

            return <GenericErrorPage />;
        }

        return this.props.children;
    }
}
```

## Best Practices

### 1. Always Handle Errors

Never ignore errors or use empty catch blocks:

```javascript
// Bad
try {
    await api.secrets.create(data);
} catch (e) {
    // Silent failure
}

// Good
try {
    await api.secrets.create(data);
} catch (error) {
    logger.error("Failed to create secret:", error);

    if (error.isValidationError()) {
        // Show validation errors to user
        showValidationErrors(error.details);
    } else {
        // Show generic error message
        showErrorNotification("Failed to create secret. Please try again.");
    }

    // Re-throw if needed
    throw error;
}
```

### 2. Provide Context

Include relevant context when logging errors:

```javascript
try {
    await api.secrets.update(secretId, data);
} catch (error) {
    logger.error("Failed to update secret", {
        error,
        secretId,
        projectId: data.projectId,
        environment: data.environment,
        userId: currentUser.id,
    });
    throw error;
}
```

### 3. User-Friendly Messages

Translate technical errors to user-friendly messages:

```javascript
function getUserMessage(error) {
    const messages = {
        INVALID_API_KEY: "Your session has expired. Please log in again.",
        QUOTA_EXCEEDED: "You've reached your plan limit. Upgrade to continue.",
        RESOURCE_NOT_FOUND: "The item you're looking for doesn't exist.",
        VALIDATION_ERROR: "Please check your input and try again.",
        RATE_LIMIT_EXCEEDED: "Too many requests. Please wait a moment.",
    };

    return messages[error.code] || "Something went wrong. Please try again.";
}
```

### 4. Graceful Degradation

Provide fallback behavior when possible:

```javascript
async function getProjectWithFallback(projectId) {
    try {
        // Try to get fresh data
        return await api.projects.get(projectId);
    } catch (error) {
        // Check cache
        const cached = cache.get(`project:${projectId}`);
        if (cached) {
            showWarning("Using cached data. Some information may be outdated.");
            return cached;
        }

        // Return minimal data
        if (error.code === "RESOURCE_NOT_FOUND") {
            return null;
        }

        // Can't recover
        throw error;
    }
}
```

## Testing Error Handling

### Unit Tests

```javascript
describe("API Error Handling", () => {
    it("should handle validation errors", async () => {
        const mockResponse = {
            status: 422,
            json: async () => ({
                error: {
                    code: "VALIDATION_ERROR",
                    message: "Invalid input",
                    details: {
                        key: ["Required field"],
                    },
                },
            }),
        };

        fetch.mockResolvedValueOnce(mockResponse);

        await expect(api.secrets.create({})).rejects.toThrow(ApiError);

        try {
            await api.secrets.create({});
        } catch (error) {
            expect(error.isValidationError()).toBe(true);
            expect(error.details.key).toContain("Required field");
        }
    });
});
```

### Integration Tests

```javascript
it("should retry on server errors", async () => {
    let attempts = 0;

    server.use(
        rest.post("/api/v1/secrets", (req, res, ctx) => {
            attempts++;

            if (attempts < 3) {
                return res(
                    ctx.status(500),
                    ctx.json({
                        error: {
                            code: "INTERNAL_ERROR",
                            message: "Server error",
                        },
                    }),
                );
            }

            return res(ctx.json({ data: { id: "sec_123" } }));
        }),
    );

    const result = await retryableRequest(() => api.secrets.create(data));

    expect(attempts).toBe(3);
    expect(result.data.id).toBe("sec_123");
});
```

## Next Steps

- [API Overview](./overview) - General API information
- [Authentication](./authentication) - Authentication errors
- [Rate Limiting](./rate-limiting) - Rate limit errors
- [Webhooks](/documentation/v1/drafts/api/webhooks) *(Coming Soon)* - Webhook error handling
- [Best Practices](/documentation/v1/best-practices) - General best practices
