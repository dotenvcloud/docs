---
title: User Management
slug: user-management
order: 1
tags: [administration, users, access-control, teams]
---

# User Management

This guide covers user management in DotEnv, including creating users, managing roles and permissions, implementing access controls, and maintaining user security. Learn how to effectively manage your organization's users and their access to resources.

## User Management Overview

### User Hierarchy

```
┌─────────────────────────────────────────┐
│         Organization Owner              │
│  • Full control over organization      │
│  • Billing management                  │
│  • Cannot be removed                   │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│         Organization Admin              │
│  • User management                     │
│  • Project creation                    │
│  • Settings management                 │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│         Project Admin                   │
│  • Project settings                    │
│  • Team management                     │
│  • Environment configuration           │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│         Team Member                     │
│  • Access based on team permissions    │
│  • Environment-specific access         │
│  • Read/write based on role           │
└─────────────────────────────────────────┘
```

## User Creation and Onboarding

### Creating Users

#### Via Web Dashboard

```javascript
// User creation process
const userCreation = {
    required_fields: {
        email: "user@company.com",
        first_name: "John",
        last_name: "Doe",
        role: "member",
    },

    optional_fields: {
        department: "Engineering",
        title: "Senior Developer",
        phone: "+1-555-0123",
        timezone: "America/New_York",
    },

    initial_settings: {
        send_invitation: true,
        require_mfa: true,
        temporary_password: false,
        initial_projects: [],
    },
};

// Bulk user import
async function bulkImportUsers(csvFile) {
    const users = await parseCSV(csvFile);

    const results = {
        successful: [],
        failed: [],
    };

    for (const userData of users) {
        try {
            const user = await createUser({
                email: userData.email,
                name: `${userData.first_name} ${userData.last_name}`,
                role: userData.role || "member",
                teams: userData.teams?.split(",") || [],
            });

            results.successful.push(user);
        } catch (error) {
            results.failed.push({
                email: userData.email,
                error: error.message,
            });
        }
    }

    return results;
}
```

#### Via API

```bash
# Create a new user
curl -X POST https://api.dotenv.cloud/v1/organizations/{org_id}/users \
  -H "Authorization: Bearer $DOTENV_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "newuser@company.com",
    "first_name": "Jane",
    "last_name": "Smith",
    "role": "member",
    "send_invitation": true
  }'

# Bulk create users
curl -X POST https://api.dotenv.cloud/v1/organizations/{org_id}/users/bulk \
  -H "Authorization: Bearer $DOTENV_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "users": [
      {
        "email": "user1@company.com",
        "name": "User One",
        "role": "member"
      },
      {
        "email": "user2@company.com",
        "name": "User Two",
        "role": "admin"
      }
    ]
  }'
```

### User Onboarding

#### Automated Onboarding Workflow

```javascript
class UserOnboarding {
    async onboardNewUser(userId) {
        const workflow = {
            steps: [
                { name: "send_welcome_email", status: "pending" },
                { name: "create_initial_access", status: "pending" },
                { name: "assign_to_teams", status: "pending" },
                { name: "setup_mfa", status: "pending" },
                { name: "complete_training", status: "pending" },
            ],
        };

        // Send welcome email
        await this.sendWelcomeEmail(userId);
        workflow.steps[0].status = "completed";

        // Create initial access
        await this.setupInitialAccess(userId);
        workflow.steps[1].status = "completed";

        // Assign to default teams
        await this.assignDefaultTeams(userId);
        workflow.steps[2].status = "completed";

        // Require MFA setup
        await this.requireMFASetup(userId);
        workflow.steps[3].status = "in_progress";

        return workflow;
    }

    async setupInitialAccess(userId) {
        // Grant basic permissions
        await grantPermissions(userId, {
            projects: ["onboarding-project"],
            environments: ["development"],
            permissions: ["read"],
        });

        // Create personal sandbox
        await createProject({
            name: `${user.name}-sandbox`,
            owner: userId,
            type: "sandbox",
        });
    }
}
```

#### Onboarding Checklist

