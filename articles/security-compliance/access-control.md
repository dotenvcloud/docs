---
title: Access Control
slug: access-control
order: 3
tags: [security, access-control, rbac, permissions]
---

# Access Control

DotEnv implements comprehensive access control mechanisms to ensure your secrets are only accessible to authorized users and systems. This guide covers authentication, authorization, and best practices for managing access.

## Access Control Architecture

### Multi-Layer Security Model

```
┌─────────────────────────────────────────┐
│         Authentication Layer            │
│  (Who are you?)                        │
│  • Username/Password                    │
│  • MFA/2FA                            │
│  • SSO/SAML                           │
│  • API Keys                           │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│         Authorization Layer             │
│  (What can you do?)                    │
│  • Role-Based Access Control           │
│  • Environment Permissions             │
│  • Resource-Level Access               │
│  • Time-Based Restrictions             │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│         Audit Layer                     │
│  (What did you do?)                    │
│  • Access Logs                         │
│  • Change Tracking                     │
│  • Compliance Reports                  │
└─────────────────────────────────────────┘
```

## Authentication

### 1. User Authentication

#### Password Requirements

```javascript
// Password policy configuration
const passwordPolicy = {
    minLength: 12,
    maxLength: 128,
    requireUppercase: true,
    requireLowercase: true,
    requireNumbers: true,
    requireSymbols: true,
    prohibitCommonPasswords: true,
    prohibitUserInfo: true,
    expirationDays: 90,
    historyCount: 5, // Cannot reuse last 5 passwords
};

// Password validation
function validatePassword(password, user) {
    const checks = [
        {
            test: password.length >= passwordPolicy.minLength,
            message: `Password must be at least ${passwordPolicy.minLength} characters`,
        },
        {
            test: /[A-Z]/.test(password),
            message: "Password must contain uppercase letters",
        },
        {
            test: /[a-z]/.test(password),
            message: "Password must contain lowercase letters",
        },
        {
            test: /[0-9]/.test(password),
            message: "Password must contain numbers",
        },
        {
            test: /[!@#$%^&*(),.?":{}|<>]/.test(password),
            message: "Password must contain special characters",
        },
        {
            test: !commonPasswords.includes(password.toLowerCase()),
            message: "Password is too common",
        },
        {
            test: !password.toLowerCase().includes(user.email.split("@")[0]),
            message: "Password cannot contain your username",
        },
    ];

    const failures = checks.filter((check) => !check.test);
    return {
        valid: failures.length === 0,
        errors: failures.map((f) => f.message),
    };
}
```

#### Multi-Factor Authentication (MFA)

```javascript
// MFA configuration
class MFAService {
    // Time-based One-Time Password (TOTP)
    async setupTOTP(user) {
        const secret = authenticator.generateSecret();

        const qrCode = await QRCode.toDataURL(
            authenticator.keyuri(user.email, "DotEnv", secret),
        );

        return {
            secret,
            qrCode,
            backupCodes: this.generateBackupCodes(),
        };
    }

    // SMS-based verification
    async sendSMSCode(user) {
        const code = this.generateCode();

        await sms.send({
            to: user.phone,
            message: `Your DotEnv verification code is: ${code}`,
        });

        await cache.set(
            `mfa:sms:${user.id}`,
            code,
            300, // 5 minutes
        );
    }

    // Hardware key support (WebAuthn)
    async registerSecurityKey(user) {
        const challenge = crypto.randomBytes(32);

        const registrationOptions = {
            challenge,
            rp: {
                name: "DotEnv",
                id: "dotenv.cloud",
            },
            user: {
                id: Buffer.from(user.id),
                name: user.email,
                displayName: user.name,
            },
            authenticatorSelection: {
                authenticatorAttachment: "cross-platform",
                requireResidentKey: false,
                userVerification: "preferred",
            },
        };

        return registrationOptions;
    }

    // Backup codes
    generateBackupCodes(count = 10) {
        return Array.from({ length: count }, () =>
            crypto.randomBytes(4).toString("hex").toUpperCase(),
        );
    }
}
```

