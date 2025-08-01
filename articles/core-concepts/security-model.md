---
title: Security Model
slug: security-model
order: 4
tags: [security, encryption, authentication, compliance]
---

# Security Model

DotEnv implements defense-in-depth security to protect your most sensitive data. This guide explains our comprehensive security architecture and practices.

## Security Principles

### 1. Zero Trust Architecture

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│   Client    │────▶│ Auth Layer   │────▶│ Permission   │
│ (Untrusted) │     │ (Verify All) │     │ Layer        │
└─────────────┘     └──────────────┘     └──────┬───────┘
                                                 │
                                                 ▼
                    ┌──────────────┐     ┌──────────────┐
                    │ Audit Layer  │────▶│ Encrypted    │
                    │ (Log All)    │     │ Storage      │
                    └──────────────┘     └──────────────┘
```

### 2. Principle of Least Privilege

Every access request is evaluated against:

- Who is requesting (Authentication)
- What they want to access (Resource)
- What they want to do (Action)
- Where they are (Context)
- When they are accessing (Time)

### 3. Defense in Depth

Multiple security layers:

1. Network security (TLS, firewalls)
2. Authentication (multi-factor)
3. Authorization (RBAC)
4. Encryption (at-rest and in-transit)
5. Audit logging (immutable)
6. Monitoring (anomaly detection)

## Encryption Architecture

### Encryption at Rest

All secrets are encrypted using AES-256-GCM:

```
Plaintext Secret: "DATABASE_URL=postgresql://..."
                          ↓
                    [AES-256-GCM]
                          ↓
Ciphertext: {
  "value": "gy7Kl9mN3pQ...",  // Encrypted data
  "nonce": "x4Zt8Km2...",     // 96-bit nonce
  "tag": "pL3nM9kJ...",       // Auth tag
  "version": 2,                // Encryption version
  "algorithm": "AES-256-GCM"   // Algorithm identifier
}
```

### Key Hierarchy

```
AWS KMS Customer Master Key (CMK)
    └── Organization Master Key (OMK)
            └── Project Encryption Key (PEK)
                    └── Secret Data Encryption Key (DEK)
```

**Key Rotation:**

- CMK: Managed by AWS, annual rotation
- OMK: Rotated every 90 days
- PEK: Rotated on demand or schedule
- DEK: Unique per secret, rotated with secret

### Encryption in Transit

All data in transit uses TLS 1.3:

```
Client ←──[TLS 1.3]──→ API Gateway ←──[TLS 1.3]──→ Backend Services
                                                          ↓
                                                    [TLS 1.3]
                                                          ↓
                                                    Database
```

**TLS Configuration:**

- Minimum version: TLS 1.3
- Cipher suites: AEAD only
- Certificate pinning available
- Perfect forward secrecy

## Authentication & Authorization

### Authentication Methods

#### 1. API Keys

```
Format: dotenv_api_[environment]_[random]
Example: dotenv_api_prod_x9Kl3mN7pQ2vR8s
```

Features:

- Scoped to organization/project
- Expiration support
- IP restriction capable
- Rate limiting per key

#### 2. OAuth 2.0

For web dashboard and integrations:

```
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGc...
```

Flow:

1. User initiates login
2. Redirect to auth provider
3. User authenticates + MFA
4. Receive authorization code
5. Exchange for access token
6. Access DotEnv resources

#### 3. Service Accounts

For automated systems:

```json
{
    "type": "service_account",
    "project_id": "my-project",
    "private_key_id": "key-id",
    "private_key": "-----BEGIN PRIVATE KEY-----\n...",
    "client_email": "ci-bot@my-project.dotenv.cloud",
    "auth_uri": "https://api.dotenv.cloud/oauth2/auth"
}
```

### Authorization Model

#### Role-Based Access Control (RBAC)

```yaml
roles:
    owner:
        permissions: ["*"]

    admin:
        permissions:
            - projects:*
            - secrets:*
            - members:*
            - settings:read

    developer:
        permissions:
            - projects:read
            - secrets:read
            - secrets:write
            - environments:development:*
            - environments:staging:*

    viewer:
        permissions:
            - projects:read
            - secrets:read
            - audit:read
