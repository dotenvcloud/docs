---
title: Penetration Testing
slug: penetration-testing
order: 8
tags:
    [security, penetration-testing, vulnerability-assessment, security-testing]
---

# Penetration Testing

This guide covers DotEnv's penetration testing program, methodologies, and how to conduct security assessments of the platform. Learn about our testing approach, scope, and how to report findings.

## Penetration Testing Overview

### Testing Program

```yaml
# Penetration Testing Program Structure
program:
    frequency:
        external_testing: quarterly
        internal_testing: bi-annual
        continuous_testing: ongoing
        red_team_exercises: annual

    providers:
        external:
            - name: "Security Firm A"
              specialization: "Web Application Security"
              certifications: ["OSCP", "GWAPT", "OSWE"]

            - name: "Security Firm B"
              specialization: "Cloud Infrastructure"
              certifications: ["AWS Security", "Azure Security"]

        internal:
            team_size: 4
            certifications: ["CEH", "OSCP", "SANS"]

    scope:
        included:
            - Production web applications
            - API endpoints
            - Mobile applications
            - Cloud infrastructure
            - Internal networks

        excluded:
            - Third-party services
            - Office networks
            - Employee devices
```

### Testing Methodology

```
┌─────────────────────────────────────────┐
│      1. Planning & Reconnaissance       │
│  • Scope definition                     │
│  • Information gathering                │
│  • Threat modeling                      │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│         2. Scanning & Enumeration       │
│  • Port scanning                        │
│  • Service enumeration                  │
│  • Vulnerability scanning               │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│         3. Exploitation                 │
│  • Vulnerability validation             │
│  • Controlled exploitation              │
│  • Privilege escalation                 │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│      4. Post-Exploitation               │
│  • Lateral movement                     │
│  • Data access assessment               │
│  • Persistence testing                  │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│         5. Reporting                    │
│  • Technical documentation              │
│  • Risk assessment                      │
│  • Remediation guidance                 │
└─────────────────────────────────────────┘
```

## Testing Scope & Rules

### Scope Definition

```javascript
// Penetration testing scope
const testingScope = {
    web_applications: {
        included: [
            "https://app.dotenv.cloud/*",
            "https://api.dotenv.cloud/v1/*",
            "https://dashboard.dotenv.cloud/*",
        ],

        excluded: [
            "https://blog.dotenv.cloud",
            "https://status.dotenv.cloud",
            "https://docs.dotenv.cloud",
        ],

        testing_accounts: {
            organization: "pentest-org",
            users: [
                "pentest-user-1@dotenv.cloud",
                "pentest-user-2@dotenv.cloud",
                "pentest-admin@dotenv.cloud",
            ],
        },
    },

    infrastructure: {
        networks: [
            "10.0.0.0/16", // Production VPC
            "10.1.0.0/16", // Staging VPC
        ],

        cloud_accounts: {
            aws: {
                account_id: "123456789012",
                regions: ["us-east-1", "eu-west-1"],
                services: ["EC2", "RDS", "S3", "Lambda"],
            },
        },
    },

    mobile_applications: {
        ios: {
            bundle_id: "com.dotenv.mobile",
            test_devices: ["iPhone 13", "iPad Pro"],
        },

        android: {
            package_name: "com.dotenv.android",
            test_devices: ["Pixel 6", "Samsung Galaxy S22"],
        },
    },
};
```

### Rules of Engagement

