---
title: Team Collaboration
slug: team-collaboration
order: 4
tags: [use-cases, teams, collaboration, workflows]
---

# Team Collaboration

Learn how to effectively use DotEnv for team collaboration, including onboarding processes, access control patterns, and collaborative workflows for managing secrets across development teams.

## Overview

Team collaboration challenges:

- Developer onboarding
- Access control and permissions
- Secret sharing workflows
- Audit trails and compliance
- Environment isolation
- Change management

## Team Structure

### 1. Role-Based Organization

```yaml
# team-structure.yaml
organization:
    name: "My Company"
    teams:
        - name: "Engineering"
          roles:
              - name: "Backend Team"
                members: ["alice@company.com", "bob@company.com"]
                projects:
                    - api-service
                    - auth-service
                permissions:
                    - read: [development, staging]
                    - write: [development]

              - name: "Frontend Team"
                members: ["charlie@company.com", "dave@company.com"]
                projects:
                    - web-app
                    - mobile-app
                permissions:
                    - read: [development, staging]
                    - write: [development]

              - name: "DevOps Team"
                members: ["eve@company.com", "frank@company.com"]
                projects: ["*"]
                permissions:
                    - read: [development, staging, production]
                    - write: [development, staging]
                    - admin: [production]

        - name: "QA"
          members: ["grace@company.com", "henry@company.com"]
          projects: ["*"]
          permissions:
              - read: [qa, staging]
              - write: [qa]
```

### 2. Permission Matrix

```javascript
// config/permissions.js
const permissionMatrix = {
    roles: {
        developer: {
            development: ["read", "write"],
            staging: ["read"],
            production: [],
        },
        senior_developer: {
            development: ["read", "write", "delete"],
            staging: ["read", "write"],
            production: ["read"],
        },
        devops: {
            development: ["read", "write", "delete"],
            staging: ["read", "write", "delete"],
            production: ["read", "write"],
        },
        admin: {
            development: ["read", "write", "delete", "admin"],
            staging: ["read", "write", "delete", "admin"],
            production: ["read", "write", "delete", "admin"],
        },
    },

    special_permissions: {
        rotate_keys: ["devops", "admin"],
        create_environment: ["senior_developer", "devops", "admin"],
        delete_project: ["admin"],
        audit_access: ["devops", "admin"],
    },
};

function canUserAccess(userRole, environment, action) {
    const rolePermissions = permissionMatrix.roles[userRole];
    if (!rolePermissions) return false;

    const envPermissions = rolePermissions[environment] || [];
    return envPermissions.includes(action);
}

function hasSpecialPermission(userRole, permission) {
    const allowedRoles = permissionMatrix.special_permissions[permission] || [];
    return allowedRoles.includes(userRole);
}

module.exports = { canUserAccess, hasSpecialPermission };
```

## Onboarding Workflows

### 1. New Developer Setup

```bash
#!/bin/bash
# scripts/onboard-developer.sh

echo "🚀 DotEnv Developer Onboarding"
echo "=============================="

# Get developer information
read -p "Developer email: " email
read -p "Team (backend/frontend/devops): " team
read -p "Role (developer/senior_developer): " role

# Create user account
echo "Creating DotEnv account..."
dotenv user invite \
  --email "$email" \
  --role "$role" \
  --team "$team"

# Set up project access
case "$team" in
  "backend")
    projects=("api-service" "auth-service" "shared-libs")
    ;;
  "frontend")
    projects=("web-app" "mobile-app" "ui-components")
    ;;
  "devops")
    projects=("*")
    ;;
esac

# Grant project permissions
for project in "${projects[@]}"; do
  echo "Granting access to $project..."

  if [ "$role" == "developer" ]; then
    dotenv access grant \
      --user "$email" \
      --project "$project" \
      --environment "development" \
      --permission "read,write"

    dotenv access grant \
      --user "$email" \
      --project "$project" \
      --environment "staging" \
      --permission "read"
  else
    dotenv access grant \
      --user "$email" \
      --project "$project" \
      --environment "development,staging" \
      --permission "read,write"
  fi
done

# Create development environment
echo "Setting up development environment..."
cat > .env.template << EOF
# Development Environment Setup
# Run: dotenv pull development --project <project-name>

# Generated for: $email
# Team: $team
# Role: $role
# Date: $(date)

# Available projects:
$(printf '# - %s\n' "${projects[@]}")
EOF

echo "✅ Onboarding complete!"
echo ""
echo "Next steps for $email:"
echo "1. Check email for DotEnv invitation"
echo "2. Set up 2FA for account security"
echo "3. Install DotEnv CLI: curl -fsSL https://cli.dotenv.cloud/install.sh | sh"
echo "4. Run: dotenv auth login"
echo "5. Pull development secrets for your projects"
```