```yaml
# user-onboarding-checklist.yaml
onboarding_tasks:
    immediate:
        - verify_email_address
        - set_secure_password
        - enable_mfa
        - accept_terms_of_service

    first_day:
        - complete_security_training
        - review_access_permissions
        - join_assigned_teams
        - configure_cli_access

    first_week:
        - complete_platform_training
        - setup_development_environment
        - review_security_policies
        - emergency_contact_info

    ongoing:
        - quarterly_access_review
        - annual_security_training
        - password_rotation
        - permission_updates
```

## Role-Based Access Control (RBAC)

### Role Definitions

```javascript
// Role configuration
const roles = {
    owner: {
        name: "Organization Owner",
        permissions: ["*"], // All permissions
        restrictions: [],
        system_role: true,
    },

    admin: {
        name: "Administrator",
        permissions: [
            "users.*",
            "projects.*",
            "teams.*",
            "settings.read",
            "settings.write",
            "billing.read",
        ],
        restrictions: ["organization.delete", "billing.write"],
    },

    developer: {
        name: "Developer",
        permissions: [
            "projects.read",
            "secrets.read",
            "secrets.write",
            "environments.read",
            "deployments.create",
        ],
        restrictions: ["users.*", "teams.write", "projects.delete"],
    },

    readonly: {
        name: "Read Only",
        permissions: ["*.read"],
        restrictions: ["*.write", "*.delete", "*.create"],
    },
};

// Custom role creation
async function createCustomRole(roleData) {
    const role = {
        id: generateId(),
        name: roleData.name,
        description: roleData.description,
        permissions: roleData.permissions,
        created_by: getCurrentUser(),
        created_at: new Date(),
    };

    // Validate permissions
    validatePermissions(role.permissions);

    // Check for conflicts
    checkPermissionConflicts(role);

    // Save role
    await saveRole(role);

    return role;
}
```

### Permission Management

```javascript
// Permission system
class PermissionManager {
    constructor() {
        this.permissions = {
            // User permissions
            "users.read": "View users",
            "users.create": "Create users",
            "users.update": "Update users",
            "users.delete": "Delete users",

            // Project permissions
            "projects.read": "View projects",
            "projects.create": "Create projects",
            "projects.update": "Update projects",
            "projects.delete": "Delete projects",

            // Secret permissions
            "secrets.read": "View secrets",
            "secrets.write": "Create/update secrets",
            "secrets.delete": "Delete secrets",
            "secrets.export": "Export secrets",

            // Environment permissions
            "environments.read": "View environments",
            "environments.write": "Manage environments",

            // Key permissions
            "keys.read": "View encryption keys",
            "keys.rotate": "Rotate encryption keys",
            "keys.export": "Export encryption keys",
        };
    }

    async checkPermission(userId, permission, resource) {
        // Get user's roles
        const roles = await getUserRoles(userId);

        // Check each role
        for (const role of roles) {
            // Check exact permission
            if (role.permissions.includes(permission)) {
                return this.checkResourceAccess(userId, resource);
            }

            // Check wildcard permissions
            if (this.matchesWildcard(permission, role.permissions)) {
                return this.checkResourceAccess(userId, resource);
            }
        }

        return false;
    }

    matchesWildcard(permission, rolePermissions) {
        for (const rolePermission of rolePermissions) {
            if (rolePermission.includes("*")) {
                const pattern = rolePermission.replace("*", ".*");
                if (new RegExp(pattern).test(permission)) {
                    return true;
                }
            }
        }
        return false;
    }
}
```

## Team Management

### Creating and Managing Teams

```javascript
// Team management
class TeamManager {
    async createTeam(teamData) {
        const team = {
            id: generateId(),
            name: teamData.name,
            description: teamData.description,
            created_by: getCurrentUser(),
            created_at: new Date(),
            members: [],
            permissions: teamData.permissions || [],
        };

        // Add creator as team admin
        team.members.push({
            user_id: getCurrentUser(),
            role: "admin",
            added_at: new Date(),
        });

        await saveTeam(team);

        return team;
    }

    async addTeamMember(teamId, userId, role = "member") {
        const team = await getTeam(teamId);

        // Check if already member
        if (team.members.find((m) => m.user_id === userId)) {
            throw new Error("User already in team");
        }

        // Add member
        team.members.push({
            user_id: userId,
            role: role,
            added_at: new Date(),
            added_by: getCurrentUser(),
        });

        await updateTeam(team);

        // Grant team permissions to user
        await this.grantTeamPermissions(userId, team);

        return team;
    }

    async grantTeamPermissions(userId, team) {
        // Get team's project access
        const projects = await getTeamProjects(team.id);

        for (const project of projects) {
            await grantProjectAccess(userId, project.id, {
                permissions: team.permissions,
                environments: team.environments,
                via_team: team.id,
            });
        }
    }
}
```

