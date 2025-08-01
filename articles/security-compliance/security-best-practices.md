---
title: Security Best Practices
slug: security-best-practices
order: 6
tags: [security, best-practices, guidelines, recommendations]
---

# Security Best Practices

This guide provides comprehensive security best practices for using DotEnv effectively while maintaining the highest security standards. Follow these recommendations to protect your secrets and maintain a secure environment.

## Organization Security

### 1. Account Security

#### Strong Authentication

```javascript
// ✅ Recommended: Enforce strong authentication
const authenticationPolicy = {
    password: {
        minLength: 14,
        requireUppercase: true,
        requireLowercase: true,
        requireNumbers: true,
        requireSymbols: true,
        prohibitCommonPasswords: true,
        prohibitPersonalInfo: true,
        expirationDays: 90,
        historyCount: 12, // Cannot reuse last 12 passwords
    },

    mfa: {
        required: true,
        allowedMethods: ["totp", "webauthn", "sms"],
        gracePeriodDays: 7, // For new accounts
        backupCodes: 10,
    },

    session: {
        timeoutMinutes: 15,
        absoluteTimeoutHours: 8,
        concurrentSessions: 3,
        ipPinning: true,
    },
};

// ❌ Avoid: Weak authentication
const weakAuth = {
    password: { minLength: 8 },
    mfa: { required: false },
    session: { timeout: null },
};
```

#### Account Management

```javascript
// Regular account reviews
class AccountSecurityManager {
    async performQuarterlyReview() {
        const users = await this.getAllUsers();

        for (const user of users) {
            // Check last activity
            if (this.isInactive(user, 90)) {
                await this.disableAccount(user);
                await this.notifyUser(user, "account_disabled_inactivity");
            }

            // Check privileged access
            if (this.hasPrivilegedAccess(user)) {
                await this.requestAccessJustification(user);
            }

            // Check MFA status
            if (!user.mfaEnabled) {
                await this.enforceMFA(user);
            }

            // Check password age
            if (this.passwordExpired(user)) {
                await this.forcePasswordReset(user);
            }
        }
    }
}
```

### 2. Team Structure

#### Role-Based Access

```yaml
# teams/security-roles.yaml
roles:
    # Minimal access for most users
    developer:
        name: Developer
        permissions:
            - project:read
            - secret:read
            - secret:write:development
            - environment:read
        restrictions:
            - no_production_write
            - no_key_management
            - no_audit_access

    # Elevated access for deployments
    devops:
        name: DevOps Engineer
        permissions:
            - project:read
            - secret:read
            - secret:write
            - environment:*
            - key:retrieve
        restrictions:
            - no_user_management
            - no_billing_access

    # Security team access
    security:
        name: Security Engineer
        permissions:
            - audit:read
            - key:rotate
            - compliance:read
            - incident:manage
        restrictions:
            - no_secret_values
            - read_only_config
```

#### Separation of Duties

```javascript
// Implement separation of duties
const separationOfDuties = {
    // Different people for different tasks
    key_management: {
        generation: ["security_team"],
        rotation: ["security_team", "devops_lead"],
        storage: ["automated_system"],
        retrieval: ["authorized_applications"],
    },

    // Approval requirements
    sensitive_operations: {
        production_deployment: {
            approvers: 2,
            roles: ["devops_lead", "security_engineer"],
            time_window: "24_hours",
        },

        key_rotation: {
            approvers: 2,
            roles: ["security_engineer", "security_manager"],
            notification: "all_stakeholders",
        },

        user_privilege_elevation: {
            approvers: 1,
            roles: ["team_manager", "security_team"],
            justification: "required",
            duration: "time_bound",
        },
    },
};
```

## Secret Management

### 1. Secret Hygiene

#### Naming Conventions

