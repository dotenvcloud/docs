---
title: Access Control
slug: access-control
order: 6
tags: [permissions, rbac, access-control, security]
---

# Access Control

DotEnv implements a comprehensive access control system that ensures users can only access the resources they need. This guide explains how permissions work and how to manage access effectively.

## Access Control Model

DotEnv uses a hybrid model combining:

- **RBAC** (Role-Based Access Control) for standard permissions
- **ABAC** (Attribute-Based Access Control) for fine-grained control
- **ReBAC** (Relationship-Based Access Control) for hierarchical permissions

```
User → Roles → Permissions → Resources
  ↓       ↓         ↓            ↓
Alice  Developer  Read/Write  Secrets in Dev
```

## Permission Hierarchy

### Organization Level

```yaml
Organization (ACME Corp)
├── Roles
│   ├── Owner (full control)
│   ├── Admin (manage organization)
│   ├── Member (access projects)
│   └── Billing (financial only)
└── Permissions
├── org:manage
├── billing:manage
├── members:manage
└── projects:create
```

### Project Level

```yaml
Project (api-service)
├── Roles
│   ├── Owner (full project control)
│   ├── Admin (manage project)
│   ├── Developer (read/write secrets)
│   └── Viewer (read only)
└── Permissions
├── project:manage
├── secrets:read
├── secrets:write
└── environments:manage
```

### Environment Level

```yaml
Environment (production)
├── Access Rules
│   ├── Require MFA
│   ├── Require Approval
│   └── IP Restrictions
└── Permissions
├── env:read
├── env:write
└── env:delete
```

## Roles

### Built-in Organization Roles

#### Owner

- Full organization control
- Cannot be restricted
- Billing access
- Delete organization

```bash
# Assign owner (requires current owner)
dotenv members role set alice@example.com owner \
  --organization acme-corp \
  --transfer-from current-owner@example.com
```

#### Admin

- Manage organization settings
- Manage members
- Create/delete projects
- View billing (no changes)

```bash
# Assign admin
dotenv members role set bob@example.com admin \
  --organization acme-corp
```

#### Member

- Access assigned projects
- Create new projects
- Manage own API keys
- View organization info

```bash
# Assign member
dotenv members role set charlie@example.com member \
  --organization acme-corp
```

#### Billing

- View/manage billing only
- No access to projects
- Update payment methods
- View invoices

```bash
# Assign billing role
dotenv members role set finance@example.com billing \
  --organization acme-corp
```

### Built-in Project Roles

#### Project Owner

```yaml
permissions:
    - project:delete
    - project:transfer
    - settings:manage
    - members:manage
    - secrets:*
    - environments:*
```

#### Project Admin

```yaml
permissions:
    - project:update
    - settings:read
    - members:add
    - secrets:*
    - environments:manage
```

#### Developer

```yaml
permissions:
    - project:read
    - secrets:read
    - secrets:write
    - environments:read
    - audit:read
```

#### Viewer

```yaml
permissions:
    - project:read
    - secrets:read
    - environments:read
    - audit:read
```

### Custom Roles

Create roles tailored to your needs:

```bash
# Create custom role
dotenv roles create senior-developer \
  --organization acme-corp \
  --description "Senior developer with production read access"

# Add permissions
dotenv roles permissions add senior-developer \
  --permissions \
    "projects:*:read" \
    "secrets:*:write" \
    "environments:development:*" \
    "environments:staging:*" \
    "environments:production:read"

# Assign to user
dotenv members role set alice@example.com senior-developer \
  --organization acme-corp
```

## Permissions

### Permission Format

Permissions follow the pattern: `resource:action:scope`

```
secrets:write:production
│       │     └── Scope (optional)
│       └── Action
└── Resource
```

### Resource Types

| Resource       | Description  | Actions                      |
| -------------- | ------------ | ---------------------------- |
| `org`          | Organization | manage, read                 |
| `billing`      | Billing      | manage, read                 |
| `members`      | Members      | manage, add, remove, read    |
| `projects`     | Projects     | create, read, update, delete |
| `secrets`      | Secrets      | read, write, delete          |
| `environments` | Environments | create, read, update, delete |
| `audit`        | Audit logs   | read, export                 |
| `api-keys`     | API Keys     | create, read, revoke         |

### Action Types

- `*` - All actions
- `create` - Create new resources
- `read` - View resources
- `update` - Modify resources
- `delete` - Remove resources
- `manage` - All CRUD operations

### Scope

Scope narrows permissions to specific resources:

```bash
# Project-specific
secrets:read:my-app

# Environment-specific
secrets:write:my-app/production

# Pattern matching
secrets:read:my-app/*
projects:*:frontend-*
```

## Access Policies

### Policy Definition