### 2. SSO/SAML Integration

```javascript
// SAML configuration
class SAMLProvider {
    constructor(config) {
        this.saml = new SAML({
            callbackUrl: config.callbackUrl,
            entryPoint: config.entryPoint,
            issuer: config.issuer,
            cert: config.idpCert,
            privateCert: config.spPrivateCert,
            decryptionPvk: config.spDecryptionPvk,
            signatureAlgorithm: "sha256",
            authnRequestBinding: "HTTP-POST",
        });
    }

    // Generate SAML request
    async createLoginRequest() {
        return new Promise((resolve, reject) => {
            this.saml.getAuthorizeUrl((err, url) => {
                if (err) reject(err);
                else resolve(url);
            });
        });
    }

    // Validate SAML response
    async validateResponse(samlResponse) {
        return new Promise((resolve, reject) => {
            this.saml.validatePostResponse(samlResponse, (err, profile) => {
                if (err) reject(err);
                else resolve(profile);
            });
        });
    }

    // Map SAML attributes to user
    mapUserAttributes(profile) {
        return {
            email: profile.email || profile.nameID,
            name: profile.displayName || profile.cn,
            groups: profile.groups || [],
            attributes: profile.attributes || {},
        };
    }
}

// OAuth2/OIDC configuration
class OAuthProvider {
    constructor(config) {
        this.client = new OAuth2Client({
            clientId: config.clientId,
            clientSecret: config.clientSecret,
            redirectUri: config.redirectUri,
            authorizationEndpoint: config.authEndpoint,
            tokenEndpoint: config.tokenEndpoint,
            userInfoEndpoint: config.userInfoEndpoint,
        });
    }

    getAuthorizationUrl(state) {
        return this.client.getAuthorizationUrl({
            scope: ["openid", "profile", "email"],
            state: state,
            response_type: "code",
        });
    }

    async exchangeCodeForTokens(code) {
        const tokens = await this.client.exchangeCode(code);
        const userInfo = await this.client.getUserInfo(tokens.access_token);

        return {
            tokens,
            user: {
                id: userInfo.sub,
                email: userInfo.email,
                name: userInfo.name,
                picture: userInfo.picture,
            },
        };
    }
}
```

### 3. API Authentication

```javascript
// API key management
class APIKeyManager {
    // Generate API key
    async createAPIKey(user, options = {}) {
        const key = this.generateSecureKey();
        const hashedKey = await this.hashKey(key);

        const apiKey = await db.apiKeys.create({
            user_id: user.id,
            name: options.name || "API Key",
            key_hash: hashedKey,
            prefix: key.substring(0, 8),
            permissions: options.permissions || ["read"],
            expires_at: options.expiresAt,
            ip_whitelist: options.ipWhitelist || [],
            rate_limit: options.rateLimit || 1000,
        });

        // Return key only once
        return {
            id: apiKey.id,
            key: `dotenv_${key}`, // Full key shown only on creation
            prefix: apiKey.prefix,
            created_at: apiKey.created_at,
        };
    }

    generateSecureKey() {
        return crypto.randomBytes(32).toString("base64url");
    }

    async hashKey(key) {
        return crypto.createHash("sha256").update(key).digest("hex");
    }

    // Validate API key
    async validateAPIKey(key, request) {
        if (!key.startsWith("dotenv_")) {
            throw new Error("Invalid API key format");
        }

        const keyValue = key.substring(7); // Remove prefix
        const hashedKey = await this.hashKey(keyValue);

        const apiKey = await db.apiKeys.findOne({
            key_hash: hashedKey,
            deleted_at: null,
        });

        if (!apiKey) {
            throw new Error("Invalid API key");
        }

        // Check expiration
        if (apiKey.expires_at && new Date() > apiKey.expires_at) {
            throw new Error("API key expired");
        }

        // Check IP whitelist
        if (apiKey.ip_whitelist.length > 0) {
            const clientIP = request.ip;
            if (!apiKey.ip_whitelist.includes(clientIP)) {
                throw new Error("IP not whitelisted");
            }
        }

        // Check rate limit
        await this.checkRateLimit(apiKey, request);

        // Update last used
        await db.apiKeys.update(apiKey.id, {
            last_used_at: new Date(),
            usage_count: apiKey.usage_count + 1,
        });

        return apiKey;
    }
}
```

