---
title: Key Concepts
slug: key-concepts
order: 7
tags: [concepts, fundamentals, overview]
---

# Key Concepts

This guide covers essential concepts that will help you understand and effectively use DotEnv throughout your development workflow.

## Core Terminology

### Secret

A **secret** is any sensitive piece of information that your application needs but should never be exposed publicly:

- Database passwords
- API keys
- Encryption keys
- OAuth tokens
- Certificates

```bash
# Example secrets
DATABASE_URL="postgresql://user:password@host/db"
STRIPE_API_KEY="sk_live_..."
JWT_SECRET="your-256-bit-secret"
```

### Environment Variable

An **environment variable** is a key-value pair available to your application at runtime:

- Can be sensitive (secret) or non-sensitive
- Accessed via `process.env` in Node.js
- Available to all child processes

```javascript
// Accessing environment variables
const port = process.env.PORT || 3000;
const dbUrl = process.env.DATABASE_URL;
```

### Project

A **project** is a container for related secrets and configurations:

- Represents a single application or service
- Contains multiple environments
- Has its own access controls
- Maintains separate audit logs

```
my-app/
├── environments/
│   ├── development/
│   ├── staging/
│   └── production/
└── settings/
```

### Environment

An **environment** is an isolated namespace within a project:

- Contains its own set of secrets
- Represents different deployment contexts
- Can inherit from other environments
- Examples: development, staging, production

### Organization

An **organization** is the top-level container:

- Groups related projects
- Manages team members
- Handles billing
- Enforces security policies

## Advanced Concepts

### Secret Versioning

Every secret change creates a new version:

```bash
# Version history
v3 (current):  "sk_live_new_key_2024"
v2:            "sk_live_old_key_2023"
v1:            "sk_test_initial_key"
```

Benefits:

- Rollback capability
- Audit trail
- Zero-downtime rotation
- Historical analysis

### Secret Rotation

The process of replacing secrets with new values:

```bash
# Manual rotation
dotenv secrets rotate API_KEY --project my-app

# Scheduled rotation
dotenv secrets rotate DATABASE_PASSWORD \
  --schedule "0 0 1 * *" \
  --project my-app
```

Types:

- **Immediate**: Old key invalid immediately
- **Gradual**: Both keys valid during transition
- **Scheduled**: Automatic rotation on schedule

### Inheritance

Environments can inherit secrets from parent environments:

```yaml
base:
    LOG_LEVEL: "info"
    API_VERSION: "v1"

development (inherits from base):
    LOG_LEVEL: "debug" # Override
    API_VERSION: "v1" # Inherited
    DEBUG: "true" # New
```

### Secret Injection

Methods of getting secrets into your application:

1. **Runtime Injection**

```bash
dotenv run -- node app.js
```

2. **Build-time Injection**

```dockerfile
RUN dotenv secrets pull > .env
```

3. **SDK Loading**

```javascript
await dotenv.load();
```

4. **File Generation**

```bash
dotenv secrets pull > .env
```

### Access Token vs API Key

**Access Token**:

- Short-lived (minutes to hours)
- User-specific
- Obtained via OAuth flow
- Used for web/mobile apps

**API Key**:

- Long-lived (months to years)
- Service/project specific
- Generated manually
- Used for server-to-server

### Encryption Key Management

DotEnv uses hierarchical key management:

```
Master Key (KMS)
    └── Organization Key
            └── Project Key
                    └── Secret Encryption
```

Types:

- **Client-Managed**: You control the key
- **Server-Managed**: DotEnv manages the key
- **Hybrid**: Double encryption with both

### Zero-Knowledge Architecture

In zero-knowledge mode:

- Secrets encrypted client-side
- DotEnv never sees plaintext
- You manage encryption keys
- Maximum security, more complexity

```javascript
// Client-side encryption
const encrypted = encrypt(secret, clientKey);
await dotenv.store(encrypted);
```

### Secret Sprawl

The proliferation of secrets across systems:

- Multiple `.env` files
- Hardcoded values
- Scattered configuration
- Inconsistent management

DotEnv prevents sprawl by centralizing secret management.

### Environment Parity

Keeping environments as similar as possible:

- Same secret keys across environments
- Different values per environment
- Consistent configuration structure
- Reduces deployment surprises

```yaml
# Good parity
development:
    DATABASE_URL: "postgres://localhost/dev"
    CACHE_ENABLED: "false"

production:
    DATABASE_URL: "postgres://prod.aws/app"
    CACHE_ENABLED: "true"
```

### Secret Lifecycle

The complete journey of a secret:

1. **Creation**: Generated or set
2. **Storage**: Encrypted and stored
3. **Distribution**: Securely delivered
4. **Usage**: Accessed by application
5. **Rotation**: Periodically updated
6. **Revocation**: Invalidated when needed
7. **Deletion**: Permanently removed

### Compliance Scope

Areas covered by DotEnv compliance:

- **Data Protection**: Encryption, access control
- **Audit Trail**: Complete activity logging
- **Access Management**: RBAC, MFA, SSO
- **Data Residency**: Regional storage
- **Retention Policies**: Automatic cleanup
- **Right to Deletion**: GDPR compliance