```

#### Attribute-Based Access Control (ABAC)

```python
# Access decision based on attributes
def can_access(user, resource, action, context):
    return all([
        user.role in resource.allowed_roles,
        user.ip in context.allowed_ips,
        context.time in resource.access_hours,
        user.has_2fa or resource.security_level != "high"
    ])
```

### Multi-Factor Authentication

#### Supported Methods

1. **TOTP (Time-based One-Time Password)**

    - Google Authenticator
    - Authy
    - 1Password

2. **SMS**

    - Backup method
    - Not recommended for high-security

3. **WebAuthn/FIDO2**

    - Hardware keys (YubiKey, Titan)
    - Platform authenticators
    - Biometrics

4. **Recovery Codes**
    - One-time use
    - Encrypted storage
    - 10 codes generated

#### MFA Enforcement

```bash
# Organization-wide
dotenv orgs security mfa \
  --require all \
  --grace-period 7d

# Role-specific
dotenv security require-mfa \
  --roles admin,owner \
  --immediate

# Environment-specific
dotenv environments security production \
  --require-mfa \
  --no-exceptions
```

## Access Control

### Project-Level Security

```yaml
project: api-service
security:
    access_control:
        default: deny
        rules:
            - principal: role:developer
              environments: [development, staging]
              actions: [read, write]

            - principal: role:admin
              environments: [all]
              actions: [all]

            - principal: user:john@example.com
              environments: [production]
              actions: [read]
              conditions:
                  require_mfa: true
                  ip_range: "10.0.0.0/8"
                  time_range: "business_hours"
```

### Secret-Level Security

```yaml
secret: DATABASE_PASSWORD
security:
    classification: critical
    encryption:
        algorithm: AES-256-GCM
        key_rotation: 30d
    access:
        require_mfa: true
        require_approval: true
        approvers: [admin, security-team]
        audit_level: enhanced
    compliance:
        tags: [pci, pii, financial]
        retention: 7y
```

### Network Security

#### IP Allowlisting

```bash
# Organization level
dotenv orgs security ip-allowlist \
  --add "10.0.0.0/8" \
  --add "192.168.1.0/24" \
  --description "Office networks"

# Project level
dotenv projects security api-service \
  --ip-allowlist "10.0.1.0/24" \
  --enforcement strict
```

#### VPC Peering

```yaml
vpc_config:
    region: us-east-1
    vpc_id: vpc-12345
    peering:
        - vpc_id: vpc-customer
          account_id: "123456789"
          routes: ["10.0.0.0/16"]
```

## Audit & Compliance

### Audit Logging

Every action is logged with:

```json
{
    "event_id": "evt_9Kl3mN7pQ2vR8s",
    "timestamp": "2024-01-15T14:30:00Z",
    "actor": {
        "type": "user",
        "id": "usr_x4Zt8Km2",
        "email": "alice@example.com",
        "ip": "203.0.113.45",
        "user_agent": "DotEnv-CLI/1.0.0"
    },
    "action": {
        "type": "secret.update",
        "resource": "projects/api/secrets/DATABASE_URL",
        "environment": "production"
    },
    "result": {
        "status": "success",
        "duration_ms": 127
    },
    "security_context": {
        "mfa_verified": true,
        "session_age": 1820,
        "risk_score": 0.2
    }
}
```

### Compliance Features

#### SOC 2 Type II

- Annual audits
- Continuous monitoring
- Evidence collection
- Control documentation

#### GDPR Compliance

```bash
# Data export
dotenv privacy export \
  --user alice@example.com \
  --format json

# Data deletion
dotenv privacy delete \
  --user alice@example.com \
  --confirm \
  --reason "user_request"
```

#### HIPAA Compliance

- BAA available
- PHI encryption
- Access controls
- Audit trails

#### PCI DSS

- Network segmentation
- Encryption requirements
- Access logging
- Key management

### Data Residency

```bash
# Configure regional storage
dotenv orgs configure acme-eu \
  --region eu-west-1 \
  --data-residency eu \
  --compliance gdpr

# Verify data location
dotenv compliance verify \
  --organization acme-eu \
  --requirement data-residency
```

## Security Operations

### Threat Detection

#### Anomaly Detection

```python
# Behavioral analysis
anomalies = [
    "access_from_new_location",
    "unusual_access_time",
    "bulk_secret_access",
    "permission_escalation",
    "rapid_api_calls"
]