## Authorization

### 1. Role-Based Access Control (RBAC)

```javascript
// Role definitions
const roles = {
    owner: {
        name: "Owner",
        permissions: ["*"], // All permissions
        inherits: [],
    },

    admin: {
        name: "Administrator",
        permissions: [
            "organization:update",
            "project:*",
            "team:*",
            "user:*",
            "secret:*",
            "key:*",
            "audit:read",
        ],
        inherits: ["developer"],
    },

    developer: {
        name: "Developer",
        permissions: [
            "project:read",
            "secret:read",
            "secret:write",
            "secret:decrypt",
            "environment:read",
        ],
        inherits: ["viewer"],
    },

    devops: {
        name: "DevOps Engineer",
        permissions: [
            "project:read",
            "secret:*",
            "key:retrieve",
            "key:rotate",
            "environment:*",
            "deployment:*",
        ],
        inherits: ["developer"],
    },

    viewer: {
        name: "Viewer",
        permissions: [
            "organization:read",
            "project:read",
            "secret:read",
            "audit:read",
        ],
        inherits: [],
    },
};

// Permission checker
class PermissionChecker {
    constructor(user) {
        this.user = user;
        this.permissions = this.expandPermissions();
    }

    expandPermissions() {
        const userRoles = this.user.roles || [];
        const permissions = new Set();

        userRoles.forEach((roleName) => {
            this.addRolePermissions(roleName, permissions);
        });

        // Add user-specific permissions
        (this.user.permissions || []).forEach((p) => permissions.add(p));

        return Array.from(permissions);
    }

    addRolePermissions(roleName, permissions) {
        const role = roles[roleName];
        if (!role) return;

        // Add role permissions
        role.permissions.forEach((p) => {
            if (p.includes("*")) {
                // Expand wildcards
                const prefix = p.replace("*", "");
                this.expandWildcard(prefix, permissions);
            } else {
                permissions.add(p);
            }
        });

        // Add inherited permissions
        role.inherits.forEach((inherited) => {
            this.addRolePermissions(inherited, permissions);
        });
    }

    can(permission, resource = null) {
        // Check wildcard permission
        if (this.permissions.includes("*")) return true;

        // Check exact permission
        if (this.permissions.includes(permission)) return true;

        // Check wildcard patterns
        const [resource_type, action] = permission.split(":");
        if (this.permissions.includes(`${resource_type}:*`)) return true;

        // Check resource-specific permissions
        if (resource) {
            return this.checkResourcePermission(permission, resource);
        }

        return false;
    }

    checkResourcePermission(permission, resource) {
        // Check ownership
        if (resource.owner_id === this.user.id) {
            return true; // Owners have full access
        }

        // Check team membership
        if (resource.team_id) {
            const membership = this.user.team_memberships.find(
                (m) => m.team_id === resource.team_id,
            );

            if (membership) {
                // Check team role permissions
                return this.checkTeamPermission(permission, membership.role);
            }
        }

        return false;
    }
}
```

### 2. Environment-Based Permissions

