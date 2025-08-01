---
title: Compliance
slug: compliance
order: 5
tags: [compliance, regulations, certifications, standards]
---

# Compliance

DotEnv maintains comprehensive compliance with industry standards and regulations to ensure your data is protected according to the highest security standards. This guide covers our compliance certifications, features, and implementation guidance.

## Compliance Overview

### Current Certifications

| Certification       | Status         | Scope                                   | Last Audit | Next Audit |
| ------------------- | -------------- | --------------------------------------- | ---------- | ---------- |
| **SOC 2 Type II**   | ✅ Active      | Security, Availability, Confidentiality | 2024-01-15 | 2025-01-15 |
| **ISO 27001:2022**  | ✅ Active      | Information Security Management         | 2023-11-20 | 2024-11-20 |
| **GDPR**            | ✅ Compliant   | Data Protection & Privacy               | Ongoing    | Ongoing    |
| **HIPAA**           | ✅ Compliant   | Healthcare Data Protection              | 2024-02-28 | 2025-02-28 |
| **PCI DSS Level 1** | ✅ Certified   | Payment Card Data                       | 2024-03-10 | 2025-03-10 |
| **FedRAMP**         | 🔄 In Progress | Government Cloud Security               | -          | 2024-Q4    |
| **CCPA**            | ✅ Compliant   | California Privacy Rights               | Ongoing    | Ongoing    |

### Compliance Framework

```
┌─────────────────────────────────────────┐
│         Governance Layer                │
│  • Policies & Procedures               │
│  • Risk Management                     │
│  • Compliance Committee                │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│         Technical Controls              │
│  • Encryption                          │
│  • Access Control                      │
│  • Audit Logging                       │
│  • Data Protection                     │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│      Administrative Controls            │
│  • Training & Awareness                │
│  • Incident Response                   │
│  • Vendor Management                   │
│  • Change Management                   │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│      Physical Controls                  │
│  • Data Center Security                │
│  • Environmental Controls              │
│  • Media Disposal                      │
└─────────────────────────────────────────┘
```

## SOC 2 Type II Compliance

### Trust Service Criteria

#### 1. Security

```javascript
// Security controls implementation
const securityControls = {
    // CC6.1: Logical and Physical Access Controls
    accessControl: {
        authentication: {
            mfa_required: true,
            password_policy: {
                min_length: 12,
                complexity: "high",
                rotation: 90,
            },
            session_timeout: 900, // 15 minutes
        },

        authorization: {
            model: "RBAC",
            principle: "least_privilege",
            review_frequency: "quarterly",
        },

        physical: {
            data_centers: "SOC 2 certified",
            access_logs: true,
            biometric_controls: true,
        },
    },

    // CC6.6: Encryption
    encryption: {
        at_rest: {
            algorithm: "AES-256-GCM",
            key_management: "HSM",
            key_rotation: 90,
        },

        in_transit: {
            protocol: "TLS 1.3",
            cipher_suites: "AEAD only",
            certificate_pinning: true,
        },
    },

    // CC7.1: System Monitoring
    monitoring: {
        security_events: {
            collection: "real-time",
            analysis: "automated",
            retention: 365,
        },

        vulnerability_scanning: {
            frequency: "weekly",
            scope: "full_stack",
            remediation_sla: {
                critical: 24,
                high: 72,
                medium: 168,
                low: 720,
            },
        },
    },
};
```

#### 2. Availability

```javascript
// Availability controls
const availabilityControls = {
    // A1.2: System Availability
    sla: {
        uptime_target: 99.99,
        measurement_period: "monthly",
        exclusions: ["scheduled_maintenance"],
    },

    // Redundancy
    infrastructure: {
        regions: ["us-east-1", "us-west-2", "eu-west-1"],
        availability_zones: "multi",
        failover: "automatic",
        rto: 60, // Recovery Time Objective in seconds
        rpo: 60, // Recovery Point Objective in seconds
    },

    // Backup and Recovery
    backup: {
        frequency: "continuous",
        retention: {
            daily: 30,
            weekly: 52,
            monthly: 84,
        },
        testing: "quarterly",
        encryption: true,
    },
};
```

#### 3. Confidentiality

```javascript
// Confidentiality controls
const confidentialityControls = {
    // C1.1: Confidential Information Protection
    data_classification: {
        levels: ["public", "internal", "confidential", "restricted"],
        handling: {
            restricted: {
                encryption: "required",
                access: "need_to_know",
                transmission: "secure_channels_only",
            },
        },
    },

    // C1.2: Disposal of Confidential Information
    data_disposal: {
        electronic: {
            method: "cryptographic_erasure",
            verification: "required",
            certificate: "generated",
        },

        physical: {
            method: "cross_cut_shredding",
            particle_size: "1mm x 5mm",
            witness: "required",
        },
    },
};
```