### Team Access Patterns

```javascript
// Team-based access control
const teamAccessPatterns = {
    // Development team
    development: {
        projects: ["app-backend", "app-frontend"],
        environments: ["development", "staging"],
        permissions: ["secrets.read", "secrets.write", "deployments.create"],
    },

    // Operations team
    operations: {
        projects: ["*"], // All projects
        environments: ["*"], // All environments
        permissions: [
            "secrets.read",
            "environments.*",
            "deployments.*",
            "monitoring.read",
        ],
    },

    // Security team
    security: {
        projects: ["*"],
        environments: ["*"],
        permissions: ["audit.read", "keys.read", "users.read", "security.*"],
    },

    // QA team
    qa: {
        projects: ["app-*"], // All app projects
        environments: ["staging", "qa"],
        permissions: ["secrets.read", "deployments.read", "environments.read"],
    },
};
```

## Access Control Management

### Access Reviews

```javascript
// Automated access review
class AccessReviewManager {
    async scheduleQuarterlyReview() {
        const users = await getAllUsers();

        for (const user of users) {
            const review = {
                user_id: user.id,
                reviewer_id: user.manager_id,
                due_date: getNextQuarterEnd(),
                status: "pending",
                access_summary: await this.generateAccessSummary(user.id),
            };

            await createAccessReview(review);
            await notifyReviewer(review);
        }
    }

    async generateAccessSummary(userId) {
        const summary = {
            roles: await getUserRoles(userId),
            teams: await getUserTeams(userId),
            projects: await getUserProjects(userId),
            last_activity: await getLastActivity(userId),
            permissions: await getEffectivePermissions(userId),
            privileged_access: await getPrivilegedAccess(userId),
        };

        // Flag unusual access
        summary.flags = [];

        if (summary.last_activity > 90) {
            summary.flags.push("inactive_user");
        }

        if (summary.privileged_access.length > 0 && !user.requires_privileged) {
            summary.flags.push("unnecessary_privileged_access");
        }

        return summary;
    }

    async processReviewDecision(reviewId, decision) {
        const review = await getReview(reviewId);

        switch (decision.action) {
            case "approve":
                review.status = "approved";
                break;

            case "modify":
                await this.modifyAccess(review.user_id, decision.changes);
                review.status = "modified";
                break;

            case "revoke":
                await this.revokeAccess(
                    review.user_id,
                    decision.access_to_revoke,
                );
                review.status = "revoked";
                break;
        }

        review.completed_at = new Date();
        review.completed_by = getCurrentUser();

        await updateReview(review);
    }
}
```

### Just-In-Time Access

```javascript
// JIT access implementation
class JITAccessManager {
    async requestAccess(request) {
        const accessRequest = {
            id: generateId(),
            user_id: request.userId,
            resource_type: request.resourceType,
            resource_id: request.resourceId,
            permissions: request.permissions,
            duration: request.duration,
            justification: request.justification,
            status: "pending",
            requested_at: new Date(),
        };

        // Auto-approve based on policy
        if (this.canAutoApprove(accessRequest)) {
            return await this.autoApprove(accessRequest);
        }

        // Route to approver
        await this.routeToApprover(accessRequest);

        return accessRequest;
    }

    canAutoApprove(request) {
        const policy = getJITPolicy();

        // Check if user has pre-approval
        if (policy.pre_approved_users.includes(request.user_id)) {
            return true;
        }

        // Check if resource allows auto-approval
        if (policy.auto_approve_resources.includes(request.resource_id)) {
            return true;
        }

        // Check duration
        if (request.duration <= policy.auto_approve_duration) {
            return true;
        }

        return false;
    }

    async grantTemporaryAccess(request) {
        const grant = {
            user_id: request.user_id,
            permissions: request.permissions,
            resource: {
                type: request.resource_type,
                id: request.resource_id,
            },
            expires_at: new Date(Date.now() + request.duration * 1000),
            granted_by: request.approved_by,
            request_id: request.id,
        };

        // Grant access
        await grantAccess(grant);

        // Schedule revocation
        await scheduleJob(grant.expires_at, async () => {
            await this.revokeTemporaryAccess(grant);
        });

        return grant;
    }
}
```

