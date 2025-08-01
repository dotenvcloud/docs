---
title: How DotEnv Works
slug: how-it-works
order: 1
tags: [architecture, overview, concepts]
---

# How DotEnv Works

DotEnv is a secure, centralized environment variable management system. This guide explains the architecture and flow of how your secrets move from storage to your application.

## Architecture Overview

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Your Code     │────▶│   DotEnv API    │────▶│ Encrypted Store │
├─────────────────┤     ├─────────────────┤     ├─────────────────┤
│ • CLI           │     │ • Auth          │     │ • AES-256-GCM   │
│ • SDKs          │     │ • Access Control│     │ • At-rest       │
│ • GitHub Action │     │ • Audit Logs    │     │ • In-transit    │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

## Core Components

### 1. Secret Storage

All secrets are encrypted using industry-standard AES-256-GCM encryption:

```
Original Secret: DATABASE_URL=postgresql://user:pass@host/db
                          ↓
Encrypted: ey0pK2x3M9Z8Q7...vN1mR4a3B6
```

**Encryption Process:**

1. Generate unique 256-bit key per project
2. Create 96-bit nonce for each secret
3. Encrypt value with AES-256-GCM
4. Store encrypted value + nonce + auth tag
5. Key stored separately with additional encryption

### 2. Authentication & Authorization

```
Request Flow:
Client ──(API Key)──▶ Auth Service ──(Token)──▶ API ──(Permissions)──▶ Secrets
```

**Authentication Layers:**

- API Keys: Project or organization scoped
- OAuth 2.0: For web dashboard
- Service Accounts: For CI/CD
- SSO/SAML: Enterprise authentication

### 3. Access Control

Fine-grained permission system:

```
User → Organization → Project → Environment → Secret
         ↓              ↓           ↓           ↓
       Owner         Admin      Developer    Read/Write
```

## Data Flow

### Writing Secrets

```mermaid
sequenceDiagram
    participant CLI
    participant API
    participant Encryption
    participant Database

    CLI->>API: POST /secrets (with value)
    API->>API: Validate permissions
    API->>Encryption: Encrypt value
    Encryption->>Database: Store encrypted
    Database-->>API: Confirmation
    API-->>CLI: Success
```

**Steps:**

1. Client sends secret with authentication
2. API validates user permissions
3. Secret encrypted with project key
4. Encrypted value stored in database
5. Audit log entry created
6. Confirmation returned to client

### Reading Secrets

```mermaid
sequenceDiagram
    participant App
    participant SDK
    participant API
    participant Cache
    participant Database

    App->>SDK: Load secrets
    SDK->>API: GET /secrets
    API->>Cache: Check cache
    Cache-->>API: Miss
    API->>Database: Fetch encrypted
    Database-->>API: Return encrypted
    API->>API: Decrypt
    API->>Cache: Store decrypted
    API-->>SDK: Return secrets
    SDK-->>App: Set env vars
```

**Steps:**

1. Application requests secrets via SDK
2. SDK authenticates with API
3. API checks permissions
4. Encrypted secrets retrieved
5. Decryption happens server-side
6. Secrets returned over TLS
7. SDK populates environment

## Security Layers

### 1. Encryption at Rest

```
Database Storage:
┌────────────────────────────────────┐
│ secrets table                      │
├────────────────────────────────────┤
│ id: uuid                           │
│ key: "DATABASE_URL"                │
│ value: <encrypted_blob>            │
│ nonce: <random_96_bit>             │
│ tag: <auth_tag>                    │
│ version: 1                         │
│ created_at: timestamp              │
└────────────────────────────────────┘
```

### 2. Encryption in Transit

All API communication uses TLS 1.3:

- Certificate pinning available
- Perfect forward secrecy
- No deprecated ciphers

### 3. Key Management

```
Master Key (AWS KMS)
    └── Organization Key
            └── Project Key
                    └── Secret Encryption
```

**Key Rotation:**

- Automatic rotation available
- Zero-downtime rotation
- Maintains version history

## Client Operations

### CLI Workflow

```bash
# 1. Authentication
$ dotenv login
Browser opened → OAuth flow → Token stored

# 2. Project Selection
$ dotenv use my-app
Context set to project: my-app

# 3. Secret Management
$ dotenv secrets set API_KEY=value
Encrypted and stored

# 4. Secret Retrieval
$ dotenv secrets pull > .env
Decrypted and written
```

### SDK Workflow

```javascript
// 1. Initialize
const dotenv = new DotEnv({
    apiKey: "your-api-key",
    project: "my-app",
});

// 2. Load secrets
await dotenv.load();
// Makes API call, decrypts, sets process.env

// 3. Access secrets
console.log(process.env.DATABASE_URL);
// Already decrypted and available
```