### SOC 2 Evidence Collection

```javascript
// Automated evidence collection
class SOC2EvidenceCollector {
    async collectEvidence(controlId, period) {
        const evidence = {
            control_id: controlId,
            period: period,
            collected_at: new Date(),
            artifacts: [],
        };

        switch (controlId) {
            case "CC6.1": // Access Control
                evidence.artifacts =
                    await this.collectAccessControlEvidence(period);
                break;

            case "CC6.6": // Encryption
                evidence.artifacts =
                    await this.collectEncryptionEvidence(period);
                break;

            case "CC7.1": // Monitoring
                evidence.artifacts =
                    await this.collectMonitoringEvidence(period);
                break;
        }

        return evidence;
    }

    async collectAccessControlEvidence(period) {
        return [
            {
                type: "user_access_reviews",
                data: await this.getUserAccessReviews(period),
            },
            {
                type: "authentication_logs",
                data: await this.getAuthenticationLogs(period),
            },
            {
                type: "permission_changes",
                data: await this.getPermissionChanges(period),
            },
        ];
    }
}
```

## ISO 27001:2022 Compliance

### Information Security Management System (ISMS)

#### 1. Context of the Organization

```yaml
# ISMS Scope Definition
scope:
    services:
        - Secret Management Platform
        - API Services
        - Web Application
        - CLI Tools

    locations:
        - Primary: US-East (Virginia)
        - Secondary: US-West (Oregon)
        - Tertiary: EU-West (Ireland)

    exclusions:
        - Corporate IT (separate ISMS)
        - Marketing Systems
```

#### 2. Risk Assessment

```javascript
// Risk assessment framework
class ISO27001RiskAssessment {
    assessRisk(asset, threat, vulnerability) {
        const likelihood = this.calculateLikelihood(threat, vulnerability);
        const impact = this.calculateImpact(asset, threat);

        const riskLevel = likelihood * impact;

        return {
            asset: asset.name,
            threat: threat.name,
            vulnerability: vulnerability.name,
            likelihood: likelihood,
            impact: impact,
            risk_level: riskLevel,
            treatment: this.determineRiskTreatment(riskLevel),
        };
    }

    calculateLikelihood(threat, vulnerability) {
        // Based on threat probability and vulnerability severity
        const matrix = {
            high: { high: 5, medium: 4, low: 3 },
            medium: { high: 4, medium: 3, low: 2 },
            low: { high: 3, medium: 2, low: 1 },
        };

        return matrix[threat.probability][vulnerability.severity];
    }

    determineRiskTreatment(riskLevel) {
        if (riskLevel >= 20) return "avoid";
        if (riskLevel >= 12) return "mitigate";
        if (riskLevel >= 6) return "transfer";
        return "accept";
    }
}
```

#### 3. Control Implementation

```javascript
// ISO 27001 Annex A Controls
const iso27001Controls = {
    // A.5: Information Security Policies
    "A.5.1": {
        control: "Information security policy",
        implementation: {
            policy_document: "IS-POL-001",
            review_frequency: "annual",
            approval: "CISO",
            distribution: "all_employees",
        },
    },

    // A.9: Access Control
    "A.9.1": {
        control: "Access control policy",
        implementation: {
            policy: "AC-POL-001",
            enforcement: "automated",
            technologies: ["RBAC", "ABAC", "MFA"],
        },
    },

    // A.12: Cryptography
    "A.12.1": {
        control: "Cryptographic controls",
        implementation: {
            algorithms: ["AES-256-GCM", "RSA-4096", "SHA-256"],
            key_management: "HSM",
            standards: ["FIPS 140-2", "NIST SP 800-57"],
        },
    },
};
```

## GDPR Compliance

### Data Protection Principles

#### 1. Lawfulness, Fairness, and Transparency

```javascript
// GDPR consent management
class GDPRConsentManager {
    async recordConsent(userId, purpose, details) {
        const consent = {
            user_id: userId,
            purpose: purpose,
            given_at: new Date(),
            details: {
                what_data: details.dataTypes,
                why_needed: details.purpose,
                how_long: details.retention,
                who_access: details.processors,
                withdrawal_method: details.withdrawalProcess,
            },
            version: this.getCurrentConsentVersion(),
            ip_address: details.ipAddress,
            user_agent: details.userAgent,
        };

        await db.consents.create(consent);

        return consent;
    }

    async withdrawConsent(userId, purpose) {
        const withdrawal = {
            user_id: userId,
            purpose: purpose,
            withdrawn_at: new Date(),
            consequences_acknowledged: true,
        };

        await db.consentWithdrawals.create(withdrawal);

        // Trigger data processing cessation
        await this.stopProcessing(userId, purpose);

        return withdrawal;
    }
}
```