## User Security

### Multi-Factor Authentication

```javascript
// MFA management
class MFAManager {
    async enforceMFA(userId) {
        const user = await getUser(userId);

        if (!user.mfa_enabled) {
            // Send MFA setup requirement
            await sendMFASetupNotification(userId);

            // Set grace period
            await setMFAGracePeriod(userId, 7); // 7 days

            // Restrict access after grace period
            await scheduleAccessRestriction(userId, 7);
        }
    }

    async setupMFA(userId, method) {
        switch (method) {
            case "totp":
                return await this.setupTOTP(userId);

            case "sms":
                return await this.setupSMS(userId);

            case "webauthn":
                return await this.setupWebAuthn(userId);

            default:
                throw new Error("Unsupported MFA method");
        }
    }

    async setupTOTP(userId) {
        // Generate secret
        const secret = speakeasy.generateSecret({
            length: 32,
            name: `DotEnv (${user.email})`,
            issuer: "DotEnv",
        });

        // Store encrypted secret
        await storeMFASecret(userId, {
            method: "totp",
            secret: encrypt(secret.base32),
            backup_codes: this.generateBackupCodes(),
        });

        return {
            qr_code: secret.otpauth_url,
            manual_entry: secret.base32,
            backup_codes: backup_codes,
        };
    }

    generateBackupCodes(count = 10) {
        const codes = [];

        for (let i = 0; i < count; i++) {
            codes.push(crypto.randomBytes(4).toString("hex"));
        }

        return codes;
    }
}
```

### Session Management

```javascript
// Session security
class SessionManager {
    constructor() {
        this.config = {
            session_timeout: 15 * 60 * 1000, // 15 minutes
            absolute_timeout: 8 * 60 * 60 * 1000, // 8 hours
            concurrent_sessions: 3,
            ip_pinning: true,
        };
    }

    async createSession(userId, context) {
        // Check concurrent sessions
        const activeSessions = await getActiveSessions(userId);

        if (activeSessions.length >= this.config.concurrent_sessions) {
            // Terminate oldest session
            await this.terminateSession(activeSessions[0].id);
        }

        const session = {
            id: generateSessionId(),
            user_id: userId,
            created_at: new Date(),
            last_activity: new Date(),
            expires_at: new Date(Date.now() + this.config.absolute_timeout),
            ip_address: context.ip_address,
            user_agent: context.user_agent,
            device_id: context.device_id,
        };

        await saveSession(session);

        return session;
    }

    async validateSession(sessionId, context) {
        const session = await getSession(sessionId);

        if (!session) {
            throw new Error("Invalid session");
        }

        // Check expiration
        if (new Date() > new Date(session.expires_at)) {
            await this.terminateSession(sessionId);
            throw new Error("Session expired");
        }

        // Check timeout
        const idleTime = Date.now() - new Date(session.last_activity).getTime();
        if (idleTime > this.config.session_timeout) {
            await this.terminateSession(sessionId);
            throw new Error("Session timeout");
        }

        // Check IP pinning
        if (
            this.config.ip_pinning &&
            session.ip_address !== context.ip_address
        ) {
            await this.terminateSession(sessionId);
            throw new Error("IP address mismatch");
        }

        // Update activity
        session.last_activity = new Date();
        await updateSession(session);

        return session;
    }
}
```

## User Lifecycle Management

### User Offboarding

