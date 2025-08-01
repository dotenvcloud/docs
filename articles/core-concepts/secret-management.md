---
title: Secret Management Best Practices
slug: secret-management
order: 5
tags: [secrets, security, best-practices]
---

# Secret Management Best Practices

Effective secret management is crucial for application security. This guide provides comprehensive best practices for managing secrets with DotEnv.

## What Are Secrets?

Secrets are sensitive pieces of information that your application needs to function but should never be exposed:

- **Credentials**: Database passwords, API keys
- **Certificates**: SSL/TLS certificates, signing keys
- **Tokens**: OAuth tokens, JWT secrets
- **Configuration**: Encryption keys, connection strings

## Secret Lifecycle

### 1. Creation

```bash
# Generate strong secrets
dotenv secrets generate DATABASE_PASSWORD \
  --length 32 \
  --special-chars \
  --project my-app

# Set with validation
dotenv secrets set API_KEY \
  --project my-app \
  --validate-format "sk_[a-zA-Z0-9]{32}" \
  --description "Stripe API key for payment processing"
```

### 2. Storage

Secrets are stored with:

- **Encryption**: AES-256-GCM
- **Access Control**: Role-based permissions
- **Versioning**: History tracking
- **Metadata**: Descriptions, tags, expiration

### 3. Distribution

```javascript
// SDK handles secure distribution
const dotenv = new DotEnv({
    project: "my-app",
    environment: process.env.NODE_ENV,
    cacheTimeout: 300, // 5 minutes
});

await dotenv.load();
```

### 4. Rotation

```bash
# Manual rotation
dotenv secrets rotate DATABASE_PASSWORD \
  --project my-app \
  --notify ops-team

# Scheduled rotation
dotenv secrets schedule-rotation JWT_SECRET \
  --interval 90d \
  --project my-app
```

### 5. Revocation

```bash
# Immediate revocation
dotenv secrets revoke API_KEY \
  --project my-app \
  --reason "Compromised in commit abc123"

# Expire after time
dotenv secrets expire TEMP_TOKEN \
  --after 24h \
  --project my-app
```

## Secret Types & Strategies

### Database Credentials

```bash
# Connection string pattern
DATABASE_URL="postgresql://user:password@host:5432/database?sslmode=require"

# Separate components (more flexible)
DB_HOST="db.example.com"
DB_PORT="5432"
DB_USER="app_user"
DB_PASSWORD="<generated-password>"
DB_NAME="production"
DB_SSL_MODE="require"
```

**Best Practices:**

- Use read-only credentials where possible
- Separate read/write credentials
- Enable SSL/TLS always
- Rotate passwords quarterly

### API Keys

```bash
# External service keys
STRIPE_SECRET_KEY="sk_live_..."
STRIPE_PUBLISHABLE_KEY="pk_live_..."
STRIPE_WEBHOOK_SECRET="whsec_..."

# Internal API keys
INTERNAL_API_KEY="<uuid-v4>"
SERVICE_TO_SERVICE_KEY="<signed-jwt>"
```

**Best Practices:**

- Use separate keys per environment
- Implement key scoping
- Monitor key usage
- Set expiration dates

### Encryption Keys

```bash
# Application encryption
ENCRYPTION_KEY="<base64-encoded-32-bytes>"
ENCRYPTION_SALT="<base64-encoded-16-bytes>"

# JWT signing
JWT_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----..."
JWT_PUBLIC_KEY="-----BEGIN PUBLIC KEY-----..."
JWT_ALGORITHM="RS256"
```

**Best Practices:**

- Generate cryptographically secure keys
- Use appropriate key lengths
- Separate signing and encryption keys
- Implement key rotation

### OAuth Credentials

```bash
# OAuth 2.0 configuration
OAUTH_CLIENT_ID="client_123456"
OAUTH_CLIENT_SECRET="<secret>"
OAUTH_REDIRECT_URI="https://app.example.com/auth/callback"
OAUTH_SCOPE="read:user email"
```

**Best Practices:**

- Store refresh tokens securely
- Implement token refresh logic
- Use PKCE for public clients
- Validate redirect URIs

## Security Patterns

### 1. Principle of Least Privilege

```yaml
# Role-based secret access
roles:
  frontend_dev:
    secrets:
      - NEXT_PUBLIC_* # Only public keys
      - API_ENDPOINT
    deny:
      - *_SECRET
      - *_PRIVATE_KEY

  backend_dev:
    secrets:
      - DATABASE_URL
      - REDIS_URL
      - API_KEYS
    environments:
      - development
      - staging
```

### 2. Defense in Depth