### 2. Onboarding Checklist

```markdown
## Developer Onboarding Checklist

### Account Setup

- [ ] DotEnv account created
- [ ] 2FA enabled
- [ ] Added to correct team
- [ ] Role assigned

### Access Configuration

- [ ] Project permissions granted
- [ ] Environment access configured
- [ ] API token generated (if needed)
- [ ] Webhooks configured (if applicable)

### Local Development

- [ ] DotEnv CLI installed
- [ ] Authentication configured
- [ ] Development secrets pulled
- [ ] IDE integration set up

### Documentation

- [ ] Security guidelines reviewed
- [ ] Team workflows understood
- [ ] Emergency procedures known
- [ ] Support contacts saved
```

## Collaborative Workflows

### 1. Pull Request Workflow

```yaml
# .github/workflows/secret-validation.yml
name: Secret Validation

on:
    pull_request:
        paths:
            - ".dotenv/**"
            - "config/**"
            - "**/*.env.example"

jobs:
    validate-secrets:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v3

            - name: Check for hardcoded secrets
              uses: trufflesecurity/trufflehog@main
              with:
                  path: ./
                  base: ${{ github.event.pull_request.base.sha }}
                  head: ${{ github.event.pull_request.head.sha }}

            - name: Validate env.example files
              run: |
                  # Check that all .env.example files are properly formatted
                  find . -name "*.env.example" -type f | while read file; do
                    echo "Validating $file..."
                    
                    # Check for actual values (should only have placeholders)
                    if grep -E "=[\w\d]{20,}" "$file"; then
                      echo "❌ Found potential secret in $file"
                      exit 1
                    fi
                    
                    # Validate format
                    if ! grep -E "^[A-Z_]+=" "$file" > /dev/null; then
                      echo "❌ Invalid format in $file"
                      exit 1
                    fi
                  done

            - name: Check required secrets
              run: |
                  # Ensure all required secrets are documented
                  ./scripts/check-required-secrets.sh
```

### 2. Secret Request Process

```typescript
// utils/secret-request.ts
interface SecretRequest {
    id: string;
    requestedBy: string;
    approvedBy?: string;
    project: string;
    environment: string;
    secrets: Array<{
        key: string;
        description: string;
        type: "api_key" | "database" | "service" | "other";
        required: boolean;
    }>;
    justification: string;
    status: "pending" | "approved" | "rejected";
    createdAt: Date;
    reviewedAt?: Date;
}

class SecretRequestManager {
    async createRequest(
        request: Omit<SecretRequest, "id" | "status" | "createdAt">,
    ): Promise<SecretRequest> {
        const newRequest: SecretRequest = {
            ...request,
            id: generateId(),
            status: "pending",
            createdAt: new Date(),
        };

        // Save to database
        await this.saveRequest(newRequest);

        // Notify approvers
        await this.notifyApprovers(newRequest);

        return newRequest;
    }

    async approveRequest(requestId: string, approverId: string): Promise<void> {
        const request = await this.getRequest(requestId);

        if (request.status !== "pending") {
            throw new Error("Request already processed");
        }

        // Validate approver permissions
        if (!this.canApprove(approverId, request)) {
            throw new Error("Insufficient permissions to approve");
        }

        // Update request
        request.status = "approved";
        request.approvedBy = approverId;
        request.reviewedAt = new Date();

        await this.saveRequest(request);

        // Create secrets in DotEnv
        await this.createSecrets(request);

        // Notify requester
        await this.notifyRequester(request);
    }

    private async createSecrets(request: SecretRequest): Promise<void> {
        const client = new DotEnvClient();

        for (const secret of request.secrets) {
            await client.createSecret({
                project: request.project,
                environment: request.environment,
                key: secret.key,
                value: "", // Placeholder - actual value set separately
                metadata: {
                    requestId: request.id,
                    requestedBy: request.requestedBy,
                    approvedBy: request.approvedBy,
                    description: secret.description,
                },
            });
        }
    }

    private canApprove(userId: string, request: SecretRequest): boolean {
        // Check if user has approval permissions for the environment
        const user = getUserById(userId);
        return hasPermission(user, request.environment, "approve_secrets");
    }
}
```