```yaml
# policies/production-access.yml
name: production-access
description: Production environment access policy
rules:
    - effect: allow
      principals:
          - role:senior-developer
          - user:alice@example.com
      actions:
          - secrets:read
      resources:
          - project:api-service/environment:production
      conditions:
          - mfa: required
          - time: business_hours
          - ip: corporate_network

    - effect: deny
      principals:
          - role:developer
      actions:
          - secrets:write
          - secrets:delete
      resources:
          - project:*/environment:production
```

### Policy Enforcement

```javascript
// Policy evaluation
async function canAccess(user, action, resource) {
    const policies = await getPoliciesForUser(user);

    for (const policy of policies) {
        for (const rule of policy.rules) {
            if (
                matchesPrincipal(user, rule.principals) &&
                matchesAction(action, rule.actions) &&
                matchesResource(resource, rule.resources) &&
                meetsConditions(user, rule.conditions)
            ) {
                if (rule.effect === "deny") {
                    return false; // Explicit deny
                }
                if (rule.effect === "allow") {
                    allowed = true; // Mark as allowed
                }
            }
        }
    }

    return allowed; // Default deny
}
```

## Conditional Access

### Time-Based Access

```yaml
conditions:
    time:
        type: business_hours
        timezone: America/New_York
        schedule:
            monday-friday: "09:00-18:00"
            saturday-sunday: deny
        exceptions:
            - date: 2024-12-25
              access: deny
              reason: "Holiday"
```

### Location-Based Access

```yaml
conditions:
    location:
        type: ip_range
        allowed:
            - "10.0.0.0/8" # Office network
            - "192.168.1.0/24" # VPN
            - "203.0.113.0/24" # Backup office
        denied:
            - "0.0.0.0/0" # Everything else
        geoip:
            allowed_countries: ["US", "CA", "GB"]
            denied_countries: ["CN", "RU", "KP"]
```

### Device-Based Access

```yaml
conditions:
    device:
        require_managed: true
        allowed_platforms: ["windows", "macos", "linux"]
        require_encryption: true
        require_antivirus: true
        certificate_required: true
```

### Risk-Based Access

```javascript
// Dynamic risk assessment
const riskFactors = {
    newDevice: 0.3,
    unusualLocation: 0.4,
    unusualTime: 0.2,
    rapidRequests: 0.5,
    sensitiveResource: 0.6,
};

function calculateRiskScore(context) {
    let score = 0;

    if (isNewDevice(context.device)) score += riskFactors.newDevice;
    if (isUnusualLocation(context.ip)) score += riskFactors.unusualLocation;
    if (isUnusualTime(context.time)) score += riskFactors.unusualTime;
    if (isSensitive(context.resource)) score += riskFactors.sensitiveResource;

    return Math.min(score, 1.0);
}

// Apply additional controls based on risk
if (riskScore > 0.7) {
    requireMFA();
    requireApproval();
    enhancedLogging();
}
```

## Access Management

### Granting Access

```bash
# Grant project access
dotenv projects members add alice@example.com \
  --project api-service \
  --role developer

# Grant specific permissions
dotenv permissions grant alice@example.com \
  --permissions "secrets:read:production" \
  --project api-service

# Grant conditional access
dotenv access grant alice@example.com \
  --resource "project:api-service/env:production" \
  --permissions "secrets:read" \
  --conditions "mfa:required,ip:10.0.0.0/8"
```

### Access Reviews

```bash
# List user access
dotenv access list alice@example.com

# Review project access
dotenv projects members list api-service \
  --include-permissions \
  --include-last-access

# Find over-privileged users
dotenv access audit \
  --find-excessive-permissions \
  --suggest-reductions
```

### Revoking Access

```bash
# Revoke project access
dotenv projects members remove alice@example.com \
  --project api-service

# Revoke specific permissions
dotenv permissions revoke alice@example.com \
  --permissions "secrets:write:production"

# Emergency revocation
dotenv access revoke-all alice@example.com \
  --reason "Security incident" \
  --immediate
```

## Delegation

### Temporary Access

```bash
# Grant temporary access
dotenv access grant-temporary bob@example.com \
  --permissions "secrets:read:production" \
  --duration 4h \
  --reason "Debugging production issue"

# Delegate permissions
dotenv access delegate \
  --from alice@example.com \
  --to bob@example.com \
  --permissions "secrets:read" \
  --expires "2024-01-20" \
  --revocable
```

### Access Requests

```yaml
# access-request.yml
request:
    user: developer@example.com
    resource: project:api-service/env:production
    permissions: ["secrets:read"]
    reason: "Debug customer issue #1234"
    duration: 2h

approval:
    required_approvers: 1
    approvers:
        - role:admin
        - user:security@example.com
    auto_expire: 24h
```

