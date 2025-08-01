---
title: Team Setup and Collaboration
slug: team-setup
order: 5
tags: [team, collaboration, permissions, security]
---

# Team Setup and Collaboration

DotEnv makes it easy to collaborate with your team while maintaining security and access control. This guide covers everything you need to set up team access.

## Understanding Organizations

Organizations are the top-level container for all your projects and team members.

```
Organization
├── Team Members
├── Projects
│   ├── Project A
│   ├── Project B
│   └── Project C
├── Billing
└── Settings
```

## Setting Up Your Organization

### Creating an Organization

When you sign up, a personal organization is created automatically. To create a team organization:

```bash
# Create new organization
dotenv orgs create "ACME Corp" \
  --slug acme-corp \
  --type team

# Switch to new organization
dotenv config set organization acme-corp
```

### Organization Settings

Configure organization-wide settings:

```bash
# Require 2FA for all members
dotenv orgs config acme-corp --require-2fa

# Set security policies
dotenv orgs config acme-corp \
  --password-policy strong \
  --session-timeout 8h \
  --ip-allowlist "10.0.0.0/8"
```

## Team Member Management

### Inviting Team Members

**Via CLI:**

```bash
# Basic invite
dotenv members invite alice@example.com

# Invite with specific role
dotenv members invite bob@example.com --role developer

# Invite multiple members
dotenv members invite-bulk team.csv
```

**Via Dashboard:**

1. Navigate to Organization Settings
2. Click "Team Members"
3. Click "Invite Member"
4. Enter email and select role

### Organization Roles

| Role        | Description                | Permissions                               |
| ----------- | -------------------------- | ----------------------------------------- |
| **Owner**   | Organization owner         | Full control, billing access              |
| **Admin**   | Organization administrator | Manage members, projects, settings        |
| **Member**  | Regular member             | Create projects, access assigned projects |
| **Billing** | Billing administrator      | View and manage billing only              |

### Project-Level Permissions

Assign members to specific projects with granular permissions:

```bash
# Add member to project
dotenv projects members add alice@example.com \
  --project my-app \
  --role developer

# Available project roles:
# - owner: Full project control
# - admin: Manage project settings
# - developer: Read/write secrets
# - viewer: Read-only access
```

## Permission System

### Permission Matrix

| Action              | Owner | Admin | Developer | Viewer |
| ------------------- | ----- | ----- | --------- | ------ |
| View secrets        | ✅    | ✅    | ✅        | ✅     |
| Create/edit secrets | ✅    | ✅    | ✅        | ❌     |
| Delete secrets      | ✅    | ✅    | ✅        | ❌     |
| Manage environments | ✅    | ✅    | ❌        | ❌     |
| Invite members      | ✅    | ✅    | ❌        | ❌     |
| Project settings    | ✅    | ✅    | ❌        | ❌     |
| Delete project      | ✅    | ❌    | ❌        | ❌     |

### Environment-Specific Permissions

Control access by environment:

```bash
# Grant production access only to seniors
dotenv environments permissions production \
  --project my-app \
  --allow "senior-developer,admin,owner"

# Restrict development to specific team
dotenv environments permissions development \
  --project my-app \
  --team "frontend-team"
```

### Custom Roles

Create custom roles for your organization:

```bash
# Create QA role
dotenv roles create qa-engineer \
  --organization acme-corp \
  --permissions "secrets:read,environments:staging:write"

# Create DevOps role
dotenv roles create devops \
  --organization acme-corp \
  --permissions "secrets:*,environments:*,deploy:*"
```

## Team Workflows

### Code Review Workflow

Require approval for production changes:

```yaml
# .dotenv/workflow.yml
workflows:
    production-changes:
        triggers:
            - environment: production
              actions: [create, update, delete]

        approvals:
            required: 2
            from: [admin, owner]
            timeout: 24h

        notifications:
            slack: "#deployments"
            email: "ops-team@example.com"
```

### Deployment Permissions

Set up deployment-specific access:

```bash
# Create deployment user
dotenv members create-service-account \
  --name "ci-deployer" \
  --role deployment

# Grant specific permissions
dotenv permissions grant ci-deployer \
  --actions "secrets:read,environments:production:read" \
  --project my-app

# Generate API key
dotenv api-keys create \
  --name "GitHub Actions" \
  --account ci-deployer \
  --expires 90d
```

### Audit Trail

Track all team actions:

```bash
# View recent activity
dotenv audit logs --organization acme-corp --last 7d

# Filter by member
dotenv audit logs --member alice@example.com

# Export for compliance
dotenv audit export \
  --format csv \
  --from 2024-01-01 \
  --to 2024-12-31 \
  > audit-2024.csv
```

## Security Best Practices

### 1. Principle of Least Privilege

Grant minimum necessary access:

```bash
# ✅ Good - Specific access
dotenv projects members add developer@example.com \
  --project api-service \
  --role developer \
  --environments "development,staging"

# ❌ Bad - Overly broad access
dotenv members invite developer@example.com --role admin
```

### 2. Regular Access Reviews

Audit team access quarterly:

```bash
# Generate access report
dotenv reports access \
  --organization acme-corp \
  --include-last-activity

# Remove inactive members
dotenv members cleanup \
  --inactive-days 90 \
  --dry-run
```

### 3. Enforce 2FA