```javascript
// Testing rules and limitations
const rulesOfEngagement = {
    timing: {
        testing_window: {
            start: "09:00 UTC",
            end: "17:00 UTC",
            days: ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"],
        },

        rate_limiting: {
            requests_per_second: 10,
            concurrent_connections: 50,
            total_daily_requests: 100000,
        },
    },

    prohibited_activities: [
        "Denial of Service (DoS) attacks",
        "Physical security testing",
        "Social engineering of employees",
        "Testing on production data",
        "Automated vulnerability scanning during business hours",
        "Accessing or modifying real customer data",
        "Testing third-party integrations",
    ],

    allowed_techniques: {
        web_testing: [
            "SQL injection",
            "Cross-site scripting (XSS)",
            "Authentication bypass",
            "Authorization testing",
            "Session management testing",
            "Input validation testing",
            "Business logic testing",
        ],

        infrastructure_testing: [
            "Port scanning",
            "Service enumeration",
            "Configuration review",
            "Patch level assessment",
            "Network segmentation testing",
            "Cryptography assessment",
        ],

        api_testing: [
            "Authentication testing",
            "Rate limit testing",
            "Input fuzzing",
            "JWT manipulation",
            "API versioning tests",
            "GraphQL specific tests",
        ],
    },

    data_handling: {
        screenshots: "Redact sensitive data",
        logs: "Sanitize before sharing",
        poc_code: "Minimize data exposure",
        reports: "Encrypt with PGP",
    },
};
```

## Testing Methodologies

### Web Application Testing

```javascript
// Web application security testing
class WebAppPentest {
    async performTesting() {
        const results = {
            vulnerabilities: [],
            informational: [],
            timestamp: new Date(),
        };

        // Authentication testing
        results.vulnerabilities.push(...(await this.testAuthentication()));

        // Authorization testing
        results.vulnerabilities.push(...(await this.testAuthorization()));

        // Input validation
        results.vulnerabilities.push(...(await this.testInputValidation()));

        // Session management
        results.vulnerabilities.push(...(await this.testSessionManagement()));

        // Business logic
        results.vulnerabilities.push(...(await this.testBusinessLogic()));

        return results;
    }

    async testAuthentication() {
        const findings = [];

        // Test password policies
        const passwordTests = [
            { password: "123456", expected: "rejected" },
            { password: "password", expected: "rejected" },
            { password: "P@ssw0rd", expected: "rejected" },
            { password: "SecureP@ssw0rd123!", expected: "accepted" },
        ];

        for (const test of passwordTests) {
            const result = await this.tryPassword(test.password);
            if (result !== test.expected) {
                findings.push({
                    type: "Weak Password Policy",
                    severity: "Medium",
                    details: `Password "${test.password}" was ${result}`,
                });
            }
        }

        // Test account lockout
        const lockoutResult = await this.testAccountLockout();
        if (!lockoutResult.implemented) {
            findings.push({
                type: "No Account Lockout",
                severity: "High",
                details:
                    "Account lockout not implemented after multiple failed attempts",
            });
        }

        // Test MFA bypass
        const mfaBypass = await this.testMFABypass();
        if (mfaBypass.vulnerable) {
            findings.push({
                type: "MFA Bypass",
                severity: "Critical",
                details: mfaBypass.details,
            });
        }

        return findings;
    }

    async testInputValidation() {
        const findings = [];
        const payloads = {
            sql_injection: [
                "' OR '1'='1",
                "1; DROP TABLE users--",
                "' UNION SELECT * FROM users--",
                "admin'--",
            ],

            xss: [
                "<script>alert('XSS')</script>",
                "<img src=x onerror=alert('XSS')>",
                "javascript:alert('XSS')",
                "<svg onload=alert('XSS')>",
            ],

            command_injection: [
                "; ls -la",
                "| whoami",
                "`id`",
                "$(cat /etc/passwd)",
            ],

            xxe: [
                '<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]><foo>&xxe;</foo>',
                '<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://internal.server/">]><foo>&xxe;</foo>',
            ],
        };

        // Test each payload type
        for (const [vulnType, testPayloads] of Object.entries(payloads)) {
            for (const payload of testPayloads) {
                const result = await this.testPayload(payload, vulnType);
                if (result.vulnerable) {
                    findings.push({
                        type: vulnType.toUpperCase(),
                        severity: this.getSeverity(vulnType),
                        endpoint: result.endpoint,
                        parameter: result.parameter,
                        payload: payload,
                        response: result.response,
                    });
                }
            }
        }

        return findings;
    }
}
```