## Service Accounts

### Creating Service Accounts

```bash
# Create service account
dotenv service-accounts create ci-bot \
  --description "CI/CD automation" \
  --project api-service

# Assign permissions
dotenv service-accounts permissions ci-bot \
  --add "secrets:read:*" \
  --add "environments:staging:deploy"

# Generate credentials
dotenv service-accounts key create ci-bot \
  --expires 90d \
  --scopes "secrets:read" \
  > ci-bot-credentials.json
```

### Service Account Security

```yaml
# service-account-policy.yml
service_accounts:
    ci-bot:
        permissions:
            - secrets:read:staging
            - secrets:read:production
            - deploy:staging
        restrictions:
            ip_whitelist:
                - "10.0.1.0/24" # CI network
            rate_limit:
                requests_per_minute: 100
            key_rotation:
                max_age: 90d
                auto_rotate: true
```

## API Key Management

### API Key Types

```bash
# Organization API key (broad access)
dotenv api-keys create org-key \
  --organization acme-corp \
  --permissions "projects:create,members:read"

# Project API key (scoped)
dotenv api-keys create project-key \
  --project api-service \
  --permissions "secrets:read,environments:list"

# Environment API key (narrow)
dotenv api-keys create env-key \
  --project api-service \
  --environment production \
  --permissions "secrets:read" \
  --read-only
```

### API Key Security

```javascript
// API key validation
class ApiKeyValidator {
    async validate(key) {
        const keyData = await this.parseKey(key);

        // Check expiration
        if (keyData.expiresAt < Date.now()) {
            throw new Error("API key expired");
        }

        // Check revocation
        if (await this.isRevoked(keyData.id)) {
            throw new Error("API key revoked");
        }

        // Check rate limits
        if (await this.isRateLimited(keyData.id)) {
            throw new Error("Rate limit exceeded");
        }

        // Validate signature
        if (!this.verifySignature(keyData)) {
            throw new Error("Invalid API key");
        }

        return keyData;
    }
}
```

## Audit & Compliance

### Access Logs

```json
{
    "timestamp": "2024-01-15T14:30:00Z",
    "user": "alice@example.com",
    "action": "secrets:read",
    "resource": "project:api-service/env:production/secret:DATABASE_URL",
    "result": "allowed",
    "context": {
        "ip": "203.0.113.45",
        "mfa": true,
        "session_age": 3600,
        "risk_score": 0.2
    }
}
```

### Compliance Reports

```bash
# Generate access report
dotenv compliance report access \
  --period "last-quarter" \
  --format pdf \
  --include-privileged-access \
  --include-anomalies

# SOX compliance
dotenv compliance sox \
  --verify-separation-of-duties \
  --verify-access-reviews \
  --generate-evidence
```

## Best Practices

### 1. Principle of Least Privilege

```bash
# ✅ Good: Specific permissions
dotenv permissions grant developer@example.com \
  --permissions "secrets:read:development"

# ❌ Bad: Overly broad
dotenv permissions grant developer@example.com \
  --permissions "secrets:*:*"
```

### 2. Regular Access Reviews

```bash
# Schedule quarterly reviews
dotenv access reviews schedule \
  --frequency quarterly \
  --reviewers "security-team" \
  --auto-revoke-unused
```

### 3. Separation of Duties

```yaml
# Ensure no single person can:
separation_rules:
    - rule: deploy_and_approve
      description: "Cannot deploy and approve same change"
      roles_excluded: ["developer", "approver"]

    - rule: create_and_audit
      description: "Cannot create resources and audit them"
      roles_excluded: ["admin", "auditor"]
```

### 4. Emergency Access

```bash
# Break-glass procedure
dotenv access emergency \
  --user oncall@example.com \
  --grant "secrets:*:production" \
  --duration 1h \
  --reason "Production incident #12345" \
  --notify "security-team" \
  --require-dual-approval
```

## Troubleshooting

### Access Denied

```bash
# Debug access issues
dotenv access debug alice@example.com \
  --action "secrets:read" \
  --resource "production/DATABASE_URL" \
  --explain

# Check effective permissions
dotenv permissions effective alice@example.com \
  --project api-service
```

### Permission Conflicts

```bash
# Find conflicting policies
dotenv policies conflicts \
  --user alice@example.com \
  --action "secrets:write"

# Test policy changes
dotenv policies test \
  --policy new-policy.yml \
  --dry-run
```

## Next Steps

- [Key Concepts](./key-concepts) - Additional important concepts
- [Team Setup](/documentation/v1/getting-started/team-setup) - Configure team access
- [Security Model](./security-model) - Security architecture
- [Audit Logging](/documentation/v1/administration/audit-logs) - Track access