### GitHub Action Workflow

```yaml
- uses: dotenv/actions@v1
  with:
      api-key: ${{ secrets.DOTENV_API_KEY }}
# Action performs:
# 1. Authenticate with API
# 2. Fetch encrypted secrets
# 3. Decrypt server-side
# 4. Inject into runner environment
```

## Caching Strategy

### Server-Side Cache

```
API Server
├── L1 Cache (Memory) - 5 min TTL
├── L2 Cache (Redis) - 60 min TTL
└── Database (Source of truth)
```

### Client-Side Cache

```javascript
// SDKs implement smart caching
const cache = new Map();
const TTL = 5 * 60 * 1000; // 5 minutes

async function getSecrets() {
    if (cache.has("secrets") && cache.get("expires") > Date.now()) {
        return cache.get("secrets");
    }

    const secrets = await fetchFromAPI();
    cache.set("secrets", secrets);
    cache.set("expires", Date.now() + TTL);
    return secrets;
}
```

## Audit Trail

Every operation is logged:

```json
{
    "id": "evt_2x4y6z8a",
    "action": "secret.updated",
    "actor": {
        "id": "usr_1a2b3c4d",
        "email": "alice@example.com",
        "ip": "203.0.113.45"
    },
    "resource": {
        "type": "secret",
        "id": "sec_5e6f7g8h",
        "key": "DATABASE_URL",
        "project": "my-app",
        "environment": "production"
    },
    "timestamp": "2024-01-15T10:30:00Z",
    "changes": {
        "before": "<encrypted>",
        "after": "<encrypted>"
    }
}
```

## Performance Optimization

### 1. Parallel Loading

```javascript
// Load multiple projects concurrently
const [api, web, mobile] = await Promise.all([
    dotenv.load({ project: "api" }),
    dotenv.load({ project: "web" }),
    dotenv.load({ project: "mobile" }),
]);
```

### 2. Selective Loading

```javascript
// Load only specific keys
await dotenv.load({
    keys: ["DATABASE_URL", "REDIS_URL", "API_KEY"],
});
```

### 3. Lazy Loading

```javascript
// Load on first access
const secrets = new Proxy(
    {},
    {
        get: async (target, prop) => {
            if (!target[prop]) {
                await dotenv.loadKey(prop);
            }
            return target[prop];
        },
    },
);
```

## Disaster Recovery

### Backup Strategy

```
Primary Region (us-east-1)
    ├── Synchronous replication → Secondary (us-west-2)
    └── Async backup → Cold storage (S3)
```

### Recovery Procedures

1. **Automatic Failover**: < 30 seconds
2. **Manual Failover**: < 5 minutes
3. **Full Restoration**: < 1 hour

## Integration Points

### 1. Development Tools

- VS Code Extension
- IntelliJ Plugin
- Terminal Integration

### 2. CI/CD Platforms

- GitHub Actions
- GitLab CI
- Jenkins
- CircleCI
- Travis CI

### 3. Cloud Providers

- AWS Secrets Manager sync
- Azure Key Vault sync
- Google Secret Manager sync
- Kubernetes Secrets

### 4. Monitoring

- Datadog integration
- New Relic integration
- Custom webhooks

## Best Practices

### 1. Minimize Secret Exposure

```javascript
// ❌ Bad: Logging secrets
console.log(process.env);

// ✅ Good: Selective access
const dbUrl = process.env.DATABASE_URL;
// Use dbUrl without logging
```

### 2. Use Least Privilege

```bash
# ❌ Bad: Broad access
dotenv api-keys create --scopes "*"

# ✅ Good: Specific access
dotenv api-keys create \
  --scopes "secrets:read" \
  --projects "web-app" \
  --environments "production"
```

### 3. Regular Rotation

```bash
# Set up automatic rotation
dotenv secrets rotate-schedule \
  --key API_KEY \
  --interval 90d \
  --notify ops@example.com
```

## Common Questions

### Q: Where are secrets stored?

A: Encrypted in highly available databases across multiple regions with automatic backups.

### Q: Can DotEnv employees see my secrets?

A: No. Secrets are encrypted with keys that only you control. We cannot decrypt your secrets.

### Q: What happens if DotEnv is down?

A: SDKs include fallback mechanisms and caching. Your applications continue running with cached values.

### Q: How fast is secret retrieval?

A: Average latency:

- Cached: < 1ms
- API call: < 50ms globally
- Cold start: < 200ms

## Next Steps

- [Organizations & Projects](./organizations-projects) - Structure your secrets
- [Environments](./environments-concept) - Manage multiple configurations
- [Security Model](./security-model) - Deep dive into security
- [API Reference](/documentation/v1/api/overview) - Direct API usage