### API Security Testing

```javascript
// API penetration testing
class APIPentest {
    async testAPI() {
        const findings = [];

        // API enumeration
        const endpoints = await this.enumerateEndpoints();

        // Test each endpoint
        for (const endpoint of endpoints) {
            findings.push(...(await this.testEndpoint(endpoint)));
        }

        // Test rate limiting
        findings.push(...(await this.testRateLimiting()));

        // Test API versioning
        findings.push(...(await this.testVersioning()));

        return findings;
    }

    async testEndpoint(endpoint) {
        const findings = [];

        // Authentication tests
        const authTests = [
            { test: "missing_auth", headers: {} },
            {
                test: "invalid_token",
                headers: { Authorization: "Bearer invalid" },
            },
            {
                test: "expired_token",
                headers: { Authorization: "Bearer " + this.expiredToken },
            },
            {
                test: "wrong_scope",
                headers: { Authorization: "Bearer " + this.limitedToken },
            },
        ];

        for (const test of authTests) {
            const response = await this.makeRequest(endpoint, test.headers);
            if (response.status === 200) {
                findings.push({
                    type: "Authentication Bypass",
                    severity: "Critical",
                    endpoint: endpoint,
                    test: test.test,
                    details: `Endpoint accessible with ${test.test}`,
                });
            }
        }

        // IDOR testing
        const idorResult = await this.testIDOR(endpoint);
        if (idorResult.vulnerable) {
            findings.push({
                type: "IDOR Vulnerability",
                severity: "High",
                endpoint: endpoint,
                details: idorResult.details,
            });
        }

        // Mass assignment
        const massAssignment = await this.testMassAssignment(endpoint);
        if (massAssignment.vulnerable) {
            findings.push({
                type: "Mass Assignment",
                severity: "Medium",
                endpoint: endpoint,
                details: massAssignment.details,
            });
        }

        return findings;
    }

    async testRateLimiting() {
        const findings = [];
        const endpoints = [
            "/api/v1/auth/login",
            "/api/v1/secrets",
            "/api/v1/projects",
        ];

        for (const endpoint of endpoints) {
            const requests = [];

            // Send 1000 requests rapidly
            for (let i = 0; i < 1000; i++) {
                requests.push(this.makeRequest(endpoint));
            }

            const responses = await Promise.all(requests);
            const successCount = responses.filter(
                (r) => r.status === 200,
            ).length;

            if (successCount > 100) {
                findings.push({
                    type: "Insufficient Rate Limiting",
                    severity: "Medium",
                    endpoint: endpoint,
                    details: `${successCount}/1000 requests succeeded`,
                });
            }
        }

        return findings;
    }
}
```

### Infrastructure Testing

```javascript
// Infrastructure penetration testing
class InfraPentest {
    async performTesting() {
        const findings = [];

        // Network scanning
        findings.push(...(await this.networkScanning()));

        // Service enumeration
        findings.push(...(await this.serviceEnumeration()));

        // Configuration review
        findings.push(...(await this.configurationReview()));

        // Cloud security assessment
        findings.push(...(await this.cloudAssessment()));

        return findings;
    }

    async networkScanning() {
        const findings = [];
        const networks = ["10.0.0.0/24", "10.0.1.0/24"];

        for (const network of networks) {
            // Port scanning
            const openPorts = await this.scanPorts(network);

            for (const host of openPorts) {
                // Check for unnecessary open ports
                const unnecessaryPorts = host.ports.filter(
                    (p) => ![80, 443, 22].includes(p.port),
                );

                if (unnecessaryPorts.length > 0) {
                    findings.push({
                        type: "Unnecessary Open Ports",
                        severity: "Low",
                        host: host.ip,
                        ports: unnecessaryPorts,
                    });
                }

                // Check for outdated services
                for (const port of host.ports) {
                    const version = await this.getServiceVersion(
                        host.ip,
                        port.port,
                    );
                    if (this.isOutdated(version)) {
                        findings.push({
                            type: "Outdated Service",
                            severity: "Medium",
                            host: host.ip,
                            service: version.service,
                            version: version.version,
                            latest: version.latest,
                        });
                    }
                }
            }
        }

        return findings;
    }

    async cloudAssessment() {
        const findings = [];

        // S3 bucket security
        const buckets = await this.listS3Buckets();
        for (const bucket of buckets) {
            // Check public access
            if (bucket.publicAccess) {
                findings.push({
                    type: "Public S3 Bucket",
                    severity: "High",
                    resource: bucket.name,
                    details: "Bucket allows public access",
                });
            }

            // Check encryption
            if (!bucket.encryption) {
                findings.push({
                    type: "Unencrypted S3 Bucket",
                    severity: "Medium",
                    resource: bucket.name,
                    details: "Bucket not encrypted at rest",
                });
            }
        }

        // IAM assessment
        const iamFindings = await this.assessIAM();
        findings.push(...iamFindings);

        // Network security
        const networkFindings = await this.assessNetworkSecurity();
        findings.push(...networkFindings);

        return findings;
    }
}
```