```javascript
// Offboarding process
class UserOffboarding {
    async offboardUser(userId, options = {}) {
        const workflow = {
            user_id: userId,
            initiated_by: getCurrentUser(),
            initiated_at: new Date(),
            steps: [],
        };

        // Immediate actions
        if (!options.delay) {
            workflow.steps.push(
                await this.disableAccount(userId),
                await this.revokeAllSessions(userId),
                await this.revokeAPIKeys(userId),
            );
        }

        // Delayed actions
        const delay = options.delay || 0;

        setTimeout(
            async () => {
                workflow.steps.push(
                    await this.transferOwnership(userId),
                    await this.revokeAllAccess(userId),
                    await this.archiveUserData(userId),
                );

                // Final cleanup
                if (options.delete_after) {
                    setTimeout(
                        async () => {
                            await this.deleteUser(userId);
                        },
                        options.delete_after * 24 * 60 * 60 * 1000,
                    );
                }
            },
            delay * 24 * 60 * 60 * 1000,
        );

        return workflow;
    }

    async transferOwnership(userId) {
        const ownedResources = await getOwnedResources(userId);

        for (const resource of ownedResources) {
            const newOwner = await determineNewOwner(resource);

            await transferResourceOwnership(resource, {
                from: userId,
                to: newOwner,
                reason: "user_offboarding",
            });
        }

        return {
            step: "transfer_ownership",
            resources_transferred: ownedResources.length,
        };
    }
}
```

### Account Recovery

```javascript
// Account recovery system
class AccountRecovery {
    async initiateRecovery(email) {
        const user = await getUserByEmail(email);

        if (!user) {
            // Don't reveal if user exists
            return { message: "If account exists, recovery email sent" };
        }

        const recovery = {
            id: generateId(),
            user_id: user.id,
            token: generateSecureToken(),
            expires_at: new Date(Date.now() + 60 * 60 * 1000), // 1 hour
            method: user.mfa_enabled ? "mfa" : "email",
            attempts: 0,
        };

        await saveRecoveryRequest(recovery);
        await sendRecoveryEmail(user.email, recovery.token);

        return { message: "If account exists, recovery email sent" };
    }

    async validateRecovery(token, verification) {
        const recovery = await getRecoveryByToken(token);

        if (!recovery || new Date() > recovery.expires_at) {
            throw new Error("Invalid or expired recovery token");
        }

        recovery.attempts++;

        if (recovery.attempts > 3) {
            await invalidateRecovery(recovery.id);
            throw new Error("Too many attempts");
        }

        // Verify based on method
        if (recovery.method === "mfa") {
            if (!(await verifyMFA(recovery.user_id, verification.mfa_code))) {
                await updateRecovery(recovery);
                throw new Error("Invalid MFA code");
            }
        }

        // Generate password reset token
        const resetToken = await generatePasswordResetToken(recovery.user_id);

        await invalidateRecovery(recovery.id);

        return { reset_token: resetToken };
    }
}
```

## Monitoring and Reporting

### User Activity Monitoring

```javascript
// Activity monitoring
class UserActivityMonitor {
    async trackActivity(userId, activity) {
        const event = {
            user_id: userId,
            type: activity.type,
            resource: activity.resource,
            action: activity.action,
            timestamp: new Date(),
            ip_address: activity.ip_address,
            user_agent: activity.user_agent,
            success: activity.success,
            metadata: activity.metadata,
        };

        // Store activity
        await storeActivityEvent(event);

        // Check for anomalies
        await this.checkAnomalies(userId, event);

        // Update user metrics
        await this.updateUserMetrics(userId, event);
    }

    async generateActivityReport(userId, period) {
        const activities = await getUserActivities(userId, period);

        const report = {
            user_id: userId,
            period: period,
            summary: {
                total_activities: activities.length,
                login_count: activities.filter((a) => a.type === "login")
                    .length,
                secret_accesses: activities.filter(
                    (a) => a.type === "secret_access",
                ).length,
                api_calls: activities.filter((a) => a.type === "api_call")
                    .length,
            },
            patterns: {
                most_active_hours: this.calculateActiveHours(activities),
                common_resources: this.calculateResourceUsage(activities),
                access_patterns: this.analyzeAccessPatterns(activities),
            },
            anomalies: await this.detectAnomalies(activities),
            risk_score: await this.calculateRiskScore(userId, activities),
        };

        return report;
    }
}
```