### 3. Change Tracking

```javascript
// middleware/audit-logger.js
const crypto = require("crypto");

class AuditLogger {
    constructor(storage) {
        this.storage = storage;
    }

    async logSecretChange(change) {
        const entry = {
            id: crypto.randomUUID(),
            timestamp: new Date().toISOString(),
            user: change.user,
            action: change.action,
            project: change.project,
            environment: change.environment,
            resource: {
                type: "secret",
                key: change.secretKey,
            },
            changes: this.sanitizeChanges(change.changes),
            metadata: {
                ip: change.ip,
                userAgent: change.userAgent,
                sessionId: change.sessionId,
            },
        };

        await this.storage.save("audit_log", entry);

        // Real-time notifications for critical changes
        if (this.isCriticalChange(change)) {
            await this.notifySecurityTeam(entry);
        }
    }

    sanitizeChanges(changes) {
        // Never log actual secret values
        return {
            before: changes.before ? "[REDACTED]" : null,
            after: changes.after ? "[REDACTED]" : null,
            metadata: changes.metadata,
        };
    }

    isCriticalChange(change) {
        return (
            change.environment === "production" ||
            change.action === "delete" ||
            change.secretKey.includes("ADMIN") ||
            change.secretKey.includes("ROOT")
        );
    }

    async getAuditTrail(filters) {
        const query = this.buildQuery(filters);
        const entries = await this.storage.query("audit_log", query);

        return entries.map((entry) => ({
            ...entry,
            // Add human-readable descriptions
            description: this.generateDescription(entry),
        }));
    }

    generateDescription(entry) {
        const templates = {
            create: `${entry.user} created secret "${entry.resource.key}" in ${entry.environment}`,
            update: `${entry.user} updated secret "${entry.resource.key}" in ${entry.environment}`,
            delete: `${entry.user} deleted secret "${entry.resource.key}" from ${entry.environment}`,
            rotate: `${entry.user} rotated secret "${entry.resource.key}" in ${entry.environment}`,
        };

        return (
            templates[entry.action] ||
            `${entry.user} performed ${entry.action} on ${entry.resource.key}`
        );
    }
}

module.exports = AuditLogger;
```

## Access Control Patterns

### 1. Team-Based Access

```typescript
// models/team-access.ts
interface TeamAccess {
    teamId: string;
    projects: string[] | "*";
    environments: {
        [env: string]: Permission[];
    };
    restrictions?: {
        timeBasedAccess?: {
            start: string; // ISO time
            end: string; // ISO time
        };
        ipWhitelist?: string[];
        requireMFA?: boolean;
    };
}

type Permission = "read" | "write" | "delete" | "rotate" | "admin";

class TeamAccessControl {
    async grantTeamAccess(config: TeamAccess): Promise<void> {
        // Validate team exists
        const team = await this.getTeam(config.teamId);
        if (!team) throw new Error("Team not found");

        // Apply access to all team members
        for (const memberId of team.members) {
            await this.grantUserAccess({
                userId: memberId,
                projects: config.projects,
                environments: config.environments,
                restrictions: config.restrictions,
            });
        }

        // Store team access configuration
        await this.storeTeamAccess(config);
    }

    async checkAccess(
        userId: string,
        project: string,
        environment: string,
        action: Permission,
    ): Promise<boolean> {
        // Get user's team memberships
        const teams = await this.getUserTeams(userId);

        for (const team of teams) {
            const teamAccess = await this.getTeamAccess(team.id);

            // Check project access
            if (
                teamAccess.projects !== "*" &&
                !teamAccess.projects.includes(project)
            ) {
                continue;
            }

            // Check environment permissions
            const envPermissions = teamAccess.environments[environment] || [];
            if (!envPermissions.includes(action)) {
                continue;
            }

            // Check restrictions
            if (teamAccess.restrictions) {
                if (!this.checkRestrictions(userId, teamAccess.restrictions)) {
                    continue;
                }
            }

            return true;
        }

        return false;
    }

    private checkRestrictions(
        userId: string,
        restrictions: TeamAccess["restrictions"],
    ): boolean {
        // Time-based access
        if (restrictions?.timeBasedAccess) {
            const now = new Date();
            const start = new Date(restrictions.timeBasedAccess.start);
            const end = new Date(restrictions.timeBasedAccess.end);

            if (now < start || now > end) {
                return false;
            }
        }

        // IP whitelist
        if (restrictions?.ipWhitelist) {
            const userIp = this.getUserIp(userId);
            if (!restrictions.ipWhitelist.includes(userIp)) {
                return false;
            }
        }

        // MFA requirement
        if (restrictions?.requireMFA) {
            if (!this.userHasMFA(userId)) {
                return false;
            }
        }

        return true;
    }
}
```