## Vulnerability Assessment

### Risk Rating

```javascript
// CVSS v3.1 scoring implementation
class CVSSCalculator {
    calculateScore(vulnerability) {
        const baseScore = this.calculateBaseScore(vulnerability);
        const temporalScore = this.calculateTemporalScore(
            vulnerability,
            baseScore,
        );
        const environmentalScore = this.calculateEnvironmentalScore(
            vulnerability,
            temporalScore,
        );

        return {
            base: baseScore,
            temporal: temporalScore,
            environmental: environmentalScore,
            vector: this.generateVector(vulnerability),
            severity: this.getSeverityRating(environmentalScore),
        };
    }

    calculateBaseScore(vuln) {
        // Base metrics
        const attackVector = this.getAttackVector(vuln.av);
        const attackComplexity = this.getAttackComplexity(vuln.ac);
        const privilegesRequired = this.getPrivilegesRequired(vuln.pr);
        const userInteraction = this.getUserInteraction(vuln.ui);
        const scope = this.getScope(vuln.s);
        const confidentiality = this.getConfidentiality(vuln.c);
        const integrity = this.getIntegrity(vuln.i);
        const availability = this.getAvailability(vuln.a);

        // Calculate exploitability
        const exploitability =
            8.22 *
            attackVector *
            attackComplexity *
            privilegesRequired *
            userInteraction;

        // Calculate impact
        const impactBase =
            1 - (1 - confidentiality) * (1 - integrity) * (1 - availability);
        const impact =
            scope === "unchanged"
                ? 6.42 * impactBase
                : 7.52 * (impactBase - 0.029) -
                  3.25 * Math.pow(impactBase - 0.02, 15);

        // Calculate base score
        if (impact <= 0) return 0;

        const baseScore =
            scope === "unchanged"
                ? Math.min(impact + exploitability, 10)
                : Math.min(1.08 * (impact + exploitability), 10);

        return Math.round(baseScore * 10) / 10;
    }

    getSeverityRating(score) {
        if (score === 0) return "None";
        if (score <= 3.9) return "Low";
        if (score <= 6.9) return "Medium";
        if (score <= 8.9) return "High";
        return "Critical";
    }
}
```

### Finding Classification

