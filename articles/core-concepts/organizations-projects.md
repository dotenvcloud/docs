---
title: Organizations & Projects
slug: organizations-projects
order: 2
tags: [organizations, projects, structure, hierarchy]
---

# Organizations & Projects

Understanding the relationship between organizations and projects is key to effectively using DotEnv. This guide explains how to structure your secrets for optimal organization and security.

## Hierarchy Overview

```
┌─────────────────────────────────────────┐
│            Organization                 │
│         "ACME Corporation"              │
├─────────────────────────────────────────┤
│ Settings │ Billing │ Members │ Audit    │
└────────────┬──────────┬─────────────────┘
             │          │
      ┌──────▼───┐   ┌──▼──────┐
      │ Project A │   │ Project B│
      ├───────────┤   ├──────────┤
      │ • Envs    │   │ • Envs   │
      │ • Secrets │   │ • Secrets│
      │ • Team    │   │ • Team   │
      └───────────┘   └──────────┘
```

## Organizations

### What is an Organization?

An organization is the top-level container that:

- Groups related projects together
- Manages team members and permissions
- Handles billing and subscriptions
- Provides audit logs and compliance
- Enforces security policies

### Types of Organizations

#### Personal Organization

- Created automatically on signup
- Named after your username
- Ideal for personal projects
- Free tier available

#### Team Organization

- Created for companies/teams
- Supports multiple members
- Advanced permission controls
- SSO/SAML integration available

### Creating Organizations

```bash
# Create team organization
dotenv orgs create "ACME Corp" \
  --slug acme-corp \
  --type team \
  --description "ACME Corporation Projects"

# List your organizations
dotenv orgs list

# Switch active organization
dotenv config set organization acme-corp
```

### Organization Settings

```bash
# Configure security requirements
dotenv orgs settings update acme-corp \
  --require-2fa \
  --enforce-sso \
  --session-timeout 8h

# Set up IP restrictions
dotenv orgs security ip-allowlist \
  --organization acme-corp \
  --add "10.0.0.0/8" \
  --add "192.168.1.0/24"

# Configure audit retention
dotenv orgs settings update acme-corp \
  --audit-retention 365d \
  --export-audit-logs s3://backups/audit/
```

## Projects

### What is a Project?

A project represents a single application or service that:

- Contains environment-specific secrets
- Has its own access controls
- Maintains separate audit logs
- Can be deployed independently

### Project Types

#### Monolithic Application

```
web-app/
├── development/
│   ├── DATABASE_URL
│   ├── API_KEY
│   └── REDIS_URL
├── staging/
└── production/
```

#### Microservices

```
Organization/
├── api-gateway/
├── auth-service/
├── payment-service/
├── notification-service/
└── shared-config/
```

#### Multi-tenant SaaS

```
Organization/
├── platform-core/
├── tenant-services/
├── customer-a/
├── customer-b/
└── customer-c/
```

### Creating Projects

```bash
# Simple project
dotenv projects create web-app

# Detailed project
dotenv projects create api-service \
  --display-name "API Service" \
  --description "Main REST API" \
  --tags "backend,nodejs,critical" \
  --default-environment development
```

### Project Configuration

```bash
# Set project metadata
dotenv projects update api-service \
  --color "#FF6B6B" \
  --icon "🚀" \
  --repository "https://github.com/acme/api"

# Configure project settings
dotenv projects settings api-service \
  --auto-rotate-keys 90d \
  --require-approval production \
  --backup-enabled
```

## Organizing Your Structure

### Strategy 1: By Application

Best for: Independent applications with separate deployment cycles

```
ACME Corp/
├── website/           # Marketing site
├── web-app/          # Main application
├── mobile-api/       # Mobile backend
├── admin-panel/      # Internal tools
└── documentation/    # Docs site
```

**Pros:**

- Clear separation
- Independent deployments
- Easy permission management

**Cons:**

- Duplicate configuration
- Cross-project secrets harder

### Strategy 2: By Service

Best for: Microservices architecture

```
ACME Corp/
├── frontend/
│   ├── web/
│   ├── mobile/
│   └── desktop/
├── backend/
│   ├── api/
│   ├── workers/
│   └── webhooks/
├── data/
│   ├── postgres/
│   ├── redis/
│   └── elasticsearch/
└── infrastructure/
    ├── monitoring/
    └── logging/
```

**Pros:**

- Logical grouping
- Shared configurations
- Service-oriented

**Cons:**

- Complex permissions
- Tighter coupling

### Strategy 3: By Environment Type

Best for: Strong environment separation requirements

```
ACME Corp/
├── production/
│   ├── apps/
│   ├── services/
│   └── databases/
├── staging/
│   └── [mirrors production]
└── development/
    ├── local/
    └── shared/
```

**Pros:**

- Environment isolation
- Clear promotion path
- Compliance friendly

**Cons:**

- More projects to manage
- Complex for small teams

### Strategy 4: Hybrid Approach

Best for: Large organizations with diverse needs

```
ACME Corp/
├── Products/
│   ├── product-a/
│   └── product-b/
├── Platform/
│   ├── auth/
│   ├── billing/
│   └── analytics/
├── Infrastructure/
│   ├── kubernetes/
│   └── terraform/
└── Customers/
    ├── enterprise/
    └── self-serve/
```

## Managing Relationships

### Shared Secrets

Use a dedicated project for shared configuration:

