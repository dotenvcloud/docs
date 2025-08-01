---
title: Security & Compliance Overview
slug: overview
order: 1
tags: [security, compliance, overview, best-practices]
---

# Security & Compliance Overview

DotEnv is built with security at its core, providing enterprise-grade protection for your secrets while maintaining compliance with industry standards. This guide covers our security architecture, compliance certifications, and best practices.

## Security Architecture

### Defense in Depth

Our multi-layered security approach:

```
┌─────────────────────────────────────────┐
│         Network Security                 │
│  ┌───────────────────────────────────┐  │
│  │      Application Security         │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │   Encryption at Rest       │  │  │
│  │  │  ┌───────────────────┐     │  │  │
│  │  │  │ Access Controls   │     │  │  │
│  │  │  │ ┌─────────────┐  │     │  │  │
│  │  │  │ │   Audit     │  │     │  │  │
│  │  │  │ │   Logs      │  │     │  │  │
│  │  │  │ └─────────────┘  │     │  │  │
│  │  │  └───────────────────┘     │  │  │
│  │  └─────────────────────────────┘  │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

## Core Security Features

### 1. Encryption

#### At Rest

- **AES-256-GCM** encryption for all secrets
- Unique encryption keys per project
- Hardware Security Module (HSM) support
- Key rotation without downtime

#### In Transit

- **TLS 1.3** for all API communications
- Certificate pinning for CLI/SDKs
- Perfect Forward Secrecy (PFS)
- HSTS enforcement

#### Client-Side Encryption

- Optional end-to-end encryption
- Keys never touch our servers
- Zero-knowledge architecture option
- Browser-based encryption support

### 2. Access Control

#### Authentication

- Multi-factor authentication (MFA)
- SSO/SAML integration
- API key management
- Session management

#### Authorization

- Role-Based Access Control (RBAC)
- Fine-grained permissions
- Environment-specific access
- Team-based isolation

### 3. Audit & Monitoring

#### Comprehensive Logging

- All secret access logged
- API activity tracking
- Configuration changes
- Failed authentication attempts

#### Real-time Monitoring

- Anomaly detection
- Suspicious activity alerts
- Rate limiting
- Geographic restrictions

## Compliance Certifications

### Current Certifications

| Standard      | Status       | Last Audit | Next Audit |
| ------------- | ------------ | ---------- | ---------- |
| SOC 2 Type II | ✅ Certified | 2024-01    | 2025-01    |
| ISO 27001     | ✅ Certified | 2023-11    | 2024-11    |
| GDPR          | ✅ Compliant | Ongoing    | Ongoing    |
| HIPAA         | ✅ Compliant | 2024-02    | 2025-02    |
| PCI DSS       | ✅ Level 1   | 2024-03    | 2025-03    |

### Compliance Features

#### GDPR Compliance

- Data portability
- Right to erasure
- Privacy by design
- DPA available

#### HIPAA Compliance

- BAA available
- PHI encryption
- Access controls
- Audit trails

#### PCI DSS Compliance

- Network segmentation
- Vulnerability management
- Access control
- Regular testing

## Security Best Practices

### 1. Organization Level

```yaml
# security-policy.yaml
organization:
    mfa_required: true
    password_policy:
        min_length: 12
        require_uppercase: true
        require_numbers: true
        require_symbols: true
        rotation_days: 90

    session_policy:
        timeout_minutes: 15
        concurrent_sessions: 3
        ip_whitelist:
            - 10.0.0.0/8
            - 192.168.0.0/16

    api_policy:
        key_rotation_days: 30
        rate_limit: 1000/hour
        ip_restrictions: true
```

### 2. Project Level

```javascript
// Secure project configuration
const projectConfig = {
    encryption: {
        algorithm: "AES-256-GCM",
        keyManagement: "client-managed",
        keyRotation: {
            enabled: true,
            frequencyDays: 90,
        },
    },

    access: {
        environments: {
            development: ["developers", "qa"],
            staging: ["qa", "devops"],
            production: ["devops", "senior-engineers"],
        },

        permissions: {
            "secret:read": ["all-members"],
            "secret:write": ["developers", "devops"],
            "secret:decrypt": ["authorized-users"],
            "key:rotate": ["security-team"],
        },
    },

    audit: {
        retention_days: 365,
        export_enabled: true,
        real_time_alerts: true,
    },
};
```

### 3. Development Practices

```bash
#!/bin/bash
# secure-development.sh

# 1. Never commit secrets
echo ".env" >> .gitignore
echo "*.key" >> .gitignore
echo "*.pem" >> .gitignore

# 2. Use environment-specific configurations
dotenv pull development --output .env.development
dotenv pull staging --output .env.staging
# Never pull production locally