```javascript
// Vulnerability classification system
const vulnerabilityClassification = {
    critical: {
        criteria: [
            "Remote code execution",
            "Authentication bypass",
            "Privilege escalation to admin",
            "Mass data exposure",
            "Encryption key compromise",
        ],

        response_time: "24 hours",

        examples: {
            rce: {
                title: "Remote Code Execution via Deserialization",
                cvss: "CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H",
                score: 10.0,
            },

            auth_bypass: {
                title: "Authentication Bypass in API",
                cvss: "CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N",
                score: 9.1,
            },
        },
    },

    high: {
        criteria: [
            "SQL injection",
            "Stored XSS",
            "IDOR with data access",
            "Privilege escalation",
            "Session hijacking",
        ],

        response_time: "7 days",

        examples: {
            sqli: {
                title: "SQL Injection in Search Function",
                cvss: "CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:L",
                score: 8.3,
            },
        },
    },

    medium: {
        criteria: [
            "Reflected XSS",
            "CSRF",
            "Information disclosure",
            "Weak cryptography",
            "Missing security headers",
        ],

        response_time: "30 days",

        examples: {
            xss: {
                title: "Reflected XSS in Error Page",
                cvss: "CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N",
                score: 6.1,
            },
        },
    },

    low: {
        criteria: [
            "Information leakage",
            "Missing best practices",
            "Verbose error messages",
            "Outdated dependencies",
            "Configuration issues",
        ],

        response_time: "90 days",
    },
};
```

## Reporting

### Report Template

````markdown
# Penetration Testing Report

## Executive Summary

### Testing Overview

- **Client**: DotEnv Inc.
- **Testing Period**: {{start_date}} - {{end_date}}
- **Methodology**: OWASP, PTES, NIST
- **Scope**: {{scope_summary}}

### Key Findings

- **Critical**: {{critical_count}}
- **High**: {{high_count}}
- **Medium**: {{medium_count}}
- **Low**: {{low_count}}
- **Informational**: {{info_count}}

### Risk Summary

{{risk_summary}}

## Technical Findings

### Finding 1: {{vulnerability_title}}

**Severity**: {{severity}}
**CVSS Score**: {{cvss_score}} ({{cvss_vector}})
**Affected Component**: {{component}}

#### Description

{{detailed_description}}

#### Proof of Concept

```{{language}}
{{poc_code}}
```
````

#### Impact

{{impact_description}}

#### Recommendation

{{remediation_steps}}

#### References

- {{reference_1}}
- {{reference_2}}

## Methodology

### Tools Used

{{tools_list}}

### Testing Approach

{{approach_description}}

## Recommendations

### Immediate Actions

1. {{immediate_1}}
2. {{immediate_2}}

### Short-term Improvements

1. {{short_term_1}}
2. {{short_term_2}}

### Long-term Enhancements

1. {{long_term_1}}
2. {{long_term_2}}

````

### Proof of Concept Guidelines

