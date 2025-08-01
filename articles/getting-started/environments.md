---
title: Understanding Environments
slug: environments
order: 4
tags: [environments, configuration, deployment]
---

# Understanding Environments

Environments in DotEnv allow you to maintain different sets of configuration values for various stages of your application lifecycle.

## What Are Environments?

Environments are isolated containers for your secrets, typically representing:

- **Development**: Local development machines
- **Staging**: Pre-production testing
- **Production**: Live application
- **QA/Testing**: Automated testing
- **Preview**: Feature branch deployments

Each environment maintains its own set of secrets while sharing the same key names.

## Default Environments

Every project comes with three default environments:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Development │     │   Staging   │     │ Production  │
├─────────────┤     ├─────────────┤     ├─────────────┤
│ Local work  │ --> │   Testing   │ --> │    Live     │
│ Debugging   │     │     QA      │     │   Users     │
│ Hot reload  │     │   Preview   │     │  Scaling    │
└─────────────┘     └─────────────┘     └─────────────┘
```

## Working with Environments

### Listing Environments

```bash
# List all environments
dotenv environments list --project my-app

# Output:
# NAME          STATUS    INHERITS FROM    SECRETS
# development   active    -                12
# staging       active    development      8
# production    active    -                15
```

### Creating Environments

```bash
# Create a new environment
dotenv environments create qa --project my-app

# Create with description
dotenv environments create preview \
  --project my-app \
  --description "Feature branch deployments"

# Create with inheritance
dotenv environments create uat \
  --project my-app \
  --inherits-from staging
```

### Environment Inheritance

Environments can inherit values from other environments:

```bash
# Set up inheritance chain
development -> staging -> uat -> production
```

Example inheritance:

```yaml
# Development (base)
DATABASE_URL: "postgresql://localhost/myapp_dev"
API_KEY: "dev-key-123"
DEBUG: "true"
LOG_LEVEL: "debug"

# Staging (inherits from development)
DATABASE_URL: "postgresql://staging.db/myapp"  # Override
API_KEY: "dev-key-123"                        # Inherited
DEBUG: "true"                                 # Inherited
LOG_LEVEL: "info"                            # Override

# Production (no inheritance)
DATABASE_URL: "postgresql://prod.db/myapp"
API_KEY: "prod-key-456"
DEBUG: "false"
LOG_LEVEL: "error"
```

### Switching Environments

**CLI:**

```bash
# Pull from specific environment
dotenv secrets pull --project my-app --environment production

# Run with specific environment
dotenv run --project my-app --environment staging -- npm start
```

**SDK:**

```javascript
// JavaScript
const environment = process.env.NODE_ENV || "development";
const dotenv = new DotEnv({
    apiKey: process.env.DOTENV_API_KEY,
    project: "my-app",
    environment: environment,
});
```

```python
# Python
import os
environment = os.getenv('ENV', 'development')
dotenv = DotEnv(
    api_key=os.getenv('DOTENV_API_KEY'),
    project='my-app',
    environment=environment
)
```

## Environment Strategies

### 1. Progressive Deployment

```
development -> staging -> canary -> production
```

```bash
# Promote from staging to canary
dotenv environments promote staging canary --project my-app

# Promote from canary to production (10% traffic)
dotenv environments promote canary production \
  --project my-app \
  --percentage 10
```

### 2. Feature Branches

Create temporary environments for features:

```bash
# Create feature environment
BRANCH=$(git branch --show-current)
dotenv environments create "feature-$BRANCH" \
  --project my-app \
  --inherits-from development \
  --ttl 7d  # Auto-delete after 7 days

# Use in CI/CD
dotenv run \
  --project my-app \
  --environment "feature-$BRANCH" \
  -- npm test
```

### 3. Regional Deployment

Separate environments by region:

```bash
# Create regional environments
for region in us-east eu-west ap-south; do
  dotenv environments create "production-$region" \
    --project my-app \
    --inherits-from production

  # Set region-specific values
  dotenv secrets set API_ENDPOINT="https://api-$region.example.com" \
    --project my-app \
    --environment "production-$region"
done
```

## Environment-Specific Configuration

### Database Connections

```bash
# Development - Local database
dotenv secrets set DATABASE_URL="postgresql://localhost/myapp_dev" \
  --project my-app \
  --environment development

# Staging - Shared test database
dotenv secrets set DATABASE_URL="postgresql://staging.rds.amazonaws.com/myapp_staging" \
  --project my-app \
  --environment staging

# Production - High-availability cluster
dotenv secrets set DATABASE_URL="postgresql://prod-cluster.rds.amazonaws.com/myapp" \
  --project my-app \
  --environment production
```

### API Endpoints

```bash
# Environment-specific API URLs
declare -A api_urls=(
  ["development"]="http://localhost:3000"
  ["staging"]="https://api-staging.example.com"
  ["production"]="https://api.example.com"
)

for env in "${!api_urls[@]}"; do
  dotenv secrets set API_URL="${api_urls[$env]}" \
    --project my-app \
    --environment "$env"
done
```

### Feature Flags

```bash
# Enable features progressively
dotenv secrets set FEATURES="basic" \
  --project my-app \
  --environment development