# Response actions
if anomaly_detected:
    actions = [
        "alert_security_team",
        "require_mfa_reauth",
        "temporarily_block",
        "initiate_review"
    ]
```

#### Security Monitoring

```yaml
monitoring:
    failed_auth_attempts:
        threshold: 5
        window: 5m
        action: block_ip

    secret_access_spike:
        threshold: 100
        window: 1m
        action: alert

    permission_changes:
        sensitivity: high
        notify: security@example.com
```

### Incident Response

#### Response Plan

1. **Detection** (< 5 minutes)

    - Automated alerts
    - Anomaly detection
    - User reports

2. **Containment** (< 15 minutes)

    - Isolate affected resources
    - Revoke compromised credentials
    - Enable emergency access controls

3. **Investigation** (< 1 hour)

    - Audit log analysis
    - Impact assessment
    - Root cause analysis

4. **Recovery** (< 4 hours)

    - Rotate affected secrets
    - Restore from backups
    - Verify system integrity

5. **Post-Incident** (< 24 hours)
    - Incident report
    - Lessons learned
    - Control improvements

#### Emergency Procedures

```bash
# Emergency access revocation
dotenv emergency revoke-all \
  --except emergency-team \
  --reason "potential_breach"

# Secret rotation
dotenv emergency rotate-all \
  --project affected-project \
  --notify-users

# Audit export
dotenv emergency export-audit \
  --last 48h \
  --include-all-details
```

## Security Best Practices

### For Administrators

1. **Enable MFA Everywhere**

```bash
dotenv security enforce-mfa \
  --organization-wide \
  --no-exceptions
```

2. **Regular Access Reviews**

```bash
dotenv security review access \
  --frequency quarterly \
  --auto-revoke-unused
```

3. **Implement Least Privilege**

```bash
dotenv security analyze permissions \
  --suggest-reductions \
  --apply-recommendations
```

4. **Monitor Continuously**

```bash
dotenv security monitor \
  --real-time \
  --alert-channel security
```

### For Developers

1. **Never Log Secrets**

```javascript
// ❌ Bad
console.log(process.env);

// ✅ Good
console.log("App started successfully");
```

2. **Use Short-Lived Tokens**

```bash
# Create temporary access
dotenv auth token \
  --expires 1h \
  --scope "secrets:read"
```

3. **Rotate Regularly**

```bash
# Set up rotation
dotenv secrets rotate DATABASE_PASSWORD \
  --schedule "0 0 1 * *" \
  --notify-before 7d
```

4. **Validate Inputs**

```javascript
// Validate before using
const dbUrl = process.env.DATABASE_URL;
if (!isValidDatabaseUrl(dbUrl)) {
    throw new Error("Invalid database configuration");
}
```

### For Security Teams

1. **Define Security Policies**

```yaml
# security-policy.yml
policies:
    password:
        min_length: 12
        require_uppercase: true
        require_special: true
        max_age: 90d

    session:
        timeout: 8h
        max_concurrent: 3

    api_keys:
        max_age: 180d
        require_rotation: true
```

2. **Implement Defense in Depth**

```bash
# Layer security controls
dotenv security layers \
  --enable-waf \
  --enable-ddos-protection \
  --enable-intrusion-detection \
  --enable-vulnerability-scanning
```

3. **Regular Security Assessments**

```bash
# Automated scanning
dotenv security scan \
  --type full \
  --include-dependencies \
  --generate-report
```

## Security Certifications

### Current Certifications

- **SOC 2 Type II** - Annual
- **ISO 27001** - Information Security
- **ISO 27017** - Cloud Security
- **ISO 27018** - Privacy
- **CSA STAR** - Cloud Security

### Compliance Reports

Available upon request:

- Penetration test results
- Vulnerability assessments
- Compliance attestations
- Security architecture reviews

## Contact Security Team

- 🚨 Security incidents: security@dotenv.cloud
- 🐛 Bug bounty: bounty.dotenv.cloud
- 📧 Security questions: security@dotenv.cloud
- 🔐 GPG Key: security.dotenv.cloud/pgp

## Next Steps

- [Secret Management](./secret-management) - Best practices for secrets
- [Access Control](./access-control) - Detailed permission model
- [Compliance](/documentation/v1/security/compliance) - Regulatory compliance
- [Security Checklist](/documentation/v1/security/checklist) - Security audit guide