### 2. Temporary Access

```javascript
// utils/temporary-access.js
class TemporaryAccess {
    async grantTemporaryAccess(config) {
        const { userId, project, environment, permissions, duration, reason } =
            config;

        // Generate access token
        const token = {
            id: crypto.randomUUID(),
            userId,
            project,
            environment,
            permissions,
            expiresAt: new Date(Date.now() + duration),
            reason,
            grantedBy: config.grantedBy,
            grantedAt: new Date(),
        };

        // Store token
        await this.storeToken(token);

        // Schedule automatic revocation
        setTimeout(() => {
            this.revokeAccess(token.id);
        }, duration);

        // Log the grant
        await this.auditLog.log({
            action: "grant_temporary_access",
            user: config.grantedBy,
            target: userId,
            details: {
                project,
                environment,
                duration,
                reason,
            },
        });

        return token;
    }

    async useToken(tokenId, action) {
        const token = await this.getToken(tokenId);

        // Validate token
        if (!token) {
            throw new Error("Invalid token");
        }

        if (new Date() > token.expiresAt) {
            throw new Error("Token expired");
        }

        if (!token.permissions.includes(action)) {
            throw new Error("Insufficient permissions");
        }

        // Log token usage
        await this.auditLog.log({
            action: "use_temporary_token",
            user: token.userId,
            tokenId: token.id,
            operation: action,
        });

        return true;
    }
}

// Usage example
const tempAccess = new TemporaryAccess();

// Grant 2-hour access for incident response
await tempAccess.grantTemporaryAccess({
    userId: "developer@company.com",
    project: "api-service",
    environment: "production",
    permissions: ["read"],
    duration: 2 * 60 * 60 * 1000, // 2 hours
    reason: "Incident response - ticket #12345",
    grantedBy: "ops-lead@company.com",
});
```

## Communication Patterns

### 1. Slack Integration

```javascript
// integrations/slack.js
const { WebClient } = require("@slack/web-api");

class SlackNotifier {
    constructor(token) {
        this.client = new WebClient(token);
        this.channels = {
            security: "#security-alerts",
            devops: "#devops",
            general: "#engineering",
        };
    }

    async notifySecretChange(change) {
        const channel = this.getChannel(change);
        const message = this.formatMessage(change);

        await this.client.chat.postMessage({
            channel,
            ...message,
        });
    }

    getChannel(change) {
        if (change.environment === "production") {
            return this.channels.security;
        }
        if (change.action === "delete" || change.action === "rotate") {
            return this.channels.devops;
        }
        return this.channels.general;
    }

    formatMessage(change) {
        const color = this.getColor(change.action);
        const emoji = this.getEmoji(change.action);

        return {
            text: `${emoji} Secret ${change.action} in ${change.environment}`,
            attachments: [
                {
                    color,
                    fields: [
                        {
                            title: "Project",
                            value: change.project,
                            short: true,
                        },
                        {
                            title: "Environment",
                            value: change.environment,
                            short: true,
                        },
                        {
                            title: "Secret",
                            value: `\`${change.secretKey}\``,
                            short: true,
                        },
                        {
                            title: "User",
                            value: change.user,
                            short: true,
                        },
                        {
                            title: "Timestamp",
                            value: new Date().toISOString(),
                            short: false,
                        },
                    ],
                    actions: [
                        {
                            type: "button",
                            text: "View Audit Log",
                            url: `https://dotenv.cloudpany.com/audit/${change.id}`,
                        },
                    ],
                },
            ],
        };
    }

    getColor(action) {
        const colors = {
            create: "good",
            update: "warning",
            delete: "danger",
            rotate: "#3AA3E3",
        };
        return colors[action] || "#808080";
    }

    getEmoji(action) {
        const emojis = {
            create: "✅",
            update: "📝",
            delete: "🗑️",
            rotate: "🔄",
        };
        return emojis[action] || "📌";
    }
}
```

### 2. Review Process

```typescript
// services/review-process.ts
interface SecretChangeReview {
    id: string;
    changeRequest: {
        project: string;
        environment: string;
        changes: Array<{
            key: string;
            operation: "create" | "update" | "delete";
            oldValue?: string;
            newValue?: string;
        }>;
    };
    requestedBy: string;
    requestedAt: Date;
    reviewers: string[];
    approvals: Array<{
        reviewer: string;
        decision: "approve" | "reject";
        comment?: string;
        timestamp: Date;
    }>;
    status: "pending" | "approved" | "rejected";
    requiredApprovals: number;
}