```javascript
// Environment permission matrix
class EnvironmentPermissions {
    constructor() {
        this.matrix = {
            development: {
                roles: ["developer", "devops", "admin", "owner"],
                restrictions: {
                    "secret:decrypt": true,
                    "secret:write": true,
                    "key:rotate": false,
                },
            },

            staging: {
                roles: ["devops", "admin", "owner"],
                restrictions: {
                    "secret:decrypt": true,
                    "secret:write": true,
                    "key:rotate": true,
                },
            },

            production: {
                roles: ["devops", "admin", "owner"],
                restrictions: {
                    "secret:decrypt": false, // Requires additional approval
                    "secret:write": false, // Requires change request
                    "key:rotate": true,
                },
                approvals: {
                    "secret:decrypt": {
                        required: 2,
                        approvers: ["admin", "owner"],
                        timeout: 3600, // 1 hour
                    },
                    "secret:write": {
                        required: 2,
                        approvers: ["admin", "owner"],
                        timeout: 86400, // 24 hours
                    },
                },
            },
        };
    }

    canAccess(user, environment, permission) {
        const config = this.matrix[environment];
        if (!config) return false;

        // Check role access
        const userRoles = user.roles || [];
        const hasRole = userRoles.some((role) => config.roles.includes(role));

        if (!hasRole) return false;

        // Check restrictions
        if (config.restrictions[permission] === false) {
            // Check if approval exists
            return this.hasApproval(user, environment, permission);
        }

        return true;
    }

    async requestApproval(user, environment, permission, context) {
        const config = this.matrix[environment];
        const approvalConfig = config.approvals?.[permission];

        if (!approvalConfig) {
            throw new Error("Approval not required for this action");
        }

        const approval = await db.approvals.create({
            requester_id: user.id,
            environment,
            permission,
            context,
            required_approvals: approvalConfig.required,
            expires_at: new Date(Date.now() + approvalConfig.timeout * 1000),
        });

        // Notify approvers
        await this.notifyApprovers(approval, approvalConfig.approvers);

        return approval;
    }
}
```

### 3. Attribute-Based Access Control (ABAC)

```javascript
// ABAC policy engine
class ABACEngine {
    constructor() {
        this.policies = [];
        this.loadPolicies();
    }

    loadPolicies() {
        this.policies = [
            {
                name: "time-based-access",
                description: "Restrict access outside business hours",
                evaluate: (context) => {
                    const hour = new Date().getHours();
                    const isBusinessHours = hour >= 9 && hour < 18;
                    const isWeekday =
                        new Date().getDay() >= 1 && new Date().getDay() <= 5;

                    // Allow unrestricted access for on-call
                    if (context.user.attributes.onCall) return true;

                    // Restrict production access outside business hours
                    if (context.resource.environment === "production") {
                        return isBusinessHours && isWeekday;
                    }

                    return true;
                },
            },

            {
                name: "location-based-access",
                description: "Restrict access based on location",
                evaluate: (context) => {
                    const userCountry = context.request.geoip.country;
                    const allowedCountries =
                        context.resource.allowedCountries || [];

                    // No restriction if not specified
                    if (allowedCountries.length === 0) return true;

                    return allowedCountries.includes(userCountry);
                },
            },

            {
                name: "compliance-based-access",
                description: "Enforce compliance requirements",
                evaluate: (context) => {
                    // GDPR compliance
                    if (context.resource.dataClassification === "pii") {
                        return context.user.attributes.gdprTrained === true;
                    }

                    // HIPAA compliance
                    if (context.resource.dataClassification === "phi") {
                        return context.user.attributes.hipaaTrained === true;
                    }

                    return true;
                },
            },
        ];
    }

    evaluate(context) {
        const results = this.policies.map((policy) => ({
            name: policy.name,
            result: policy.evaluate(context),
        }));

        const denied = results.filter((r) => !r.result);

        return {
            allowed: denied.length === 0,
            denied: denied.map((d) => d.name),
            context,
        };
    }
}
```

## Access Patterns

### 1. Just-In-Time (JIT) Access

