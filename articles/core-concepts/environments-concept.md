---
title: Environment Concepts
slug: environments-concept
order: 3
tags: [environments, configuration, concepts]
---

# Environment Concepts

Environments are a fundamental concept in DotEnv that enable you to maintain different configurations for various stages of your application lifecycle. This guide explores the concept in depth.

## What Are Environments?

Environments are isolated namespaces within a project that contain their own set of secrets. They represent different contexts where your application runs.

```
Project: e-commerce-api
├── Environment: development
│   ├── DATABASE_URL = "postgres://localhost/dev"
│   ├── STRIPE_KEY = "sk_test_..."
│   └── DEBUG = "true"
├── Environment: staging
│   ├── DATABASE_URL = "postgres://staging.aws/app"
│   ├── STRIPE_KEY = "sk_test_..."
│   └── DEBUG = "false"
└── Environment: production
    ├── DATABASE_URL = "postgres://prod.aws/app"
    ├── STRIPE_KEY = "sk_live_..."
    └── DEBUG = "false"
```

## Environment Philosophy

### 1. Development-Production Parity

Keep environments as similar as possible while changing only what's necessary:

```yaml
# All environments have same keys
DATABASE_URL: <changes per environment>
REDIS_URL: <changes per environment>
API_VERSION: "v1" # Same across all
LOG_FORMAT: "json" # Same across all
```

### 2. Progressive Disclosure

Environments should progressively reveal more production-like characteristics:

```
Local → Development → Staging → Production
├─ Mocked services  → Real test services → Full integration → Live services
├─ Debug enabled    → Verbose logging    → Standard logs   → Minimal logs
└─ Sample data      → Test data          → Migrated data  → Real data
```

### 3. Fail-Safe Defaults

Development environments should have safe defaults:

```javascript
// Environment-aware configuration
const config = {
    enableEmails: process.env.ENABLE_EMAILS === "true", // Default: false
    rateLimit: parseInt(process.env.RATE_LIMIT) || 100, // Default: restricted
    debugMode: process.env.NODE_ENV !== "production", // Default: true for non-prod
};
```

## Standard Environment Types

### Development

**Purpose**: Local development and testing
**Characteristics**:

- Relaxed security constraints
- Verbose logging and debugging
- Mock external services
- Fast feedback loops

```bash
# Common development settings
DATABASE_URL="postgresql://localhost/myapp_dev"
REDIS_URL="redis://localhost:6379"
API_URL="http://localhost:3000"
DEBUG="true"
LOG_LEVEL="debug"
MOCK_EXTERNAL_APIS="true"
HOT_RELOAD="true"
```

### Staging

**Purpose**: Pre-production testing and QA
**Characteristics**:

- Production-like infrastructure
- Real integrations with test accounts
- Performance testing capable
- Data refresh from production

```bash
# Common staging settings
DATABASE_URL="postgresql://staging-db.aws.com/myapp"
REDIS_URL="redis://staging-redis.aws.com:6379"
API_URL="https://staging-api.example.com"
DEBUG="false"
LOG_LEVEL="info"
ENABLE_PROFILING="true"
FEATURE_FLAGS="beta_features"
```

### Production

**Purpose**: Live application serving users
**Characteristics**:

- Maximum security
- Optimized performance
- Real services and data
- Minimal logging

```bash
# Common production settings
DATABASE_URL="postgresql://prod-cluster.aws.com/myapp"
REDIS_URL="redis://prod-redis.aws.com:6379"
API_URL="https://api.example.com"
DEBUG="false"
LOG_LEVEL="error"
ENABLE_MONITORING="true"
ALERT_CHANNEL="#incidents"
```

## Advanced Environment Patterns

### Feature Environments

Create temporary environments for feature development:

```bash
# Create feature environment
BRANCH="feature/new-checkout"
dotenv environments create $BRANCH \
  --project e-commerce \
  --inherits-from development \
  --ttl 14d

# Auto-generated URLs
FRONTEND_URL="https://new-checkout.preview.example.com"
API_URL="https://api-new-checkout.preview.example.com"
```

### Regional Environments

Separate environments by geographic region:

```
production-us-east/
production-eu-west/
production-ap-south/
```

```bash
# Region-specific configuration
AWS_REGION="us-east-1"      # or eu-west-1, ap-south-1
DB_ENDPOINT="<region-specific>"
CDN_ENDPOINT="<region-specific>"
COMPLIANCE_MODE="GDPR"      # for EU regions
```