# 3. Implement secret scanning
npm install --save-dev @secretlint/secretlint-rule-preset-recommend
echo '{
  "rules": {
    "@secretlint/secretlint-rule-preset-recommend": true
  }
}' > .secretlintrc.json

# 4. Regular security updates
npm audit
npm audit fix
```

## Security Controls Matrix

| Control                | Implementation | Verification       | Frequency     |
| ---------------------- | -------------- | ------------------ | ------------- |
| Encryption at Rest     | AES-256-GCM    | Automated tests    | Continuous    |
| Encryption in Transit  | TLS 1.3        | SSL Labs scan      | Monthly       |
| Access Control         | RBAC + MFA     | Access reviews     | Quarterly     |
| Key Rotation           | Automated      | Rotation logs      | As configured |
| Vulnerability Scanning | Automated      | Security reports   | Weekly        |
| Penetration Testing    | Third-party    | Test reports       | Annually      |
| Security Training      | Required       | Completion records | Annually      |
| Incident Response      | 24/7 team      | Drill results      | Quarterly     |

## Threat Model

### Protected Against

1. **Data Breaches**

    - Encryption at rest
    - Access controls
    - Network isolation
    - Regular audits

2. **Insider Threats**

    - Principle of least privilege
    - Audit logging
    - Anomaly detection
    - Regular access reviews

3. **Supply Chain Attacks**

    - Dependency scanning
    - Code signing
    - Binary verification
    - Secure build pipeline

4. **Infrastructure Compromise**
    - HSM for key storage
    - Infrastructure as Code
    - Immutable deployments
    - Regular patching

## Security Incident Response

### Response Plan

1. **Detection** (0-15 minutes)

    - Automated alerting
    - 24/7 monitoring
    - Threat intelligence

2. **Containment** (15-60 minutes)

    - Isolate affected systems
    - Revoke compromised credentials
    - Block malicious IPs

3. **Investigation** (1-4 hours)

    - Forensic analysis
    - Impact assessment
    - Root cause analysis

4. **Remediation** (4-24 hours)

    - Patch vulnerabilities
    - Rotate affected keys
    - Update configurations

5. **Recovery** (24-48 hours)

    - Restore services
    - Verify integrity
    - Monitor for recurrence

6. **Lessons Learned** (1 week)
    - Post-mortem analysis
    - Update procedures
    - Implement improvements

## Shared Responsibility Model

### DotEnv Responsibilities

- Infrastructure security
- Platform availability
- Data encryption
- Physical security
- Network controls
- Platform updates

### Customer Responsibilities

- Access management
- Secret values
- Client-side encryption keys
- API key security
- User authentication
- Application security

### Shared Responsibilities

- Incident response
- Data classification
- Business continuity
- Compliance alignment
- Security monitoring
- Risk assessment

## Security Metrics

### Key Performance Indicators

```javascript
const securityKPIs = {
    // Availability
    uptime: "99.99%",
    mttr: "< 1 hour",

    // Encryption
    encryptionCoverage: "100%",
    keyRotationCompliance: "98%",

    // Access Control
    mfaAdoption: "95%",
    orphanedAccounts: "< 1%",

    // Incident Response
    detectionTime: "< 5 minutes",
    containmentTime: "< 30 minutes",

    // Compliance
    auditFindings: "0 critical",
    patchCompliance: "100% within SLA",
};
```

## Continuous Improvement

### Security Roadmap

1. **Q1 2024**

    - Post-quantum cryptography research
    - Enhanced anomaly detection
    - Expanded compliance certifications

2. **Q2 2024**

    - Hardware security key support
    - Advanced threat protection
    - Zero-trust architecture expansion

3. **Q3 2024**

    - AI-powered security monitoring
    - Automated compliance reporting
    - Enhanced forensics capabilities

4. **Q4 2024**
    - Quantum-resistant encryption
    - Decentralized key management
    - Advanced privacy features

## Resources

### Documentation

- [Encryption Details](/documentation/v1/security-compliance/encryption)
- [Access Control](/documentation/v1/security-compliance/access-control)
- [Audit Logging](/documentation/v1/security-compliance/audit-logging)
- [Compliance Certifications](/documentation/v1/security-compliance/compliance)

### Tools & Scripts

- Security assessment toolkit
- Compliance checker
- Audit report generator
- Incident response playbooks

### Support

- Security team: security@dotenv.cloud
- Compliance team: compliance@dotenv.cloud
- Bug bounty: security.dotenv.cloud/bounty
- Status page: status.dotenv.cloud

## Next Steps

1. [Review encryption options](/documentation/v1/security-compliance/encryption)
2. [Configure access controls](/documentation/v1/security-compliance/access-control)
3. [Enable audit logging](/documentation/v1/security-compliance/audit-logging)
4. [Implement best practices](/documentation/v1/security-compliance/security-best-practices)