```javascript
// JIT access implementation
class JITAccess {
    async requestAccess(user, resource, duration) {
        // Validate request
        const validation = await this.validateRequest(user, resource);
        if (!validation.valid) {
            throw new Error(validation.reason);
        }

        // Create temporary grant
        const grant = await db.temporaryGrants.create({
            user_id: user.id,
            resource_id: resource.id,
            permissions: this.getRequestedPermissions(resource),
            expires_at: new Date(Date.now() + duration * 1000),
            reason: validation.reason,
            approved_by: validation.auto_approved ? "system" : null,
        });

        // Auto-approve if criteria met
        if (validation.auto_approved) {
            await this.activateGrant(grant);
        } else {
            await this.requestApproval(grant);
        }

        return grant;
    }

    async validateRequest(user, resource) {
        // Check if user has base permissions
        const hasBaseAccess = await this.checkBaseAccess(user, resource);

        // Check risk score
        const riskScore = await this.calculateRiskScore(user, resource);

        // Auto-approve low-risk requests
        const auto_approved = hasBaseAccess && riskScore < 30;

        return {
            valid: true,
            auto_approved,
            reason: user.request_reason,
            risk_score: riskScore,
        };
    }

    async activateGrant(grant) {
        // Add temporary permissions
        await cache.set(
            `jit:${grant.user_id}:${grant.resource_id}`,
            grant.permissions,
            grant.expires_at - Date.now(),
        );

        // Log activation
        await this.auditLog("jit_access_granted", grant);

        // Schedule revocation
        scheduler.at(grant.expires_at).do(() => {
            this.revokeGrant(grant);
        });
    }
}
```

### 2. Break-Glass Access

```javascript
// Emergency access procedures
class BreakGlassAccess {
    async initiateBreakGlass(user, resource, reason) {
        // Verify emergency
        if (!this.isValidEmergency(reason)) {
            throw new Error("Invalid emergency reason");
        }

        // Create break-glass session
        const session = await db.breakGlassSessions.create({
            user_id: user.id,
            resource_id: resource.id,
            reason: reason,
            started_at: new Date(),
            emergency_code: this.generateEmergencyCode(),
        });

        // Grant immediate access
        await this.grantEmergencyAccess(user, resource);

        // Notify security team
        await this.notifySecurityTeam(session);

        // Start recording session
        await this.startSessionRecording(session);

        return session;
    }

    async grantEmergencyAccess(user, resource) {
        // Override all restrictions
        await cache.set(`break_glass:${user.id}`, {
            permissions: ["*"],
            resources: [resource.id],
            expires_at: Date.now() + 3600000, // 1 hour
        });

        // Log with high priority
        await this.auditLog("break_glass_activated", {
            user_id: user.id,
            resource_id: resource.id,
            severity: "critical",
        });
    }

    async endBreakGlass(session) {
        // Revoke emergency access
        await cache.del(`break_glass:${session.user_id}`);

        // End session recording
        const recording = await this.endSessionRecording(session);

        // Generate report
        const report = await this.generateReport(session, recording);

        // Schedule review
        await this.scheduleSecurityReview(report);

        return report;
    }
}
```

## Access Reviews

### 1. Automated Reviews

```javascript
// Access review automation
class AccessReviewAutomation {
    async runQuarterlyReview() {
        const reviews = [];

        // Get all users with access
        const users = await db.users.findAll({
            last_active: {
                $gte: new Date(Date.now() - 90 * 24 * 60 * 60 * 1000),
            },
        });

        for (const user of users) {
            const review = await this.reviewUserAccess(user);
            reviews.push(review);
        }

        // Generate report
        const report = this.generateReviewReport(reviews);

        // Send to managers
        await this.distributeReport(report);

        return report;
    }

    async reviewUserAccess(user) {
        const access = {
            user_id: user.id,
            roles: user.roles,
            permissions: await this.getUserPermissions(user),
            resources: await this.getUserResources(user),
            last_activity: await this.getLastActivity(user),
            recommendations: [],
        };

        // Check for excessive permissions
        if (this.hasExcessivePermissions(access)) {
            access.recommendations.push({
                type: "reduce_permissions",
                reason: "User has not used these permissions in 90 days",
                permissions: this.getUnusedPermissions(access),
            });
        }

        // Check for inactive access
        if (this.isInactive(access.last_activity)) {
            access.recommendations.push({
                type: "revoke_access",
                reason: "User has been inactive for 90 days",
            });
        }

        return access;
    }
}
```

### 2. Certification Process