```javascript
// ✅ Good: Clear, consistent naming
const goodSecretNames = [
    "DATABASE_URL",
    "REDIS_CONNECTION_STRING",
    "STRIPE_API_KEY",
    "AWS_ACCESS_KEY_ID",
    "SMTP_PASSWORD",
    "JWT_SIGNING_KEY",
];

// ❌ Bad: Ambiguous or inconsistent naming
const badSecretNames = [
    "password", // Which password?
    "key", // What kind of key?
    "prod_db", // Missing what it contains
    "test123", // Meaningless name
    "temp", // Suggests insecure usage
    "APIKey", // Inconsistent casing
];

// Naming convention enforcer
function validateSecretName(name) {
    const rules = [
        { pattern: /^[A-Z][A-Z0-9_]*$/, message: "Use UPPER_SNAKE_CASE" },
        { pattern: /^.{3,}$/, message: "Minimum 3 characters" },
        { pattern: /^(?!TEMP|TEST)/, message: "No temporary prefixes" },
        {
            pattern: /_(?:KEY|TOKEN|PASSWORD|SECRET|URL)$/,
            message: "Include type suffix",
        },
    ];

    for (const rule of rules) {
        if (!rule.pattern.test(name)) {
            throw new Error(`Invalid secret name: ${rule.message}`);
        }
    }
}
```

#### Secret Rotation

```javascript
// Automated secret rotation
class SecretRotationManager {
    constructor() {
        this.rotationPolicies = {
            api_keys: { days: 90, automatic: true },
            passwords: { days: 60, automatic: true },
            certificates: { days: 30, automatic: true }, // Before expiry
            encryption_keys: { days: 365, automatic: false },
            webhook_secrets: { days: 180, automatic: true },
        };
    }

    async rotateSecret(secretName, secretType) {
        // Generate new secret
        const newSecret = await this.generateSecret(secretType);

        // Update in stages
        const stages = [
            {
                name: "add_new",
                action: () => this.addNewSecret(secretName, newSecret),
            },
            {
                name: "test_new",
                action: () => this.testNewSecret(secretName, newSecret),
            },
            {
                name: "switch_primary",
                action: () => this.switchPrimary(secretName, newSecret),
            },
            {
                name: "deprecate_old",
                action: () => this.deprecateOld(secretName),
            },
            {
                name: "remove_old",
                action: () => this.removeOld(secretName),
                delay: 7 * 24 * 60 * 60 * 1000,
            },
        ];

        for (const stage of stages) {
            try {
                if (stage.delay) {
                    await this.scheduleStage(stage);
                } else {
                    await stage.action();
                }

                await this.audit(`rotation_stage_complete: ${stage.name}`, {
                    secretName,
                });
            } catch (error) {
                await this.rollback(secretName, stage.name);
                throw error;
            }
        }
    }
}
```

### 2. Secret Classification

#### Data Classification

```javascript
// Secret classification system
const secretClassification = {
    critical: {
        description: "Business-critical secrets",
        examples: [
            "production_database_password",
            "payment_processor_key",
            "master_encryption_key",
        ],
        controls: {
            encryption: "required",
            key_type: "client_managed",
            access: "break_glass_only",
            rotation: "quarterly",
            audit: "every_access",
        },
    },

    high: {
        description: "Sensitive operational secrets",
        examples: ["api_keys", "service_passwords", "jwt_signing_keys"],
        controls: {
            encryption: "required",
            key_type: "server_managed",
            access: "role_based",
            rotation: "semi_annual",
            audit: "daily",
        },
    },

    medium: {
        description: "Internal service secrets",
        examples: ["development_passwords", "test_api_keys", "webhook_secrets"],
        controls: {
            encryption: "required",
            key_type: "server_managed",
            access: "team_based",
            rotation: "annual",
            audit: "weekly",
        },
    },

    low: {
        description: "Non-sensitive configuration",
        examples: ["feature_flags", "public_api_endpoints", "timeout_values"],
        controls: {
            encryption: "optional",
            key_type: "any",
            access: "project_wide",
            rotation: "as_needed",
            audit: "monthly",
        },
    },
};
```

### 3. Environment Isolation

#### Environment Security Levels