### Customer-Specific Environments

For multi-tenant SaaS applications:

```
production/
production-customer-a/
production-customer-b/
```

```bash
# Customer overrides
TENANT_ID="customer-a"
CUSTOM_DOMAIN="app.customer-a.com"
FEATURE_FLAGS="advanced_analytics,white_label"
API_RATE_LIMIT="10000"  # Higher for enterprise
```

### Blue-Green Environments

For zero-downtime deployments:

```
production-blue/   (current)
production-green/  (next)
```

```bash
# Traffic management
DEPLOYMENT_ID="blue"
HEALTH_CHECK_URL="/health/blue"
LOAD_BALANCER_POOL="blue-pool"
DATABASE_POOL="blue-connections"
```

## Environment Inheritance

### Inheritance Chains

Environments can inherit from each other to reduce duplication:

```
base
 └── development
      └── feature-branches
 └── staging
      └── qa
      └── load-testing
 └── production
      └── production-canary
```

### Implementation Example

```yaml
# base environment
LOG_FORMAT: "json"
API_VERSION: "v2"
CORS_ORIGINS: "https://example.com"

# development (inherits from base)
LOG_FORMAT: "json"          # inherited
API_VERSION: "v2"           # inherited
CORS_ORIGINS: "*"           # overridden
DATABASE_URL: "local://..." # new

# staging (inherits from base)
LOG_FORMAT: "json"              # inherited
API_VERSION: "v2"               # inherited
CORS_ORIGINS: "https://*.example.com"  # overridden
DATABASE_URL: "staging://..."   # new
```

### Inheritance Rules

1. **Explicit Overrides**: Child environment values override parent values
2. **Addition**: Child can add new keys not in parent
3. **No Deletion**: Cannot remove inherited keys (set to empty instead)
4. **Single Parent**: Each environment has at most one parent

## Environment Lifecycle

### 1. Creation Phase

```bash
# Standard environment
dotenv environments create qa \
  --project my-app \
  --description "QA testing environment"

# Temporary environment
dotenv environments create "preview-$PR_NUMBER" \
  --project my-app \
  --inherits-from staging \
  --ttl 7d \
  --auto-destroy
```

### 2. Active Phase

During active use, environments can be:

- Modified with new secrets
- Accessed by applications
- Monitored for usage
- Backed up regularly

### 3. Maintenance Phase

```bash
# Refresh from production
dotenv environments refresh staging \
  --from production \
  --exclude "API_KEYS,CREDENTIALS"

# Sync specific values
dotenv environments sync \
  --from production \
  --to staging \
  --keys "FEATURE_FLAGS,CONFIG_VERSION"
```

### 4. Decommission Phase

```bash
# Soft delete (recoverable)
dotenv environments archive preview-123 \
  --reason "PR merged"

# Hard delete (after grace period)
dotenv environments delete preview-123 \
  --confirm \
  --force
```

## Environment Variables vs Secrets

### Environment Variables

- Configuration that changes between environments
- Non-sensitive values
- Can be committed to version control

```bash
# Safe to commit
NODE_ENV=production
API_VERSION=v2
LOG_LEVEL=info
ENABLE_METRICS=true
```

### Secrets

- Sensitive credentials and keys
- Must never be in version control
- Managed by DotEnv

```bash
# Managed by DotEnv
DATABASE_PASSWORD=<encrypted>
API_SECRET_KEY=<encrypted>
JWT_PRIVATE_KEY=<encrypted>
AWS_SECRET_ACCESS_KEY=<encrypted>
```

### Hybrid Approach

```javascript
// config.js - Committed to repo
export default {
    app: {
        name: "My App",
        version: "1.0.0",
        environment: process.env.NODE_ENV,
    },
    database: {
        host: process.env.DB_HOST, // From DotEnv
        password: process.env.DB_PASSWORD, // From DotEnv (secret)
        pool: {
            min: 2,
            max: 10, // Hardcoded config
        },
    },
};
```

## Environment Promotion

### Manual Promotion

```bash
# Promote staging to production
dotenv environments promote \
  --from staging \
  --to production \
  --keys "FEATURE_FLAGS,API_VERSION" \
  --require-approval
```

### Automated Promotion

```yaml
# .github/workflows/promote.yml
name: Promote Staging to Production
on:
    workflow_dispatch:
    schedule:
        - cron: "0 2 * * 2" # Weekly on Tuesday

jobs:
    promote:
        steps:
            - name: Promote Configuration
              run: |
                  dotenv environments promote \
                    --from staging \
                    --to production \
                    --exclude "DEBUG,TEST_*" \
                    --create-backup
```