#### 2. Data Subject Rights

```javascript
// GDPR rights implementation
class GDPRRightsHandler {
    // Right to Access (Article 15)
    async handleAccessRequest(userId) {
        const data = {
            personal_data: await this.collectPersonalData(userId),
            processing_purposes: await this.getProcessingPurposes(userId),
            recipients: await this.getDataRecipients(userId),
            retention_periods: await this.getRetentionPeriods(userId),
            rights: this.getDataSubjectRights(),
            source: await this.getDataSource(userId),
        };

        return this.formatDataExport(data);
    }

    // Right to Rectification (Article 16)
    async handleRectificationRequest(userId, corrections) {
        const results = [];

        for (const correction of corrections) {
            const result = await this.rectifyData(
                userId,
                correction.field,
                correction.oldValue,
                correction.newValue,
            );

            results.push(result);
        }

        await this.notifyDataRecipients(userId, corrections);

        return results;
    }

    // Right to Erasure (Article 17)
    async handleErasureRequest(userId, reason) {
        // Check if erasure is allowed
        const assessment = await this.assessErasureRequest(userId, reason);

        if (!assessment.allowed) {
            return {
                status: "denied",
                reason: assessment.denial_reason,
                alternatives: assessment.alternatives,
            };
        }

        // Perform erasure
        const erasureResult = await this.erasePersonalData(userId);

        // Notify third parties
        await this.notifyThirdParties(userId, "erasure");

        return erasureResult;
    }

    // Right to Data Portability (Article 20)
    async handlePortabilityRequest(userId, format) {
        const data = await this.collectPersonalData(userId);

        switch (format) {
            case "json":
                return {
                    format: "application/json",
                    data: JSON.stringify(data, null, 2),
                };

            case "csv":
                return {
                    format: "text/csv",
                    data: this.convertToCSV(data),
                };

            case "xml":
                return {
                    format: "application/xml",
                    data: this.convertToXML(data),
                };
        }
    }
}
```

#### 3. Privacy by Design

```javascript
// Privacy by design implementation
const privacyByDesign = {
    // Data minimization
    dataCollection: {
        principle: "collect_minimum_necessary",
        implementation: {
            required_fields: ["email"],
            optional_fields: ["name", "company"],
            prohibited_fields: ["ssn", "health_data"],
        },
    },

    // Purpose limitation
    purposeLimitation: {
        defined_purposes: [
            "service_provision",
            "billing",
            "security",
            "legal_compliance",
        ],
        purpose_binding: true,
        cross_purpose_processing: false,
    },

    // Privacy defaults
    privacyDefaults: {
        data_sharing: "opt_in",
        marketing: "opt_in",
        analytics: "anonymous_only",
        retention: "minimum_required",
    },
};
```

## HIPAA Compliance

### Administrative Safeguards

```javascript
// HIPAA administrative controls
const hipaaAdministrative = {
    // Security Officer
    securityOfficer: {
        name: "Chief Security Officer",
        responsibilities: [
            "security_program_oversight",
            "risk_assessments",
            "incident_response",
            "workforce_training",
        ],
    },

    // Workforce Training
    training: {
        onboarding: {
            required: true,
            topics: [
                "phi_handling",
                "security_awareness",
                "incident_reporting",
            ],
            frequency: "upon_hire",
        },

        ongoing: {
            required: true,
            frequency: "annual",
            topics: ["updates", "threats", "best_practices"],
        },
    },

    // Access Management
    accessManagement: {
        authorization: {
            process: "role_based",
            approval: "manager_and_security_officer",
            review: "quarterly",
        },

        workforce_clearance: {
            background_check: true,
            reference_check: true,
            confidentiality_agreement: true,
        },
    },
};
```

### Technical Safeguards