```javascript
// Environment-specific security controls
const environmentSecurity = {
    production: {
        access: {
            authentication: ["mfa_required"],
            authorization: ["role:devops", "role:sre"],
            approval: "required",
            audit: "real_time",
        },

        secrets: {
            encryption: "client_managed",
            rotation: "automated",
            backup: "continuous",
            versioning: true,
        },

        network: {
            ip_whitelist: true,
            vpn_required: true,
            rate_limiting: "strict",
        },
    },

    staging: {
        access: {
            authentication: ["password"],
            authorization: ["role:developer", "role:qa"],
            approval: "not_required",
            audit: "daily",
        },

        secrets: {
            encryption: "server_managed",
            rotation: "manual",
            backup: "daily",
            versioning: true,
        },
    },

    development: {
        access: {
            authentication: ["password"],
            authorization: ["team:engineering"],
            approval: "not_required",
            audit: "weekly",
        },

        secrets: {
            encryption: "server_managed",
            rotation: "optional",
            backup: "weekly",
            versioning: false,
        },
    },
};
```

## Application Security

### 1. Secure Integration

#### SDK Usage

```javascript
// ✅ Secure SDK initialization
import { DotEnvClient } from "@dotenv/sdk";

const client = new DotEnvClient({
    // Use environment variable for API key
    apiKey: process.env.DOTENV_API_KEY,

    // Enable security features
    encryption: {
        enabled: true,
        keyManagement: "client",
    },

    // Configure timeouts
    timeout: 5000,
    retries: 3,

    // Enable certificate pinning
    tlsConfig: {
        pinCertificates: true,
        minVersion: "TLSv1.3",
    },
});

// ❌ Insecure initialization
const client = new DotEnvClient({
    apiKey: "hardcoded_api_key_in_source", // Never hardcode!
    encryption: { enabled: false },
    tlsConfig: { verify: false },
});
```

#### Error Handling

```javascript
// ✅ Secure error handling
async function loadSecrets() {
    try {
        const secrets = await client.secrets.list("production");
        return secrets;
    } catch (error) {
        // Log error without sensitive data
        logger.error("Failed to load secrets", {
            error: error.message,
            code: error.code,
            // Don't log: API keys, secret values, full stack traces
        });

        // Fail securely
        if (isProduction()) {
            // Don't start application without secrets
            process.exit(1);
        } else {
            // Use safe defaults in development
            return loadDevelopmentDefaults();
        }
    }
}

// ❌ Insecure error handling
try {
    const secrets = await client.secrets.list("production");
} catch (error) {
    console.error("Full error:", error); // Might expose sensitive data
    // Continue running without secrets - security risk!
}
```

### 2. Runtime Security

#### Memory Protection

```javascript
// Secure secret handling in memory
class SecureSecretHandler {
    constructor() {
        this.secrets = new Map();
        this.accessLog = [];
    }

    setSecret(key, value) {
        // Store encrypted even in memory
        const encrypted = this.encryptInMemory(value);
        this.secrets.set(key, encrypted);

        // Schedule cleanup
        this.scheduleCleanup(key);
    }

    getSecret(key) {
        const encrypted = this.secrets.get(key);
        if (!encrypted) return null;

        // Decrypt on demand
        const value = this.decryptInMemory(encrypted);

        // Log access
        this.logAccess(key);

        // Return copy, not reference
        return value.slice();
    }

    clearSecret(key) {
        const encrypted = this.secrets.get(key);
        if (encrypted) {
            // Overwrite memory
            crypto.randomFillSync(encrypted);
            this.secrets.delete(key);
        }
    }

    clearAll() {
        for (const key of this.secrets.keys()) {
            this.clearSecret(key);
        }
    }
}

// Use in application
process.on("SIGTERM", () => {
    secretHandler.clearAll();
    process.exit(0);
});
```

#### Secure Logging