dotenv secrets set FEATURES="basic,advanced" \
  --project my-app \
  --environment staging

dotenv secrets set FEATURES="basic,advanced,premium" \
  --project my-app \
  --environment production
```

## Automated Environment Management

### GitHub Actions

```yaml
name: Deploy
on:
    push:
        branches: [main, develop]

jobs:
    deploy:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v3

            - name: Determine Environment
              id: env
              run: |
                  if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
                    echo "environment=production" >> $GITHUB_OUTPUT
                  else
                    echo "environment=staging" >> $GITHUB_OUTPUT
                  fi

            - name: Load Secrets
              uses: dotenv/actions@v1
              with:
                  api-key: ${{ secrets.DOTENV_API_KEY }}
                  project: my-app
                  environment: ${{ steps.env.outputs.environment }}

            - name: Deploy
              run: ./deploy.sh
```

### Docker Compose

```yaml
version: "3.8"
services:
    app:
        image: myapp:latest
        environment:
            - DOTENV_ENVIRONMENT=${ENVIRONMENT:-development}
        env_file:
            - .env.${ENVIRONMENT:-development}
```

```bash
# Development
ENVIRONMENT=development docker-compose up

# Production
ENVIRONMENT=production docker-compose up
```

## Environment Security

### Access Control

```bash
# Restrict production access
dotenv environments permissions production \
  --project my-app \
  --role admin \
  --require-2fa

# Allow developers staging access
dotenv environments permissions staging \
  --project my-app \
  --role developer
```

### Audit Logging

```bash
# Enable audit logging for production
dotenv environments config production \
  --project my-app \
  --audit-log enabled \
  --audit-retention 90d

# View audit logs
dotenv audit logs \
  --project my-app \
  --environment production \
  --last 24h
```

### Environment Isolation

```javascript
// Prevent accidental production access
class SafeDotEnv {
    constructor(options) {
        if (
            options.environment === "production" &&
            !process.env.ALLOW_PRODUCTION
        ) {
            throw new Error("Production access requires ALLOW_PRODUCTION=true");
        }
        return new DotEnv(options);
    }
}
```

## Best Practices

### 1. Environment Naming

Use clear, consistent naming:

```
✅ Good:
- development
- staging
- production
- qa
- feature-auth
- production-eu

❌ Avoid:
- dev
- prod
- test123
- johns-env
```

### 2. Value Consistency

Keep key names consistent across environments:

```bash
# ✅ Good - Same keys, different values
DATABASE_URL="postgresql://localhost/app"      # development
DATABASE_URL="postgresql://staging.db/app"     # staging
DATABASE_URL="postgresql://prod.db/app"        # production

# ❌ Bad - Different keys
DB_URL="..."          # development
DATABASE="..."        # staging
POSTGRES_CONN="..."   # production
```

### 3. Progressive Revelation

Gradually introduce features:

```bash
# Development - All features
FEATURES="auth,payments,analytics,experimental"

# Staging - Stable features
FEATURES="auth,payments,analytics"

# Production - Validated features
FEATURES="auth,payments"
```

### 4. Environment Hygiene

Regular cleanup:

```bash
# List unused environments
dotenv environments list --project my-app --unused

# Clean up old feature branches
dotenv environments cleanup \
  --project my-app \
  --older-than 30d \
  --pattern "feature-*"
```

## Common Patterns

### Blue-Green Deployment

```bash
# Current production is "blue"
dotenv environments create green \
  --project my-app \
  --clone-from blue

# Deploy to green
./deploy.sh green

# Switch traffic
dotenv environments promote green blue \
  --project my-app \
  --atomic
```

### Canary Releases

```bash
# Create canary environment
dotenv environments create canary \
  --project my-app \
  --inherits-from production

# Deploy with 5% traffic
dotenv environments traffic canary \
  --project my-app \
  --percentage 5

# Monitor and increase
dotenv environments traffic canary \
  --project my-app \
  --percentage 25
```

### Disaster Recovery

```bash
# Backup production environment
dotenv environments backup production \
  --project my-app \
  --name "production-$(date +%Y%m%d)"

# Restore if needed
dotenv environments restore "production-20240115" \
  --project my-app \
  --to production
```

## Troubleshooting

### Environment Not Found

```bash
# Check available environments
dotenv environments list --project my-app

# Verify spelling and case
dotenv secrets pull --project my-app --environment Development  # ❌
dotenv secrets pull --project my-app --environment development  # ✅
```

### Inheritance Issues

```bash
# View inheritance chain
dotenv environments tree --project my-app

# Debug inherited values
dotenv secrets list \
  --project my-app \
  --environment staging \
  --show-inherited
```

### Permission Denied

```bash
# Check your environment access
dotenv environments permissions --project my-app --self

# Request access
dotenv environments request-access production --project my-app
```

## Next Steps

- [Team Setup](./team-setup) - Manage team access by environment
- [Best Practices](./best-practices) - Environment security guidelines
- [CI/CD Integration](/documentation/v1/integrations/ci-cd) - Automate environment deployments
- [Advanced Patterns](/documentation/v1/core-concepts/advanced-environments) - Complex environment strategies