### Compliance Reporting

```javascript
// User compliance reports
class UserComplianceReporter {
    async generateUserComplianceReport() {
        const report = {
            generated_at: new Date(),
            compliance_metrics: {
                mfa_adoption: await this.calculateMFAAdoption(),
                password_compliance: await this.assessPasswordCompliance(),
                access_reviews_completed: await this.getAccessReviewMetrics(),
                training_completion: await this.getTrainingMetrics(),
                policy_violations: await this.getPolicyViolations(),
            },

            user_risks: await this.identifyUserRisks(),

            recommendations: await this.generateRecommendations(),
        };

        return report;
    }

    async calculateMFAAdoption() {
        const users = await getAllUsers();
        const mfaEnabled = users.filter((u) => u.mfa_enabled).length;

        return {
            total_users: users.length,
            mfa_enabled: mfaEnabled,
            percentage: ((mfaEnabled / users.length) * 100).toFixed(2),
            by_method: {
                totp: users.filter((u) => u.mfa_method === "totp").length,
                sms: users.filter((u) => u.mfa_method === "sms").length,
                webauthn: users.filter((u) => u.mfa_method === "webauthn")
                    .length,
            },
        };
    }

    async identifyUserRisks() {
        const risks = [];
        const users = await getAllUsers();

        for (const user of users) {
            const userRisks = [];

            // Inactive users with access
            if (user.last_login > 90 && user.active_permissions.length > 0) {
                userRisks.push({
                    type: "inactive_with_access",
                    severity: "medium",
                    recommendation: "Review and possibly revoke access",
                });
            }

            // Excessive permissions
            if (user.permissions.includes("*") && user.role !== "admin") {
                userRisks.push({
                    type: "excessive_permissions",
                    severity: "high",
                    recommendation: "Apply principle of least privilege",
                });
            }

            // No MFA on privileged account
            if (user.privileged_access && !user.mfa_enabled) {
                userRisks.push({
                    type: "privileged_without_mfa",
                    severity: "critical",
                    recommendation: "Enforce MFA immediately",
                });
            }

            if (userRisks.length > 0) {
                risks.push({
                    user_id: user.id,
                    email: user.email,
                    risks: userRisks,
                });
            }
        }

        return risks;
    }
}
```

## Best Practices

### 1. Principle of Least Privilege

```javascript
// ✅ Good: Minimal necessary permissions
const developerAccess = {
    projects: ["project-1", "project-2"],
    environments: ["development", "staging"],
    permissions: ["secrets.read", "deployments.create"],
};

// ❌ Bad: Overly broad permissions
const developerAccess = {
    projects: ["*"],
    environments: ["*"],
    permissions: ["*"],
};
```

### 2. Regular Access Reviews

```javascript
// ✅ Good: Automated quarterly reviews
schedule.every("3 months").do(async () => {
    await accessReviewManager.scheduleQuarterlyReview();
});

// ❌ Bad: No regular reviews
// Access accumulates over time without review
```

### 3. Strong Authentication

```javascript
// ✅ Good: Enforce MFA for all users
const userPolicy = {
    mfa_required: true,
    mfa_enforcement: "immediate",
    password_policy: "strong",
    session_timeout: 15,
};

// ❌ Bad: Optional security measures
const userPolicy = {
    mfa_required: false,
    password_policy: "basic",
};
```

## Resources

- [NIST Identity Management Guidelines](https://www.nist.gov/identity-access-management)
- [Zero Trust Architecture](https://www.nist.gov/publications/zero-trust-architecture)
- [RBAC Best Practices](https://www.nist.gov/publications/role-based-access-control)
- [DotEnv User API](/documentation/v1/api/users)

## Next Steps

- [Configure team permissions](/documentation/v1/core-concepts/access-control)
- [Set up SSO integration](/documentation/v1/administration/sso)
- [Enable audit logging](/documentation/v1/security-compliance/audit-logging)
- [Backup & Restore](/documentation/v1/drafts/administration/backup-restore) *(Coming Soon)*