```javascript
// Multiple layers of validation
class SecureConfig {
    constructor() {
        this.validateEnvironment();
        this.validateSecrets();
        this.setupFallbacks();
    }

    validateEnvironment() {
        if (!process.env.NODE_ENV) {
            throw new Error("NODE_ENV not set");
        }
    }

    validateSecrets() {
        const required = ["DATABASE_URL", "JWT_SECRET", "API_KEY"];
        const missing = required.filter((key) => !process.env[key]);

        if (missing.length > 0) {
            throw new Error(`Missing secrets: ${missing.join(", ")}`);
        }
    }

    setupFallbacks() {
        // Non-sensitive defaults only
        process.env.LOG_LEVEL = process.env.LOG_LEVEL || "info";
        process.env.PORT = process.env.PORT || "3000";
    }
}
```

### 3. Secret Scanning Prevention

```bash
# .gitignore
.env
.env.*
!.env.example
*.key
*.pem
*.p12
credentials/
secrets/

# .gitleaks.toml
[rules]
  [[rules]]
  id = "dotenv-api-key"
  description = "DotEnv API Key"
  regex = '''dotenv_api_[a-zA-Z0-9]{32}'''

  [[rules]]
  id = "generic-secret"
  description = "Generic Secret"
  regex = '''(?i)(password|secret|key)\s*=\s*["'][^"']{8,}["']'''
```

### 4. Temporary Secrets

```bash
# Create time-limited secrets
dotenv secrets set TEMP_ACCESS_TOKEN \
  --value "$(uuidgen)" \
  --expires 1h \
  --project my-app

# One-time secrets
dotenv secrets create-one-time \
  --name SETUP_KEY \
  --project my-app
```

## Common Patterns

### Multi-Environment Strategy

```yaml
# Environment-specific secrets
development:
    DATABASE_URL: "postgresql://localhost/dev"
    STRIPE_KEY: "sk_test_..."
    DEBUG: "true"

staging:
    DATABASE_URL: "postgresql://staging.rds/app"
    STRIPE_KEY: "sk_test_..."
    DEBUG: "false"

production:
    DATABASE_URL: "postgresql://prod.rds/app"
    STRIPE_KEY: "sk_live_..."
    DEBUG: "false"
    SENTRY_DSN: "https://..."
```

### Service-to-Service Authentication

```javascript
// Service A: Generate signed request
const jwt = require("jsonwebtoken");

const serviceToken = jwt.sign(
    {
        service: "service-a",
        permissions: ["read:data"],
    },
    process.env.SERVICE_PRIVATE_KEY,
    {
        algorithm: "RS256",
        expiresIn: "5m",
    },
);

// Service B: Verify request
const payload = jwt.verify(serviceToken, process.env.SERVICE_A_PUBLIC_KEY, {
    algorithms: ["RS256"],
});
```

### Configuration Templates

```bash
# Create template
cat > .env.example << EOF
# Application
NODE_ENV=development
PORT=3000

# Database (Required)
DATABASE_URL=

# Redis (Required)
REDIS_URL=

# Authentication (Required)
JWT_SECRET=
SESSION_SECRET=

# External Services (Optional)
STRIPE_KEY=
SENDGRID_KEY=
EOF

# Validate against template
dotenv validate --template .env.example
```

## Secret Rotation Strategies

### Manual Rotation

```bash
#!/bin/bash
# rotate-db-password.sh

# 1. Generate new password
NEW_PASSWORD=$(openssl rand -base64 32)

# 2. Update database
psql -c "ALTER USER app_user PASSWORD '$NEW_PASSWORD';"

# 3. Update DotEnv
dotenv secrets set DATABASE_PASSWORD="$NEW_PASSWORD" \
  --project my-app \
  --environment production

# 4. Trigger application reload
kubectl rollout restart deployment/api
```

### Automated Rotation

```yaml
# rotation-policy.yml
policies:
    - name: database-rotation
      secret: DATABASE_PASSWORD
      schedule: "0 0 1 */3 *" # Every 3 months
      strategy: blue-green
      notifications:
          - email: ops@example.com
          - slack: "#security"

    - name: api-key-rotation
      secret: INTERNAL_API_KEY
      schedule: "0 0 * * 0" # Weekly
      strategy: gradual
      grace_period: 24h
```

### Zero-Downtime Rotation

```javascript
// Dual-key rotation support
class RotatableConfig {
    constructor() {
        this.keys = {
            current: process.env.API_KEY,
            previous: process.env.API_KEY_PREVIOUS,
        };
    }

    async validateApiKey(key) {
        // Accept both current and previous keys
        return key === this.keys.current || key === this.keys.previous;
    }

    async rotateKeys() {
        // Move current to previous
        this.keys.previous = this.keys.current;

        // Set new current
        this.keys.current = await generateNewKey();

        // Update environment
        await updateSecrets({
            API_KEY: this.keys.current,
            API_KEY_PREVIOUS: this.keys.previous,
        });
    }
}
```

## Secret Hygiene