class ReviewProcess {
    async createReview(
        changeRequest: SecretChangeReview["changeRequest"],
        requestedBy: string,
    ): Promise<SecretChangeReview> {
        // Determine required reviewers based on environment
        const reviewers = await this.getRequiredReviewers(
            changeRequest.project,
            changeRequest.environment,
        );

        const review: SecretChangeReview = {
            id: generateId(),
            changeRequest,
            requestedBy,
            requestedAt: new Date(),
            reviewers,
            approvals: [],
            status: "pending",
            requiredApprovals: this.getRequiredApprovalCount(
                changeRequest.environment,
            ),
        };

        // Save review
        await this.saveReview(review);

        // Notify reviewers
        await this.notifyReviewers(review);

        return review;
    }

    async submitApproval(
        reviewId: string,
        reviewer: string,
        decision: "approve" | "reject",
        comment?: string,
    ): Promise<void> {
        const review = await this.getReview(reviewId);

        // Validate reviewer
        if (!review.reviewers.includes(reviewer)) {
            throw new Error("Not authorized to review this change");
        }

        // Check for existing approval
        if (review.approvals.some((a) => a.reviewer === reviewer)) {
            throw new Error("Already reviewed this change");
        }

        // Add approval
        review.approvals.push({
            reviewer,
            decision,
            comment,
            timestamp: new Date(),
        });

        // Check if review is complete
        const approvalCount = review.approvals.filter(
            (a) => a.decision === "approve",
        ).length;
        const rejectionCount = review.approvals.filter(
            (a) => a.decision === "reject",
        ).length;

        if (rejectionCount > 0) {
            review.status = "rejected";
            await this.handleRejection(review);
        } else if (approvalCount >= review.requiredApprovals) {
            review.status = "approved";
            await this.handleApproval(review);
        }

        await this.saveReview(review);
    }

    private async handleApproval(review: SecretChangeReview): Promise<void> {
        // Apply the changes
        for (const change of review.changeRequest.changes) {
            await this.applyChange(
                review.changeRequest.project,
                review.changeRequest.environment,
                change,
            );
        }

        // Notify requester
        await this.notifyRequester(review, "approved");
    }

    private getRequiredApprovalCount(environment: string): number {
        const approvalCounts = {
            development: 0,
            staging: 1,
            production: 2,
        };
        return approvalCounts[environment] || 1;
    }
}
```

## Documentation and Knowledge Sharing

### 1. Secret Documentation

```yaml
# .dotenv/documentation.yaml
secrets:
    DATABASE_URL:
        description: "PostgreSQL connection string"
        format: "postgresql://[user]:[password]@[host]:[port]/[database]"
        example: "postgresql://user:pass@localhost:5432/myapp"
        required: true
        environments:
            - development
            - staging
            - production

    REDIS_URL:
        description: "Redis connection string for caching"
        format: "redis://[host]:[port]/[database]"
        example: "redis://localhost:6379/0"
        required: true
        environments:
            - development
            - staging
            - production

    STRIPE_API_KEY:
        description: "Stripe API key for payment processing"
        format: "sk_[live|test]_[alphanumeric]"
        example: "sk_test_4eC39HqLyjWDarjtT1zdp7dc"
        required: true
        environments:
            - staging
            - production
        sensitive: true
        rotation_policy: "90 days"

    FEATURE_FLAGS:
        description: "JSON object containing feature flags"
        format: "JSON"
        example: '{"new_ui": false, "beta_features": true}'
        required: false
        default: "{}"