```javascript
// Safe PoC development
class ProofOfConcept {
  constructor() {
    this.guidelines = {
      minimize_impact: true,
      avoid_real_data: true,
      provide_remediation: true,
      include_indicators: true
    };
  }

  createPoC(vulnerability) {
    const poc = {
      title: vulnerability.title,
      description: vulnerability.description,
      prerequisites: [],
      steps: [],
      indicators: [],
      cleanup: []
    };

    // Add safe demonstration steps
    switch (vulnerability.type) {
      case 'sql_injection':
        poc.steps = [
          'Navigate to /search',
          "Enter payload: ' OR 1=1--",
          'Observe all records returned',
          'Note: Use test account only'
        ];
        poc.indicators = [
          'HTTP 200 response',
          'Multiple records in response',
          'No SQL error message'
        ];
        break;

      case 'xss':
        poc.steps = [
          'Navigate to vulnerable endpoint',
          'Insert payload: <img src=x onerror=alert(1)>',
          'Observe JavaScript execution',
          'Test in isolated browser'
        ];
        poc.indicators = [
          'Alert box appears',
          'Payload reflected in response',
          'No Content-Security-Policy header'
        ];
        break;
    }

    return poc;
  }
}
````

## Remediation Tracking

### Fix Verification

```javascript
// Remediation verification process
class RemediationVerifier {
    async verifyFix(vulnerability, fix) {
        const verification = {
            vulnerability_id: vulnerability.id,
            fix_id: fix.id,
            timestamp: new Date(),
            results: [],
        };

        // Re-test the vulnerability
        const retestResult = await this.retestVulnerability(vulnerability);
        verification.results.push({
            test: "vulnerability_retest",
            passed: !retestResult.stillVulnerable,
            details: retestResult,
        });

        // Test for regression
        const regressionResult = await this.testRegression(vulnerability, fix);
        verification.results.push({
            test: "regression_test",
            passed: regressionResult.noRegression,
            details: regressionResult,
        });

        // Verify security controls
        const controlsResult = await this.verifyControls(vulnerability, fix);
        verification.results.push({
            test: "security_controls",
            passed: controlsResult.allImplemented,
            details: controlsResult,
        });

        // Performance impact
        const performanceResult = await this.testPerformance(
            vulnerability,
            fix,
        );
        verification.results.push({
            test: "performance_impact",
            passed: performanceResult.acceptable,
            details: performanceResult,
        });

        verification.overall = verification.results.every((r) => r.passed);

        return verification;
    }
}
```

## Bug Bounty Program

### Program Scope

```yaml
# Bug Bounty Program Configuration
bug_bounty:
    program_url: "https://hackerone.com/dotenv"

    in_scope:
        - "*.dotenv.cloud"
        - "DotEnv Mobile Apps"
        - "DotEnv CLI"
        - "DotEnv SDKs"

    out_of_scope:
        - "blog.dotenv.cloud"
        - "status.dotenv.cloud"
        - "Social engineering"
        - "Physical attacks"
        - "DoS attacks"

    rewards:
        critical:
            range: "$5,000 - $10,000"
            criteria: "RCE, Auth Bypass, Crypto Issues"

        high:
            range: "$1,000 - $5,000"
            criteria: "SQLi, Stored XSS, Privilege Escalation"

        medium:
            range: "$250 - $1,000"
            criteria: "CSRF, Reflected XSS, IDOR"

        low:
            range: "$50 - $250"
            criteria: "Info Disclosure, Missing Headers"

    rules:
        - "Do not access customer data"
        - "Create test accounts only"
        - "Report immediately upon discovery"
        - "Do not publicly disclose"
        - "Provide clear reproduction steps"
```

### Responsible Disclosure

```javascript
// Vulnerability disclosure process
const disclosureProcess = {
    initial_report: {
        channel: "security@dotenv.cloud",
        pgp_key: "https://dotenv.cloud/security.asc",
        expected_response: "24 hours",

        required_information: [
            "Vulnerability type",
            "Affected components",
            "Reproduction steps",
            "Impact assessment",
            "Suggested fix",
        ],
    },

    triage: {
        severity_assessment: "48 hours",
        initial_fix: {
            critical: "24 hours",
            high: "7 days",
            medium: "30 days",
            low: "90 days",
        },
    },

    communication: {
        updates: "Every 7 days",
        fix_notification: "Within 24 hours",
        public_disclosure: "90 days from fix",
    },

    recognition: {
        hall_of_fame: true,
        swag: true,
        monetary_reward: true,
        reference_letter: "upon request",
    },
};
```

## Resources

### Testing Standards

- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [PTES Technical Guidelines](http://www.pentest-standard.org/index.php/Main_Page)
- [NIST SP 800-115](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-115.pdf)
- [SANS Penetration Testing](https://www.sans.org/penetration-testing/)

### Tools

- [Burp Suite Pro](https://portswigger.net/burp/pro)
- [OWASP ZAP](https://www.zaproxy.org/)
- [Metasploit](https://www.metasploit.com/)
- [Nmap](https://nmap.org/)

### DotEnv Security Resources

- [Security Portal](https://security.dotenv.cloud)
- [Bug Bounty Program](https://hackerone.com/dotenv)
- [Security Contact](mailto:security@dotenv.cloud)
- [PGP Key](https://dotenv.cloud/security.asc)

## Next Steps

- [Review security best practices](/documentation/v1/security-compliance/security-best-practices)
- [Implement security monitoring](/documentation/v1/administration/monitoring)
- [Configure incident response](/documentation/v1/security-compliance/incident-response)
- [Enable audit logging](/documentation/v1/security-compliance/audit-logging)