### Regular Audits

```bash
# Find unused secrets
dotenv audit unused-secrets \
  --project my-app \
  --last-accessed "> 90 days"

# Find duplicate values
dotenv audit duplicate-values \
  --project my-app \
  --across-environments

# Check weak secrets
dotenv audit weak-secrets \
  --project my-app \
  --check-entropy \
  --check-patterns
```

### Secret Classification

```yaml
# secret-classification.yml
classifications:
    critical:
        description: "Business critical secrets"
        examples: ["DATABASE_PASSWORD", "PAYMENT_PRIVATE_KEY"]
        requirements:
            rotation_interval: 90d
            mfa_required: true
            approval_required: true

    high:
        description: "Important secrets"
        examples: ["API_KEYS", "JWT_SECRET"]
        requirements:
            rotation_interval: 180d
            mfa_required: true

    medium:
        description: "Standard secrets"
        examples: ["SMTP_PASSWORD", "ANALYTICS_KEY"]
        requirements:
            rotation_interval: 365d

    low:
        description: "Non-critical secrets"
        examples: ["WEBHOOK_SECRET", "FEATURE_FLAGS"]
        requirements:
            rotation_interval: optional
```

### Documentation

```bash
# Document each secret
dotenv secrets document DATABASE_URL \
  --description "PostgreSQL connection string for main database" \
  --format "postgresql://[user]:[password]@[host]:[port]/[database]" \
  --example "postgresql://app:pass@db.example.com:5432/myapp" \
  --required true \
  --classification critical

# Generate documentation
dotenv secrets export-docs \
  --project my-app \
  --format markdown \
  > secrets-documentation.md
```

## Common Mistakes to Avoid

### 1. Hardcoding Secrets

```javascript
// ❌ Never do this
const apiKey = "sk_live_1234567890abcdef";

// ✅ Always use environment variables
const apiKey = process.env.API_KEY;
if (!apiKey) {
    throw new Error("API_KEY not configured");
}
```

### 2. Logging Secrets

```javascript
// ❌ Don't log sensitive values
console.log("Config:", process.env);
logger.info(`Connecting to ${databaseUrl}`);

// ✅ Log safely
console.log("Config loaded successfully");
logger.info("Connecting to database");
```

### 3. Committing Secrets

```bash
# ❌ Don't track .env files
git add .env
git commit -m "Add config"

# ✅ Use .gitignore
echo ".env*" >> .gitignore
echo "!.env.example" >> .gitignore
git add .gitignore
```

### 4. Sharing Secrets Insecurely

```bash
# ❌ Don't share via insecure channels
email: "Here's the API key: sk_live_123..."
slack: "Database password is: mypassword123"

# ✅ Use DotEnv sharing
dotenv secrets share DATABASE_URL \
  --with alice@example.com \
  --expires 1h \
  --project my-app
```

### 5. Using Weak Secrets

```bash
# ❌ Weak secrets
PASSWORD="password123"
API_KEY="test-key"
SECRET="mysecret"

# ✅ Strong secrets
PASSWORD="$(openssl rand -base64 32)"
API_KEY="$(uuidgen | tr -d '-')"
SECRET="$(head -c 32 /dev/urandom | base64)"
```

## Compliance Considerations

### Data Protection

- Encrypt secrets at rest and in transit
- Implement access logging
- Regular security audits
- Data residency compliance

### Regulatory Requirements

```bash
# GDPR compliance
dotenv compliance enable gdpr \
  --data-residency eu \
  --audit-retention 7y \
  --right-to-deletion

# HIPAA compliance
dotenv compliance enable hipaa \
  --encryption required \
  --access-logging enhanced \
  --baa required

# PCI DSS compliance
dotenv compliance enable pci-dss \
  --key-rotation 90d \
  --split-knowledge \
  --dual-control
```

## Emergency Procedures

### Secret Compromise

1. **Immediate Actions**

```bash
# Revoke compromised secret
dotenv secrets revoke COMPROMISED_KEY --immediate

# Rotate all related secrets
dotenv secrets rotate --tag related-to-compromise

# Alert team
dotenv alert send --priority critical --message "Secret compromise detected"
```

2. **Investigation**

```bash
# Audit access logs
dotenv audit logs --secret COMPROMISED_KEY --last 30d

# Check usage patterns
dotenv audit usage --secret COMPROMISED_KEY --anomalies
```

3. **Remediation**

```bash
# Generate new secrets
dotenv secrets generate NEW_KEY --project my-app

# Update applications
dotenv deploy trigger --force-refresh
```

## Next Steps

- [Access Control](./access-control) - Permission management
- [Key Concepts](./key-concepts) - Additional concepts
- [Security Model](./security-model) - Security architecture
- [Rotation Guide](/documentation/v1/guides/secret-rotation) - Detailed rotation strategies