```bash
# Create shared project
dotenv projects create shared-config \
  --description "Shared across all services"

# Add common secrets
dotenv secrets set SENTRY_DSN="..." --project shared-config
dotenv secrets set LOG_LEVEL="info" --project shared-config

# Reference in other projects
dotenv secrets link SENTRY_DSN \
  --from shared-config \
  --to web-app
```

### Project Dependencies

Define relationships between projects:

```yaml
# .dotenv/project.yml
name: web-app
dependencies:
    - shared-config
    - auth-service

imports:
    shared-config:
        - SENTRY_DSN
        - LOG_LEVEL
    auth-service:
        - AUTH_API_URL
        - AUTH_PUBLIC_KEY
```

### Cross-Project Access

```bash
# Grant cross-project permissions
dotenv projects access grant \
  --from web-app \
  --to api-service \
  --secrets "API_INTERNAL_KEY"

# Create service link
dotenv projects link \
  --source api-service \
  --target web-app \
  --type depends-on
```

## Organization Best Practices

### 1. Naming Conventions

```bash
# ✅ Good naming
my-company/         # Organization
├── api-service/   # Projects use kebab-case
├── web-app/
└── mobile-app/

# ❌ Poor naming
MyCompany/
├── API/
├── frontend-PROD/
└── test123/
```

### 2. Logical Grouping

Group by:

- **Application**: All resources for one app
- **Team**: Projects owned by specific team
- **Environment**: Separate by deployment stage
- **Customer**: For multi-tenant applications

### 3. Access Patterns

```bash
# Team-based access
dotenv teams create backend-team
dotenv teams add-projects backend-team \
  --projects "api-service,worker-jobs,webhooks"

# Role-based access
dotenv roles create frontend-developer \
  --permissions "projects:web-*:developer"
```

### 4. Metadata and Tags

```bash
# Use consistent tags
dotenv projects tag api-service \
  --add "tier:critical" \
  --add "team:backend" \
  --add "language:nodejs"

# Query by tags
dotenv projects list --tags "tier:critical"
```

## Migration Strategies

### From Flat Structure

Before:

```
all-secrets/
├── DEV_DATABASE_URL
├── PROD_DATABASE_URL
├── DEV_API_KEY
└── PROD_API_KEY
```

After:

```bash
# Create projects
dotenv projects create app

# Migrate secrets
dotenv secrets set DATABASE_URL=$DEV_DATABASE_URL \
  --project app --environment development

dotenv secrets set DATABASE_URL=$PROD_DATABASE_URL \
  --project app --environment production
```

### From Multiple .env Files

```bash
# Import script
#!/bin/bash
for env_file in .env.*; do
  environment=$(echo $env_file | cut -d. -f3)
  dotenv secrets import $env_file \
    --project my-app \
    --environment $environment
done
```

### Consolidating Projects

```bash
# Merge projects
dotenv projects merge \
  --source old-api \
  --target new-api \
  --prefix "legacy_"

# Archive old project
dotenv projects archive old-api
```

## Compliance Considerations

### Data Residency

```bash
# Create region-specific organizations
dotenv orgs create "ACME EU" --region eu-west-1
dotenv orgs create "ACME US" --region us-east-1

# Set data residency requirements
dotenv orgs compliance acme-eu \
  --require-eu-residency \
  --gdpr-compliant
```

### Audit Requirements

```bash
# Configure compliance settings
dotenv orgs compliance acme-corp \
  --sox-compliant \
  --audit-log-retention 7y \
  --change-approval-required \
  --quarterly-access-review
```

### Separation of Duties

```yaml
# compliance.yml
separation_of_duties:
    production:
        require_approvals: 2
        approvers: ["admin", "security-team"]
        exclude_self_approval: true

    financial_data:
        require_approvals: 3
        approvers: ["cfo", "security-team", "compliance"]
        audit_trail: enhanced
```

## Monitoring & Maintenance

### Health Checks

```bash
# Organization health
dotenv orgs health acme-corp

# Project status
dotenv projects status --all

# Find issues
dotenv projects audit \
  --check-unused-secrets \
  --check-weak-permissions \
  --check-stale-keys
```

### Usage Analytics

```bash
# Organization metrics
dotenv analytics org acme-corp \
  --metrics "projects,members,secrets,api-calls"

# Project insights
dotenv analytics project web-app \
  --period 30d \
  --group-by environment
```

### Cleanup Tasks

```bash
# Remove unused projects
dotenv projects cleanup \
  --inactive-days 180 \
  --dry-run

# Archive old environments
dotenv environments archive \
  --pattern "feature-*" \
  --older-than 30d

# Purge deleted secrets
dotenv secrets purge \
  --deleted-before 90d \
  --confirm
```

## Common Patterns

### Multi-Stage Organization

```
ACME Corp/
├── Development Org/      # Isolated dev
│   └── experiment/
├── Staging Org/         # Pre-prod
│   └── staging-app/
└── Production Org/      # Live
    └── prod-app/
```

### Platform + Customers

```
Platform Org/
├── core-services/
├── shared-infra/
└── platform-api/

Customer Orgs/
├── customer-a-org/
├── customer-b-org/
└── customer-c-org/
```

### Geographic Distribution

```
ACME Global/
├── acme-na/     # North America
├── acme-eu/     # Europe
├── acme-apac/   # Asia Pacific
└── acme-global/ # Shared global
```

## Next Steps

- [Environments](./environments-concept) - Configure multiple environments
- [Team Management](/documentation/v1/getting-started/team-setup) - Set up your team
- [Security Model](./security-model) - Understand security architecture
- [Best Practices](./secret-management) - Secret management guidelines