### Service Mesh Integration

DotEnv as a secret provider for service meshes:

```yaml
# Kubernetes secret sync
apiVersion: v1
kind: Secret
metadata:
    name: app-secrets
type: Opaque
data:
    secrets: { { dotenv.secrets | b64encode } }
```

### Multi-Cloud Strategy

Using DotEnv across cloud providers:

```
DotEnv
├── AWS Secrets Manager (sync)
├── Azure Key Vault (sync)
├── Google Secret Manager (sync)
└── Kubernetes Secrets (sync)
```

### Disaster Recovery

Ensuring secret availability:

- **Backups**: Automated encrypted backups
- **Replication**: Multi-region storage
- **Failover**: Automatic region switching
- **Recovery**: Point-in-time restoration

### Rate Limiting

API protection mechanisms:

```yaml
rate_limits:
    anonymous: 100/hour
    authenticated: 1000/hour
    api_key: 10000/hour
    burst: 2x normal limit
```

### Secret Classification

Categorizing secrets by sensitivity:

| Level    | Description       | Examples            | Requirements            |
| -------- | ----------------- | ------------------- | ----------------------- |
| Critical | Business critical | Payment keys, PII   | MFA, approval, rotation |
| High     | Important         | API keys, passwords | MFA, rotation           |
| Medium   | Standard          | Webhook secrets     | Standard security       |
| Low      | Non-critical      | Feature flags       | Basic security          |

### Approval Workflows

Requiring authorization for sensitive changes:

```yaml
approval_required:
    - action: secrets:write:production
      approvers: [admin, team-lead]
      timeout: 24h

    - action: secrets:delete
      approvers: [owner]
      timeout: 1h
```

### Secret Dependencies

Managing related secrets:

```yaml
dependencies:
    DATABASE_URL:
        requires: [DB_PASSWORD, DB_HOST, DB_PORT]
        format: "postgres://{user}:{DB_PASSWORD}@{DB_HOST}:{DB_PORT}/db"

    REDIS_CLUSTER:
        requires: [REDIS_NODES, REDIS_PASSWORD]
        format: "redis://:{REDIS_PASSWORD}@{REDIS_NODES}"
```

### Deployment Strategies

How secrets are deployed:

1. **Blue-Green**: Two identical environments
2. **Canary**: Gradual rollout
3. **Rolling**: Sequential updates
4. **Recreate**: Stop and start

### Monitoring & Alerting

Tracking secret usage:

```yaml
alerts:
    - metric: secret_access_rate
      threshold: 1000/min
      action: notify

    - metric: failed_auth_attempts
      threshold: 10/min
      action: block

    - metric: secret_age
      threshold: 90d
      action: rotation_reminder
```

## Common Patterns

### The Twelve-Factor App

DotEnv aligns with [12-factor](https://12factor.net) principles:

**III. Config**: Store config in environment

```bash
# Separate config from code
dotenv run -- node app.js
```

**X. Dev/prod parity**: Keep environments similar

```bash
# Same keys, different values
dotenv secrets pull --environment development
dotenv secrets pull --environment production
```

### GitOps Integration

```yaml
# .gitops/secrets.yml
apiVersion: dotenv/v1
kind: SecretSync
metadata:
    name: app-secrets
spec:
    project: my-app
    environment: production
    target:
        namespace: default
        name: app-secrets
```

### Infrastructure as Code

```hcl
# terraform/secrets.tf
resource "dotenv_project" "app" {
  name = "my-app"
}

resource "dotenv_secret" "database" {
  project = dotenv_project.app.id
  key     = "DATABASE_URL"
  value   = aws_db_instance.main.connection_string
}
```

### CI/CD Pipeline Integration

```yaml
# .github/workflows/deploy.yml
steps:
    - uses: dotenv/actions@v1
      with:
          api-key: ${{ secrets.DOTENV_API_KEY }}

    - run: |
          # Secrets now in environment
          npm test
          npm run build
          npm run deploy
```

## Anti-Patterns to Avoid

### 1. Secret Sprawl

❌ Multiple `.env` files scattered across repos
✅ Centralized management in DotEnv

### 2. Hardcoded Values

❌ `const API_KEY = "sk_live_123";`
✅ `const API_KEY = process.env.API_KEY;`

### 3. Committed Secrets

❌ `.env` files in version control
✅ `.env.example` with dummy values

### 4. Shared Credentials

❌ One database password for all developers
✅ Individual access with audit trails

### 5. Never Rotating

❌ Same passwords for years
✅ Regular rotation schedule

### 6. Over-Permissioning

❌ Everyone has production access
✅ Least privilege principle

## Next Steps

- [Best Practices](/documentation/v1/guides/best-practices) - Detailed recommendations
- [Security Checklist](/documentation/v1/security/checklist) - Security audit guide
- [Architecture Patterns](/documentation/v1/guides/architecture) - Design patterns
- [Troubleshooting](/documentation/v1/guides/troubleshooting) - Common issues