```javascript
// HIPAA technical controls
class HIPAATechnicalSafeguards {
    // Access Control
    async enforceAccessControl(user, resource) {
        // Unique user identification
        if (!user.uniqueId) {
            throw new Error("User must have unique identifier");
        }

        // Automatic logoff
        if (Date.now() - user.lastActivity > 900000) {
            // 15 minutes
            await this.logoffUser(user);
            throw new Error("Session expired");
        }

        // Encryption and decryption
        if (resource.containsPHI && !user.hasPermission("phi_access")) {
            throw new Error("Not authorized for PHI access");
        }
    }

    // Audit Controls
    async logPHIAccess(user, action, resource) {
        const auditLog = {
            timestamp: new Date(),
            user_id: user.uniqueId,
            action: action,
            resource_type: resource.type,
            resource_id: resource.id,
            phi_accessed: true,
            details: {
                ip_address: user.ipAddress,
                session_id: user.sessionId,
                access_reason: user.accessReason,
            },
        };

        await this.auditLogger.logPHIAccess(auditLog);
    }

    // Transmission Security
    encryptPHITransmission(data) {
        return {
            encrypted: true,
            algorithm: "AES-256-GCM",
            integrity_check: this.generateHMAC(data),
            transmission_protocol: "TLS 1.3",
        };
    }
}
```

### Physical Safeguards

```javascript
// HIPAA physical controls
const hipaaPhysical = {
    // Facility Access Controls
    facilityAccess: {
        data_centers: {
            access_control: "badge_and_biometric",
            visitor_escort: "required",
            access_logs: "maintained",
            video_surveillance: "24/7",
        },
    },

    // Workstation Security
    workstationSecurity: {
        physical_locks: "required",
        positioning: "screen_not_visible_to_public",
        automatic_lock: "15_minutes",
        clean_desk_policy: true,
    },

    // Device Controls
    deviceControls: {
        disposal: {
            method: "dod_5220.22-m",
            verification: "certificate_required",
            tracking: "asset_management_system",
        },

        media_reuse: {
            sanitization: "required",
            method: "cryptographic_erasure",
            verification: "automated",
        },
    },
};
```

## PCI DSS Compliance

### Network Security

```javascript
// PCI DSS network segmentation
class PCINetworkSecurity {
    constructor() {
        this.segments = {
            cde: {
                name: "Cardholder Data Environment",
                vlan: 100,
                subnet: "10.0.100.0/24",
                access: "restricted",
            },

            dmz: {
                name: "Demilitarized Zone",
                vlan: 200,
                subnet: "10.0.200.0/24",
                access: "controlled",
            },

            corporate: {
                name: "Corporate Network",
                vlan: 300,
                subnet: "10.0.300.0/24",
                access: "standard",
            },
        };
    }

    validateNetworkAccess(source, destination) {
        // No direct access from corporate to CDE
        if (source.segment === "corporate" && destination.segment === "cde") {
            return {
                allowed: false,
                reason: "Direct access to CDE not permitted",
            };
        }

        // All CDE access must be authenticated and encrypted
        if (destination.segment === "cde") {
            return {
                allowed: true,
                requirements: ["mfa", "encryption", "audit_logging"],
            };
        }

        return { allowed: true };
    }
}
```

### Data Protection

```javascript
// PCI DSS data protection
class PCIDataProtection {
    // Primary Account Number (PAN) handling
    maskPAN(pan) {
        if (pan.length <= 6) return "*".repeat(pan.length);

        const firstSix = pan.substring(0, 6);
        const lastFour = pan.substring(pan.length - 4);
        const masked = "*".repeat(pan.length - 10);

        return `${firstSix}${masked}${lastFour}`;
    }

    // Tokenization
    async tokenizePAN(pan) {
        const token = await this.tokenVault.store(pan);

        return {
            token: token,
            first_six: pan.substring(0, 6),
            last_four: pan.substring(pan.length - 4),
            expiry: null, // Tokens don't expire
        };
    }

    // Cryptographic key management
    async manageKeys() {
        const keyManagement = {
            key_generation: {
                algorithm: "AES-256",
                source: "HSM",
                dual_control: true,
            },

            key_storage: {
                location: "HSM",
                encryption: "KEK",
                access_control: "role_based",
            },

            key_rotation: {
                frequency: "annual",
                automatic: true,
                backwards_compatible: true,
            },
        };

        return keyManagement;
    }
}
```

## Compliance Automation

### Continuous Compliance Monitoring