```javascript
// Log sanitizer
class SecureLogger {
    constructor() {
        this.sensitivePatterns = [
            /password=[\w\d]+/gi,
            /api[_-]?key=[\w\d]+/gi,
            /token=[\w\d]+/gi,
            /secret=[\w\d]+/gi,
            /Bearer\s+[\w\d\-\.]+/gi,
            /Basic\s+[\w\d\+\/=]+/gi,
        ];
    }

    sanitize(message) {
        let sanitized = message;

        for (const pattern of this.sensitivePatterns) {
            sanitized = sanitized.replace(pattern, "[REDACTED]");
        }

        return sanitized;
    }

    log(level, message, context = {}) {
        const sanitizedMessage = this.sanitize(message);
        const sanitizedContext = this.sanitizeObject(context);

        console.log({
            timestamp: new Date().toISOString(),
            level,
            message: sanitizedMessage,
            context: sanitizedContext,
        });
    }

    sanitizeObject(obj) {
        const sensitiveKeys = [
            "password",
            "secret",
            "key",
            "token",
            "authorization",
        ];
        const sanitized = {};

        for (const [key, value] of Object.entries(obj)) {
            if (sensitiveKeys.some((sk) => key.toLowerCase().includes(sk))) {
                sanitized[key] = "[REDACTED]";
            } else if (typeof value === "object" && value !== null) {
                sanitized[key] = this.sanitizeObject(value);
            } else {
                sanitized[key] = this.sanitize(String(value));
            }
        }

        return sanitized;
    }
}
```

## CI/CD Security

### 1. Pipeline Security

#### Secure CI/CD Configuration

```yaml
# .github/workflows/secure-deployment.yml
name: Secure Deployment

on:
    push:
        branches: [main]

env:
    # No secrets in workflow files!
    NODE_ENV: production

jobs:
    deploy:
        runs-on: ubuntu-latest

        # Use deployment environment for protection rules
        environment: production

        permissions:
            contents: read
            id-token: write # For OIDC

        steps:
            - uses: actions/checkout@v3

            - name: Configure AWS credentials
              uses: aws-actions/configure-aws-credentials@v2
              with:
                  role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE }}
                  aws-region: us-east-1

            - name: Install DotEnv CLI
              run: |
                  curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                  echo "$HOME/.dotenv/bin" >> $GITHUB_PATH

            - name: Load secrets
              env:
                  DOTENV_API_KEY: ${{ secrets.DOTENV_API_KEY }}
              run: |
                  # Use CLI with secure options
                  dotenv pull production \
                    --no-interactive \
                    --fail-on-missing \
                    --output .env.production

                  # Verify secrets loaded
                  dotenv validate .env.production

            - name: Build application
              run: |
                  # Secrets available as environment variables
                  npm run build

            - name: Clean up
              if: always()
              run: |
                  # Remove secrets from disk
                  shred -vfz .env.production || rm -f .env.production

                  # Clear Docker secrets
                  docker system prune -af
```

#### Container Security

```dockerfile
# Secure Dockerfile
FROM node:18-alpine AS builder

# Don't include secrets in image
ARG BUILD_VERSION
ENV NODE_ENV=production

# Use non-root user
RUN addgroup -g 1001 nodejs && \
    adduser -S -u 1001 -G nodejs nodejs

# Copy only necessary files
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY --chown=nodejs:nodejs . .

# Multi-stage build to minimize attack surface
FROM gcr.io/distroless/nodejs18-debian11

COPY --from=builder --chown=1001:1001 /app /app

WORKDIR /app
USER 1001

# Load secrets at runtime, not build time
ENTRYPOINT ["node", "src/index.js"]
```

### 2. Deployment Security

#### Secure Deployment Script