### Gradual Rollout

```bash
# Canary deployment
dotenv environments create production-canary \
  --inherits-from production

# Update canary
dotenv secrets set NEW_FEATURE="enabled" \
  --environment production-canary

# Gradual promotion
for percent in 1 5 10 25 50 100; do
  dotenv environments traffic \
    --environment production-canary \
    --percentage $percent
  sleep 3600  # Monitor for 1 hour
done
```

## Environment Isolation

### Network Isolation

```yaml
# Environment-specific networking
development:
    vpc: "vpc-dev"
    subnets: ["subnet-dev-1", "subnet-dev-2"]
    security_groups: ["sg-dev-app"]

production:
    vpc: "vpc-prod"
    subnets: ["subnet-prod-1", "subnet-prod-2", "subnet-prod-3"]
    security_groups: ["sg-prod-app", "sg-prod-db"]
```

### Access Isolation

```bash
# Environment-specific permissions
dotenv environments permissions production \
  --allow-roles "admin,senior-dev" \
  --require-2fa \
  --require-approval

dotenv environments permissions development \
  --allow-roles "developer,contractor" \
  --no-approval-required
```

### Data Isolation

```sql
-- Separate databases per environment
CREATE DATABASE myapp_development;
CREATE DATABASE myapp_staging;
CREATE DATABASE myapp_production;

-- Or schema separation
CREATE SCHEMA development;
CREATE SCHEMA staging;
CREATE SCHEMA production;
```

## Monitoring Environments

### Health Checks

```bash
# Environment status
dotenv environments health --all

# Specific checks
dotenv environments check production \
  --verify-secrets \
  --test-connections \
  --validate-certificates
```

### Usage Analytics

```bash
# Environment metrics
dotenv analytics environment production \
  --metrics "access_count,unique_users,api_calls" \
  --period 30d

# Compare environments
dotenv analytics compare \
  --environments "staging,production" \
  --metric api_calls
```

### Alerts and Notifications

```yaml
# alerts.yml
alerts:
    - name: production-access
      environment: production
      trigger: access_granted
      notify: security@example.com

    - name: staging-high-usage
      environment: staging
      trigger: api_calls > 10000/hour
      notify: "#ops-alerts"
```

## Best Practices

### 1. Consistent Naming

```bash
# ✅ Good
development
staging
production
qa
feature-auth

# ❌ Bad
dev
stage
prod-final
test123
```

### 2. Clear Purpose

Each environment should have a single, clear purpose:

- **development**: Local development only
- **staging**: Pre-production testing
- **production**: Live users
- **qa**: Quality assurance testing

### 3. Minimal Differences

Keep environments as similar as possible:

```javascript
// Use same structure, different values
const dbConfig = {
    host: process.env.DB_HOST,
    port: process.env.DB_PORT || 5432, // Same default
    ssl: process.env.NODE_ENV === "production", // Conditional
};
```

### 4. Regular Cleanup

```bash
# Automated cleanup
dotenv environments cleanup \
  --older-than 30d \
  --pattern "feature-*,preview-*" \
  --archived
```

## Common Pitfalls

### 1. Environment Drift

**Problem**: Environments become inconsistent over time
**Solution**: Regular synchronization and validation

```bash
# Detect drift
dotenv environments diff staging production

# Sync structure
dotenv environments align staging \
  --template production \
  --keys-only
```

### 2. Hardcoded Environments

**Problem**: Code assumes specific environment names
**Solution**: Use feature flags instead

```javascript
// ❌ Bad
if (process.env.NODE_ENV === "staging") {
    enableNewFeature();
}

// ✅ Good
if (process.env.FEATURE_NEW_CHECKOUT === "true") {
    enableNewFeature();
}
```

### 3. Secret Environment Proliferation

**Problem**: Too many environments to manage
**Solution**: Set TTLs and automate cleanup

```bash
# Limit environment lifetime
dotenv environments create "test-${ID}" \
  --ttl 24h \
  --auto-delete \
  --notify-before-delete 1h
```

## Next Steps

- [Security Model](./security-model) - Environment security in depth
- [Secret Management](./secret-management) - Best practices for secrets
- [Key Concepts](./key-concepts) - Other important concepts
- [Advanced Environments](/documentation/v1/guides/advanced-environments) - Complex patterns