```javascript
// Automated compliance monitoring
class ComplianceMonitor {
    async runComplianceChecks() {
        const results = {
            timestamp: new Date(),
            checks: [],
        };

        // SOC 2 checks
        results.checks.push(await this.checkSOC2Compliance());

        // ISO 27001 checks
        results.checks.push(await this.checkISO27001Compliance());

        // GDPR checks
        results.checks.push(await this.checkGDPRCompliance());

        // HIPAA checks
        if (this.scopeIncludesHIPAA()) {
            results.checks.push(await this.checkHIPAACompliance());
        }

        // PCI DSS checks
        if (this.scopeIncludesPCI()) {
            results.checks.push(await this.checkPCIDSSCompliance());
        }

        // Generate report
        await this.generateComplianceReport(results);

        // Alert on failures
        await this.alertOnFailures(results);

        return results;
    }

    async checkSOC2Compliance() {
        const checks = [
            {
                control: "CC6.1",
                description: "Logical and physical access controls",
                check: async () => await this.verifyAccessControls(),
                remediation: "Review and update access control policies",
            },
            {
                control: "CC7.2",
                description: "System monitoring",
                check: async () => await this.verifyMonitoring(),
                remediation: "Enable missing monitoring points",
            },
        ];

        const results = [];
        for (const check of checks) {
            try {
                const passed = await check.check();
                results.push({
                    control: check.control,
                    status: passed ? "pass" : "fail",
                    details: check.description,
                    remediation: passed ? null : check.remediation,
                });
            } catch (error) {
                results.push({
                    control: check.control,
                    status: "error",
                    details: error.message,
                });
            }
        }

        return {
            framework: "SOC 2",
            results: results,
        };
    }
}
```

### Compliance Reporting

```javascript
// Compliance report generator
class ComplianceReporter {
    async generateExecutiveReport(period) {
        const report = {
            period: period,
            executive_summary: {
                overall_compliance: await this.calculateOverallCompliance(),
                key_risks: await this.identifyKeyRisks(),
                remediation_progress: await this.getRemediationProgress(),
            },

            framework_status: {
                soc2: await this.getSOC2Status(),
                iso27001: await this.getISO27001Status(),
                gdpr: await this.getGDPRStatus(),
                hipaa: await this.getHIPAAStatus(),
                pci_dss: await this.getPCIDSSStatus(),
            },

            metrics: {
                policy_violations: await this.getPolicyViolations(period),
                security_incidents: await this.getSecurityIncidents(period),
                access_reviews_completed:
                    await this.getAccessReviewMetrics(period),
                training_completion: await this.getTrainingMetrics(period),
            },

            upcoming: {
                audits: await this.getUpcomingAudits(),
                certifications: await this.getExpiringCertifications(),
                required_actions: await this.getRequiredActions(),
            },
        };

        return this.formatExecutiveReport(report);
    }
}
```

## Best Practices

### 1. Continuous Compliance

```javascript
// ✅ Good: Automated continuous monitoring
schedule.every("1 hour").do(async () => {
    await complianceMonitor.runComplianceChecks();
});

// ❌ Bad: Manual annual checks only
// Check compliance once a year before audit
```

### 2. Evidence Collection

```javascript
// ✅ Good: Automated evidence collection
async function collectEvidence() {
    return {
        access_logs: await getAccessLogs(),
        change_logs: await getChangeLogs(),
        training_records: await getTrainingRecords(),
        screenshots: await captureSystemState(),
        timestamp: new Date(),
        hash: generateIntegrityHash(),
    };
}

// ❌ Bad: Manual evidence gathering
// Scramble to find evidence during audit
```

### 3. Documentation

```javascript
// ✅ Good: Living documentation
const complianceDocs = {
    policies: {
        version: "2.0",
        last_updated: "2024-01-15",
        next_review: "2024-07-15",
        auto_reminder: true,
    },

    procedures: {
        format: "structured",
        includes_examples: true,
        tested: true,
    },
};

// ❌ Bad: Outdated documentation
// Policies from 3 years ago
```

## Resources

### Standards and Frameworks

- [SOC 2 Trust Services Criteria](https://www.aicpa.org/interestareas/frc/assuranceadvisoryservices/trustservices)
- [ISO 27001:2022 Standard](https://www.iso.org/standard/82875.html)
- [GDPR Official Text](https://gdpr-info.eu/)
- [HIPAA Security Rule](https://www.hhs.gov/hipaa/for-professionals/security/index.html)
- [PCI DSS v4.0](https://www.pcisecuritystandards.org/)

### DotEnv Compliance Resources

- [Compliance Portal](https://compliance.dotenv.cloud)
- [Audit Reports](https://dotenv.cloud/compliance/reports)
- [Security Whitepaper](https://dotenv.cloud/security/whitepaper)
- [DPA Template](https://dotenv.cloud/legal/dpa)

## Next Steps

- [Implement security best practices](/documentation/v1/security-compliance/security-best-practices)
- [Set up audit logging](/documentation/v1/security-compliance/audit-logging)
- [Configure incident response](/documentation/v1/security-compliance/incident-response)
- [Review penetration testing](/documentation/v1/security-compliance/penetration-testing)