```bash
#!/bin/bash
# deploy-secure.sh

set -euo pipefail  # Fail on errors

# Load secrets securely
load_secrets() {
  echo "Loading production secrets..."

  # Use temporary file with restricted permissions
  TEMP_ENV=$(mktemp -t secrets.XXXXXX)
  chmod 600 "$TEMP_ENV"

  # Load from DotEnv
  if ! dotenv pull production --output "$TEMP_ENV"; then
    echo "Failed to load secrets" >&2
    rm -f "$TEMP_ENV"
    exit 1
  fi

  # Source secrets
  set -a
  source "$TEMP_ENV"
  set +a

  # Clean up immediately
  shred -vfz "$TEMP_ENV" || rm -f "$TEMP_ENV"
}

# Validate environment
validate_environment() {
  required_vars=(
    "DATABASE_URL"
    "REDIS_URL"
    "API_KEY"
    "JWT_SECRET"
  )

  for var in "${required_vars[@]}"; do
    if [ -z "${!var:-}" ]; then
      echo "Missing required variable: $var" >&2
      exit 1
    fi
  done
}

# Deploy with rollback capability
deploy_with_rollback() {
  # Backup current deployment
  echo "Creating backup..."
  kubectl create configmap "backup-$(date +%s)" \
    --from-literal=deployment="$(kubectl get deployment app -o yaml)"

  # Deploy new version
  echo "Deploying new version..."
  if ! kubectl apply -f k8s/; then
    echo "Deployment failed, initiating rollback..." >&2
    kubectl rollout undo deployment/app
    exit 1
  fi

  # Wait for healthy state
  if ! kubectl rollout status deployment/app --timeout=300s; then
    echo "Deployment unhealthy, rolling back..." >&2
    kubectl rollout undo deployment/app
    exit 1
  fi
}

# Main deployment flow
main() {
  load_secrets
  validate_environment
  deploy_with_rollback

  echo "Deployment completed successfully"
}

main "$@"
```

## Monitoring and Detection

### 1. Security Monitoring

#### Anomaly Detection

```javascript
// Security anomaly detector
class SecurityAnomalyDetector {
    constructor() {
        this.patterns = {
            mass_secret_access: {
                threshold: 50,
                window: 300, // 5 minutes
                severity: "critical",
            },

            unusual_access_time: {
                normal_hours: { start: 6, end: 22 },
                severity: "medium",
            },

            geographic_anomaly: {
                baseline_locations: ["US", "EU"],
                severity: "high",
            },

            failed_auth_spike: {
                threshold: 10,
                window: 600, // 10 minutes
                severity: "high",
            },
        };
    }

    async detectAnomalies(event) {
        const anomalies = [];

        // Check each pattern
        if (await this.detectMassAccess(event.user_id)) {
            anomalies.push({
                type: "mass_secret_access",
                severity: "critical",
                action: "block_user",
            });
        }

        if (this.detectUnusualTime(event.timestamp)) {
            anomalies.push({
                type: "unusual_access_time",
                severity: "medium",
                action: "additional_verification",
            });
        }

        if (await this.detectGeographicAnomaly(event.ip_address)) {
            anomalies.push({
                type: "geographic_anomaly",
                severity: "high",
                action: "require_mfa",
            });
        }

        // Take action on anomalies
        for (const anomaly of anomalies) {
            await this.handleAnomaly(anomaly, event);
        }

        return anomalies;
    }
}
```

#### Security Metrics

```javascript
// Security metrics dashboard
const securityMetrics = {
    authentication: {
        successful_logins: new Counter("auth_success_total"),
        failed_logins: new Counter("auth_failed_total"),
        mfa_usage: new Gauge("mfa_usage_percentage"),
        password_age: new Histogram("password_age_days"),
    },

    secrets: {
        access_rate: new Counter("secret_access_total"),
        rotation_compliance: new Gauge("secret_rotation_compliance"),
        encryption_coverage: new Gauge("secret_encryption_percentage"),
        classification_compliance: new Gauge(
            "secret_classification_percentage",
        ),
    },

    compliance: {
        policy_violations: new Counter("policy_violations_total"),
        audit_findings: new Gauge("audit_findings_open"),
        training_completion: new Gauge("security_training_percentage"),
        patch_compliance: new Gauge("security_patch_percentage"),
    },
};

// Alert thresholds
const alertRules = [
    {
        metric: "auth_failed_total",
        condition: "rate(5m) > 10",
        severity: "warning",
    },
    {
        metric: "secret_rotation_compliance",
        condition: "< 0.95",
        severity: "critical",
    },
    {
        metric: "policy_violations_total",
        condition: "increase(1h) > 5",
        severity: "high",
    },
];
```