```

### 2. Team Runbooks

````markdown
# Secret Management Runbook

## Common Operations

### Adding a New Secret

1. Create secret request ticket
2. Get approval from team lead
3. Add secret to development environment:
    ```bash
    dotenv secret create --key NEW_SECRET --project myapp --environment development
    ```
````

4. Document in `.dotenv/documentation.yaml`
5. Update `.env.example` file
6. Notify team via Slack

### Rotating Secrets

1. Check current secret usage:
    ```bash
    dotenv secret usage --key API_KEY --project myapp
    ```
2. Create new version:
    ```bash
    dotenv secret rotate --key API_KEY --project myapp --environment production
    ```
3. Deploy with both old and new secrets
4. Update all services to use new secret
5. Remove old secret after verification

### Emergency Procedures

#### Compromised Secret

1. Immediately rotate the secret:
    ```bash
    dotenv secret rotate --key COMPROMISED_KEY --emergency
    ```
2. Notify security team
3. Check audit logs for unauthorized access
4. Update all affected services
5. Document incident

#### Access Revocation

1. Remove user access:
    ```bash
    dotenv access revoke --user user@company.com --all
    ```
2. Rotate any secrets they had access to
3. Audit recent activity
4. Update team documentation

````

## Best Practices

### 1. Team Guidelines

```javascript
// .dotenv/team-guidelines.js
const guidelines = {
  naming_conventions: {
    secrets: {
      pattern: /^[A-Z][A-Z0-9_]*$/,
      examples: ['DATABASE_URL', 'API_KEY', 'JWT_SECRET'],
      avoid: ['db', 'key', 'secret']
    },
    projects: {
      pattern: /^[a-z][a-z0-9-]*$/,
      examples: ['api-service', 'web-app', 'auth-service'],
      avoid: ['API', 'WebApp', 'auth_service']
    }
  },

  security_practices: [
    'Never commit secrets to version control',
    'Use 2FA for all accounts',
    'Rotate secrets every 90 days',
    'Use least privilege access',
    'Document all secret usage'
  ],

  review_requirements: {
    development: {
      required_reviewers: 0,
      auto_approve: true
    },
    staging: {
      required_reviewers: 1,
      eligible_reviewers: ['senior_developer', 'devops']
    },
    production: {
      required_reviewers: 2,
      eligible_reviewers: ['devops', 'security_team'],
      requires_ticket: true
    }
  }
};
````

### 2. Automation Scripts

```bash
#!/bin/bash
# scripts/team-sync.sh

# Sync team members with DotEnv organization
echo "Syncing team members..."

# Get current team members from HR system
team_members=$(curl -s https://hr.company.com/api/engineering-team | jq -r '.[] | .email')

# Get current DotEnv users
dotenv_users=$(dotenv user list --format json | jq -r '.[] | .email')

# Add new team members
for email in $team_members; do
  if ! echo "$dotenv_users" | grep -q "$email"; then
    echo "Adding new team member: $email"
    dotenv user invite --email "$email" --role developer
  fi
done

# Remove departed team members
for email in $dotenv_users; do
  if ! echo "$team_members" | grep -q "$email"; then
    echo "Removing departed member: $email"
    dotenv user remove --email "$email"
  fi
done

echo "✅ Team sync complete"
```

## Resources

- [Team Security Best Practices](https://owasp.org/www-project-top-ten/)
- [Access Control Models](https://en.wikipedia.org/wiki/Computer_security_model)
- [Audit Logging Standards](https://www.sans.org/blog/successful-siem-and-log-management-strategies/)
- [DotEnv Access Control](/documentation/v1/core-concepts/access-control)

## Next Steps

- [Set up team structure](#team-structure)
- [Configure onboarding process](#onboarding-workflows)
- [Implement review workflows](#collaborative-workflows)
- [Enable audit logging](#change-tracking)