```bash
# Check 2FA status
dotenv members list --show-2fa-status

# Enforce for specific roles
dotenv security require-2fa \
  --roles "admin,owner" \
  --grace-period 7d
```

### 4. API Key Management

```bash
# Rotate API keys regularly
dotenv api-keys rotate \
  --older-than 90d \
  --notify

# Limit API key scope
dotenv api-keys create \
  --name "Mobile App" \
  --scopes "secrets:read" \
  --projects "mobile-api" \
  --environments "production"
```

## Collaboration Features

### Shared Environments

Create shared development environments:

```bash
# Create shared environment
dotenv environments create dev-shared \
  --project my-app \
  --description "Shared development environment"

# Allow team access
dotenv environments share dev-shared \
  --with "frontend-team,backend-team"
```

### Secret Requests

Enable secret request workflow:

```bash
# Request new secret
dotenv secrets request \
  --key PAYMENT_API_KEY \
  --project my-app \
  --environment production \
  --reason "Integration with Stripe"

# Approve request
dotenv secrets requests approve req-123 \
  --value "sk_live_..."
```

### Comments and Documentation

Add context for team members:

```bash
# Add secret documentation
dotenv secrets document DATABASE_URL \
  --project my-app \
  --description "PostgreSQL connection string" \
  --format "postgresql://user:pass@host:port/database" \
  --example "postgresql://app:secret@db.example.com:5432/myapp"

# Add environment notes
dotenv environments note staging \
  --project my-app \
  --message "Refreshed from production on 2024-01-15"
```

## Integration with Tools

### Slack Integration

```bash
# Connect Slack
dotenv integrations connect slack \
  --webhook https://hooks.slack.com/services/xxx/yyy/zzz

# Configure notifications
dotenv notifications config \
  --channel "#engineering" \
  --events "secret.created,member.invited,deploy.started"
```

### GitHub Integration

```yaml
# .github/dotenv.yml
teams:
    - github: "@acme-corp/backend"
      dotenv_role: developer
      projects: ["api", "workers"]

    - github: "@acme-corp/frontend"
      dotenv_role: developer
      projects: ["web", "mobile"]

    - github: "@acme-corp/devops"
      dotenv_role: admin
      projects: "*"
```

### SAML/SSO Setup

```bash
# Configure SAML
dotenv sso configure saml \
  --organization acme-corp \
  --metadata-url https://idp.example.com/metadata \
  --attribute-mapping "email=mail,name=displayName"

# Enforce SSO
dotenv sso enforce \
  --organization acme-corp \
  --exclude "api-keys"
```

## Common Scenarios

### Onboarding New Developer

```bash
#!/bin/bash
# onboard-developer.sh

EMAIL=$1
TEAM=$2

# 1. Invite to organization
dotenv members invite $EMAIL --role member

# 2. Add to relevant projects
for project in api web mobile; do
  dotenv projects members add $EMAIL \
    --project $project \
    --role developer
done

# 3. Grant environment access
dotenv environments permissions development \
  --project all \
  --add $EMAIL

dotenv environments permissions staging \
  --project all \
  --add $EMAIL \
  --read-only

# 4. Send welcome resources
dotenv notifications send \
  --to $EMAIL \
  --template developer-welcome
```

### Contractor Access

```bash
# Create time-limited access
dotenv members invite contractor@example.com \
  --role developer \
  --expires "2024-06-30" \
  --projects "mobile-app" \
  --environments "development"

# Restrict to specific secrets
dotenv permissions create contractor-policy \
  --allow "secrets:read" \
  --deny "secrets:write:*_KEY,secrets:write:*_SECRET" \
  --apply-to contractor@example.com
```

### Team Rotation

```bash
# Export current permissions
dotenv members export --format json > team-backup.json

# Bulk permission update
dotenv members update-bulk \
  --file new-team-structure.csv \
  --dry-run

# Notify affected members
dotenv notifications send \
  --template role-change \
  --to-affected
```

## Troubleshooting

### Access Denied Issues

```bash
# Check member's permissions
dotenv permissions check alice@example.com \
  --project my-app \
  --action secrets:write

# View permission inheritance
dotenv permissions trace alice@example.com \
  --resource "projects/my-app/secrets/API_KEY"
```

### Invitation Problems

```bash
# Resend invitation
dotenv members invite resend alice@example.com

# Check invitation status
dotenv members invitations list --pending

# Cancel invitation
dotenv members invite cancel alice@example.com
```

### SSO Issues

```bash
# Test SSO connection
dotenv sso test --organization acme-corp

# View SSO logs
dotenv sso logs --last 24h

# Bypass SSO for emergency
dotenv members emergency-access \
  --user admin@example.com \
  --duration 1h \
  --reason "SSO provider outage"
```

## Best Practices Summary

1. **Start with least privilege** - Grant only necessary access
2. **Use groups** - Manage permissions through teams, not individuals
3. **Regular audits** - Review access quarterly
4. **Document everything** - Add descriptions to secrets and permissions
5. **Automate onboarding** - Script common permission patterns
6. **Monitor activity** - Set up alerts for sensitive actions
7. **Plan for departures** - Have a offboarding checklist

## Next Steps

- [Security Best Practices](./best-practices) - Detailed security guidelines
- [API Authentication](/documentation/v1/api/authentication) - Set up API access
- [Audit Logging](/documentation/v1/administration/audit-logs) - Track team activity
- [SSO Configuration](/documentation/v1/administration/sso) - Enterprise authentication