```javascript
// Access certification workflow
class AccessCertification {
    async initiateCertification(scope) {
        const campaign = await db.certificationCampaigns.create({
            name: `Q${this.getQuarter()} ${new Date().getFullYear()} Access Review`,
            scope: scope,
            due_date: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
            status: "active",
        });

        // Identify certifiers
        const certifiers = await this.identifyCertifiers(scope);

        // Create certification tasks
        for (const certifier of certifiers) {
            const items = await this.getItemsForCertifier(certifier, scope);

            await db.certificationTasks.create({
                campaign_id: campaign.id,
                certifier_id: certifier.id,
                items: items,
                status: "pending",
            });
        }

        // Send notifications
        await this.notifyCertifiers(campaign, certifiers);

        return campaign;
    }

    async certifyAccess(taskId, decisions) {
        const task = await db.certificationTasks.findById(taskId);

        for (const decision of decisions) {
            if (decision.action === "revoke") {
                await this.scheduleRevocation(decision.item, decision.reason);
            } else if (decision.action === "modify") {
                await this.scheduleModification(
                    decision.item,
                    decision.changes,
                );
            }

            // Log decision
            await db.certificationDecisions.create({
                task_id: taskId,
                item_id: decision.item.id,
                action: decision.action,
                reason: decision.reason,
                certifier_id: task.certifier_id,
            });
        }

        // Update task status
        await db.certificationTasks.update(taskId, {
            status: "completed",
            completed_at: new Date(),
        });
    }
}
```

## Best Practices

### 1. Principle of Least Privilege

```javascript
// ✅ Good: Minimal permissions
const developerRole = {
    permissions: [
        "project:read",
        "secret:read",
        "environment:development:write",
    ],
};

// ❌ Bad: Excessive permissions
const developerRole = {
    permissions: ["*"],
};
```

### 2. Defense in Depth

```javascript
// Multiple layers of security
async function accessSecret(user, secretId) {
    // Layer 1: Authentication
    if (!user.authenticated) {
        throw new Error("Not authenticated");
    }

    // Layer 2: MFA verification
    if (secret.classification === "critical" && !user.mfaVerified) {
        throw new Error("MFA required for critical secrets");
    }

    // Layer 3: Authorization
    if (!user.can("secret:read", secret)) {
        throw new Error("Not authorized");
    }

    // Layer 4: Environment check
    if (!environmentPermissions.canAccess(user, secret.environment)) {
        throw new Error("No access to this environment");
    }

    // Layer 5: Time-based restrictions
    if (!isWithinAccessHours(user, secret)) {
        throw new Error("Outside access hours");
    }

    // Layer 6: Audit logging
    await auditLog("secret_accessed", {
        user_id: user.id,
        secret_id: secretId,
        timestamp: new Date(),
    });

    return secret;
}
```

### 3. Regular Reviews

```javascript
// Automated access review
schedule.every("3 months").do(async () => {
    const review = await accessReview.runQuarterlyReview();

    // Remove unused access
    review.recommendations
        .filter((r) => r.type === "revoke_access")
        .forEach((r) => revokeAccess(r.user_id));

    // Alert on high-risk findings
    review.findings
        .filter((f) => f.risk === "high")
        .forEach((f) => alertSecurityTeam(f));
});
```

## Troubleshooting

### Common Issues

1. **Permission Denied**

    - Verify user roles and permissions
    - Check environment-specific restrictions
    - Review ABAC policies
    - Check for expired grants

2. **MFA Issues**

    - Verify time sync for TOTP
    - Check backup codes
    - Review device registration

3. **SSO Problems**
    - Validate SAML configuration
    - Check certificate expiration
    - Review attribute mapping

## Resources

- [NIST Access Control Guide](https://csrc.nist.gov/publications/detail/sp/800-162/final)
- [RBAC Best Practices](https://www.nist.gov/blogs/cybersecurity-insights/role-based-access-control-rbac)
- [Zero Trust Architecture](https://www.nist.gov/publications/zero-trust-architecture)
- [DotEnv Access Control API](/documentation/v1/api/access-control)

## Next Steps

- [Configure audit logging](/documentation/v1/security-compliance/audit-logging)
- [Review compliance requirements](/documentation/v1/security-compliance/compliance)
- [Implement security best practices](/documentation/v1/security-compliance/security-best-practices)
- [Set up incident response](/documentation/v1/security-compliance/incident-response)