## Incident Response

### 1. Incident Preparation

#### Incident Response Plan

```javascript
// Incident response automation
class IncidentResponsePlan {
    constructor() {
        this.playbooks = {
            secret_exposure: {
                severity: "critical",
                steps: [
                    { action: "rotate_affected_secrets", timeout: 300 },
                    { action: "revoke_api_keys", timeout: 60 },
                    { action: "audit_access_logs", timeout: 600 },
                    { action: "notify_security_team", timeout: 30 },
                    { action: "create_incident_ticket", timeout: 60 },
                ],
            },

            unauthorized_access: {
                severity: "high",
                steps: [
                    { action: "disable_user_account", timeout: 30 },
                    { action: "terminate_sessions", timeout: 30 },
                    { action: "collect_forensics", timeout: 1800 },
                    { action: "reset_user_credentials", timeout: 300 },
                ],
            },

            data_breach: {
                severity: "critical",
                steps: [
                    { action: "isolate_affected_systems", timeout: 300 },
                    { action: "preserve_evidence", timeout: 600 },
                    { action: "notify_legal_team", timeout: 300 },
                    { action: "prepare_breach_notification", timeout: 3600 },
                ],
            },
        };
    }

    async executePlaybook(incidentType, context) {
        const playbook = this.playbooks[incidentType];
        if (!playbook) {
            throw new Error(`No playbook for incident type: ${incidentType}`);
        }

        const results = [];

        for (const step of playbook.steps) {
            try {
                const result = await this.executeStep(step, context);
                results.push({
                    step: step.action,
                    status: "completed",
                    result,
                });
            } catch (error) {
                results.push({
                    step: step.action,
                    status: "failed",
                    error: error.message,
                });

                // Determine if we should continue
                if (step.critical) {
                    break;
                }
            }
        }

        return results;
    }
}
```

### 2. Security Hardening Checklist

```yaml
# security-hardening-checklist.yaml
organization_level:
    authentication:
        - enforce_mfa: true
        - enforce_strong_passwords: true
        - implement_sso: recommended
        - regular_password_rotation: true

    access_control:
        - implement_rbac: true
        - principle_of_least_privilege: true
        - regular_access_reviews: quarterly
        - segregation_of_duties: true

    monitoring:
        - enable_audit_logging: true
        - implement_siem: recommended
        - anomaly_detection: true
        - regular_security_audits: true

application_level:
    secrets_management:
        - use_encryption: always
        - implement_rotation: true
        - secure_storage: true
        - access_logging: true

    secure_coding:
        - input_validation: true
        - output_encoding: true
        - parameterized_queries: true
        - secure_error_handling: true

    dependencies:
        - regular_updates: true
        - vulnerability_scanning: true
        - license_compliance: true
        - supply_chain_security: true

infrastructure_level:
    network_security:
        - network_segmentation: true
        - firewall_rules: restrictive
        - vpn_access: required
        - ddos_protection: true

    endpoint_security:
        - encryption_at_rest: true
        - antivirus: true
        - patch_management: automated
        - secure_configuration: true
```

## Resources

### Security Standards

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [CIS Controls](https://www.cisecurity.org/controls)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [ISO 27001 Controls](https://www.iso.org/isoiec-27001-information-security.html)

### DotEnv Security Resources

- [Security Whitepaper](https://dotenv.cloud/security/whitepaper)
- [Vulnerability Disclosure](https://dotenv.cloud/security/disclosure)
- [Security Updates](https://dotenv.cloud/security/updates)
- [Best Practices Guide](https://dotenv.cloud/docs/security/best-practices)

## Next Steps

- [Configure audit logging](/documentation/v1/security-compliance/audit-logging)
- [Set up incident response](/documentation/v1/security-compliance/incident-response)
- [Review penetration testing](/documentation/v1/security-compliance/penetration-testing)
- [Implement compliance controls](/documentation/v1/security-compliance/compliance)
