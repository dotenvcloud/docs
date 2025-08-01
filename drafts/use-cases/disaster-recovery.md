---
title: Disaster Recovery
slug: disaster-recovery
order: 9
tags: [use-cases, disaster-recovery, backup, security]
---

# Disaster Recovery

Learn how to implement comprehensive disaster recovery strategies using DotEnv. This guide covers backup procedures, recovery planning, incident response, and maintaining business continuity during critical events.

## Overview

Disaster recovery challenges:

- Data backup and restoration
- Secret recovery procedures
- Service continuity
- Incident response
- Compliance requirements
- Recovery time objectives (RTO)

## Disaster Recovery Planning

### 1. Recovery Strategy

```yaml
# disaster-recovery-plan.yaml
recovery_objectives:
    rto: # Recovery Time Objective
        critical_services: 15m
        standard_services: 1h
        non_critical: 4h

    rpo: # Recovery Point Objective
        database: 5m
        secrets: 1h
        files: 24h

disaster_scenarios:
    - name: "Data Center Outage"
      probability: medium
      impact: critical
      recovery_strategy: "Failover to secondary region"

    - name: "Database Corruption"
      probability: low
      impact: critical
      recovery_strategy: "Point-in-time recovery"

    - name: "Secret Compromise"
      probability: medium
      impact: high
      recovery_strategy: "Immediate rotation and audit"

    - name: "Ransomware Attack"
      probability: low
      impact: critical
      recovery_strategy: "Isolated backup restoration"

    - name: "Human Error"
      probability: high
      impact: medium
      recovery_strategy: "Rollback to previous state"

recovery_tiers:
    tier_1: # Mission Critical
        services: ["api", "database", "auth"]
        max_downtime: 15m
        data_loss_tolerance: 5m

    tier_2: # Business Critical
        services: ["admin", "reporting", "webhooks"]
        max_downtime: 1h
        data_loss_tolerance: 30m

    tier_3: # Standard
        services: ["analytics", "notifications"]
        max_downtime: 4h
        data_loss_tolerance: 2h
```

### 2. Infrastructure Setup

```terraform
# terraform/disaster-recovery/dr-infrastructure.tf
module "dr_infrastructure" {
  source = "./modules/dr"

  primary_region   = "us-east-1"
  secondary_region = "us-west-2"

  # Cross-region replication
  enable_database_replication = true
  enable_storage_replication  = true
  enable_secret_replication   = true

  # Backup configuration
  backup_schedule = {
    full_backup    = "0 2 * * *"  # 2 AM daily
    incremental    = "*/15 * * * *" # Every 15 minutes
    secret_backup  = "0 * * * *"   # Hourly
  }

  # Retention policies
  retention_policy = {
    daily_backups   = 7
    weekly_backups  = 4
    monthly_backups = 12
    yearly_backups  = 3
  }
}

# DR Database Setup
resource "aws_db_instance" "dr_database" {
  identifier = "dr-database"

  # Read replica of production
  replicate_source_db = aws_db_instance.production.id

  # Promote to master in DR scenario
  backup_retention_period = 30
  backup_window          = "03:00-04:00"

  # Different region
  availability_zone = data.aws_availability_zones.dr.names[0]

  tags = {
    Environment = "disaster-recovery"
    Purpose     = "standby"
  }
}

# DR Secret Replication
resource "aws_secretsmanager_secret" "dr_replication" {
  name = "dr-secret-replication"

  replica {
    region = var.secondary_region

    kms_key_id = aws_kms_key.dr_key.id
  }
}
```

## Backup Procedures

### 1. Automated Backup System

```javascript
// services/backup-service.js
const AWS = require("aws-sdk");
const { DotEnvClient } = require("@dotenv/sdk");
const crypto = require("crypto");

class BackupService {
    constructor() {
        this.s3 = new AWS.S3();
        this.glacier = new AWS.Glacier();
        this.dotenv = new DotEnvClient({
            apiKey: process.env.DOTENV_API_KEY,
        });

        this.backupConfig = {
            bucket: process.env.BACKUP_BUCKET,
            glacierVault: process.env.GLACIER_VAULT,
            encryptionKey: process.env.BACKUP_ENCRYPTION_KEY,
        };
    }

    async performBackup(type = "incremental") {
        const backupJob = {
            id: `backup-${Date.now()}`,
            type,
            startTime: new Date(),
            components: [],
        };

        console.log(`Starting ${type} backup: ${backupJob.id}`);

        try {
            // 1. Backup database
            const dbBackup = await this.backupDatabase(backupJob.id, type);
            backupJob.components.push(dbBackup);

            // 2. Backup secrets
            const secretsBackup = await this.backupSecrets(backupJob.id);
            backupJob.components.push(secretsBackup);

            // 3. Backup application files
            const filesBackup = await this.backupFiles(backupJob.id, type);
            backupJob.components.push(filesBackup);

            // 4. Backup configuration
            const configBackup = await this.backupConfiguration(backupJob.id);
            backupJob.components.push(configBackup);

            // 5. Create backup manifest
            backupJob.endTime = new Date();
            backupJob.status = "completed";
            await this.createManifest(backupJob);

            // 6. Verify backup integrity
            await this.verifyBackup(backupJob.id);

            // 7. Archive to Glacier if full backup
            if (type === "full") {
                await this.archiveToGlacier(backupJob);
            }

            console.log(`✅ Backup completed: ${backupJob.id}`);
            return backupJob;
        } catch (error) {
            backupJob.status = "failed";
            backupJob.error = error.message;
            await this.handleBackupFailure(backupJob);
            throw error;
        }
    }

    async backupDatabase(backupId, type) {
        console.log("Backing up database...");

        const pg = require("pg");
        const { spawn } = require("child_process");
        const fs = require("fs");

        const timestamp = new Date().toISOString();
        const filename = `db-${type}-${timestamp}.sql.gz`;
        const filepath = `/tmp/${filename}`;

        // Use pg_dump for backup
        const pgDump = spawn(
            "pg_dump",
            [
                process.env.DATABASE_URL,
                "--no-owner",
                "--no-acl",
                type === "incremental" ? "--data-only" : "",
                "|",
                "gzip",
                ">",
                filepath,
            ],
            { shell: true },
        );

        await new Promise((resolve, reject) => {
            pgDump.on("exit", (code) => {
                if (code === 0) resolve();
                else reject(new Error(`pg_dump failed with code ${code}`));
            });
        });

        // Encrypt backup
        const encrypted = await this.encryptFile(filepath);

        // Upload to S3
        await this.s3
            .putObject({
                Bucket: this.backupConfig.bucket,
                Key: `database/${backupId}/${filename}.enc`,
                Body: encrypted,
                StorageClass: "STANDARD_IA",
                ServerSideEncryption: "AES256",
                Metadata: {
                    backupId,
                    type,
                    timestamp,
                    size: encrypted.length.toString(),
                },
            })
            .promise();

        // Clean up
        fs.unlinkSync(filepath);

        return {
            component: "database",
            type,
            filename: `${filename}.enc`,
            size: encrypted.length,
            timestamp,
        };
    }

    async backupSecrets(backupId) {
        console.log("Backing up secrets...");

        const projects = await this.dotenv.listProjects();
        const allSecrets = {};

        for (const project of projects) {
            const environments = await this.dotenv.listEnvironments(
                project.name,
            );
            allSecrets[project.name] = {};

            for (const env of environments) {
                const secrets = await this.dotenv.getSecrets(
                    project.name,
                    env.name,
                );
                allSecrets[project.name][env.name] = secrets;
            }
        }

        // Create encrypted backup
        const secretsJson = JSON.stringify(allSecrets, null, 2);
        const encrypted = await this.encrypt(secretsJson);

        // Store with versioning
        const timestamp = new Date().toISOString();
        await this.s3
            .putObject({
                Bucket: this.backupConfig.bucket,
                Key: `secrets/${backupId}/secrets-${timestamp}.json.enc`,
                Body: encrypted,
                ServerSideEncryption: "aws:kms",
                SSEKMSKeyId: process.env.KMS_KEY_ID,
                Metadata: {
                    backupId,
                    timestamp,
                    projectCount: projects.length.toString(),
                },
            })
            .promise();

        // Also store in Secrets Manager for quick access
        await this.storeInSecretsManager(backupId, encrypted);

        return {
            component: "secrets",
            projectCount: projects.length,
            timestamp,
            encrypted: true,
        };
    }

    async backupFiles(backupId, type) {
        console.log("Backing up application files...");

        const tar = require("tar");
        const fs = require("fs");

        const timestamp = new Date().toISOString();
        const filename = `files-${type}-${timestamp}.tar.gz`;
        const filepath = `/tmp/${filename}`;

        // Directories to backup
        const directories = [
            "/var/app/uploads",
            "/var/app/config",
            "/var/app/certificates",
        ];

        // Create tar archive
        await tar.create(
            {
                gzip: true,
                file: filepath,
                preservePaths: false,
            },
            directories,
        );

        // Encrypt archive
        const encrypted = await this.encryptFile(filepath);

        // Upload to S3
        const upload = await this.s3
            .upload({
                Bucket: this.backupConfig.bucket,
                Key: `files/${backupId}/${filename}.enc`,
                Body: fs.createReadStream(filepath),
                StorageClass: type === "full" ? "GLACIER" : "STANDARD_IA",
                Metadata: {
                    backupId,
                    type,
                    timestamp,
                },
            })
            .promise();

        // Clean up
        fs.unlinkSync(filepath);

        return {
            component: "files",
            type,
            filename: `${filename}.enc`,
            size: fs.statSync(filepath).size,
            timestamp,
        };
    }

    async createManifest(backupJob) {
        const manifest = {
            ...backupJob,
            verification: {
                checksums: {},
                signatures: {},
            },
        };

        // Calculate checksums
        for (const component of backupJob.components) {
            manifest.verification.checksums[component.component] =
                await this.calculateChecksum(component);
        }

        // Sign manifest
        manifest.verification.signature = await this.signManifest(manifest);

        // Store manifest
        await this.s3
            .putObject({
                Bucket: this.backupConfig.bucket,
                Key: `manifests/${backupJob.id}/manifest.json`,
                Body: JSON.stringify(manifest, null, 2),
                ServerSideEncryption: "AES256",
            })
            .promise();

        return manifest;
    }

    async verifyBackup(backupId) {
        console.log("Verifying backup integrity...");

        // Get manifest
        const manifestObj = await this.s3
            .getObject({
                Bucket: this.backupConfig.bucket,
                Key: `manifests/${backupId}/manifest.json`,
            })
            .promise();

        const manifest = JSON.parse(manifestObj.Body.toString());

        // Verify each component
        for (const component of manifest.components) {
            const exists = await this.verifyComponentExists(
                backupId,
                component,
            );
            if (!exists) {
                throw new Error(
                    `Missing backup component: ${component.component}`,
                );
            }

            const checksumValid = await this.verifyChecksum(
                backupId,
                component,
                manifest.verification.checksums[component.component],
            );

            if (!checksumValid) {
                throw new Error(
                    `Checksum mismatch for: ${component.component}`,
                );
            }
        }

        console.log("✅ Backup verification passed");
    }

    async archiveToGlacier(backupJob) {
        console.log("Archiving to Glacier...");

        // Create archive description
        const archiveDescription = JSON.stringify({
            backupId: backupJob.id,
            type: backupJob.type,
            timestamp: backupJob.startTime,
            components: backupJob.components.map((c) => c.component),
        });

        // Upload to Glacier
        const archive = await this.glacier
            .uploadArchive({
                vaultName: this.backupConfig.glacierVault,
                archiveDescription,
                body: await this.createArchiveBundle(backupJob),
            })
            .promise();

        // Store archive ID for retrieval
        await this.s3
            .putObject({
                Bucket: this.backupConfig.bucket,
                Key: `glacier-index/${backupJob.id}.json`,
                Body: JSON.stringify({
                    archiveId: archive.archiveId,
                    location: archive.location,
                    checksum: archive.checksum,
                    timestamp: new Date(),
                }),
            })
            .promise();

        console.log(`✅ Archived to Glacier: ${archive.archiveId}`);
    }

    encrypt(data) {
        const algorithm = "aes-256-gcm";
        const key = Buffer.from(this.backupConfig.encryptionKey, "hex");
        const iv = crypto.randomBytes(16);

        const cipher = crypto.createCipheriv(algorithm, key, iv);

        let encrypted = cipher.update(data, "utf8", "hex");
        encrypted += cipher.final("hex");

        const authTag = cipher.getAuthTag();

        return Buffer.concat([iv, authTag, Buffer.from(encrypted, "hex")]);
    }

    decrypt(encryptedData) {
        const algorithm = "aes-256-gcm";
        const key = Buffer.from(this.backupConfig.encryptionKey, "hex");

        const iv = encryptedData.slice(0, 16);
        const authTag = encryptedData.slice(16, 32);
        const encrypted = encryptedData.slice(32);

        const decipher = crypto.createDecipheriv(algorithm, key, iv);
        decipher.setAuthTag(authTag);

        let decrypted = decipher.update(encrypted, null, "utf8");
        decrypted += decipher.final("utf8");

        return decrypted;
    }
}

module.exports = BackupService;
```

### 2. Recovery Testing

```javascript
// scripts/test-disaster-recovery.js
class DisasterRecoveryTest {
    constructor() {
        this.testResults = [];
        this.startTime = new Date();
    }

    async runDRTest(scenario = "full") {
        console.log("🧪 Starting Disaster Recovery Test");
        console.log(`Scenario: ${scenario}`);
        console.log(`Time: ${this.startTime.toISOString()}`);

        try {
            // 1. Create test environment
            await this.createTestEnvironment();

            // 2. Simulate disaster
            await this.simulateDisaster(scenario);

            // 3. Execute recovery
            const recoveryStart = Date.now();
            await this.executeRecovery(scenario);
            const recoveryTime = Date.now() - recoveryStart;

            // 4. Verify recovery
            await this.verifyRecovery();

            // 5. Test business operations
            await this.testBusinessOperations();

            // 6. Generate report
            const report = await this.generateReport(recoveryTime);

            console.log("✅ DR Test completed successfully");
            return report;
        } catch (error) {
            console.error("❌ DR Test failed:", error);
            throw error;
        } finally {
            // Clean up test environment
            await this.cleanup();
        }
    }

    async createTestEnvironment() {
        console.log("Creating test environment...");

        // Spin up DR infrastructure
        const terraform = require("./terraform-wrapper");

        await terraform.apply("disaster-recovery/test", {
            variables: {
                environment: "dr-test",
                scale: "minimal",
            },
        });

        // Wait for infrastructure
        await this.waitForInfrastructure();
    }

    async simulateDisaster(scenario) {
        console.log(`Simulating disaster: ${scenario}`);

        const scenarios = {
            "database-failure": async () => {
                // Stop primary database
                await this.stopService("database");
                this.recordEvent("Database stopped");
            },

            "region-outage": async () => {
                // Simulate region failure
                await this.blockRegion("us-east-1");
                this.recordEvent("Region blocked");
            },

            "data-corruption": async () => {
                // Corrupt test data
                await this.corruptData();
                this.recordEvent("Data corrupted");
            },

            "secret-compromise": async () => {
                // Simulate compromised secrets
                await this.compromiseSecrets();
                this.recordEvent("Secrets compromised");
            },

            full: async () => {
                // Complete system failure
                await this.stopAllServices();
                this.recordEvent("All services stopped");
            },
        };

        await scenarios[scenario]();
    }

    async executeRecovery(scenario) {
        console.log("Executing recovery procedures...");

        const recovery = require("./disaster-recovery");

        // Get latest backup
        const latestBackup = await this.getLatestBackup();
        this.recordEvent(`Using backup: ${latestBackup.id}`);

        // Execute recovery based on scenario
        await recovery.recover({
            scenario,
            backupId: latestBackup.id,
            targetEnvironment: "dr-test",
            options: {
                verifyData: true,
                rotateSecrets: scenario === "secret-compromise",
                notifyTeam: false, // Test mode
            },
        });

        this.recordEvent("Recovery executed");
    }

    async verifyRecovery() {
        console.log("Verifying recovery...");

        const checks = [
            { name: "Database connectivity", fn: this.checkDatabase },
            { name: "Application health", fn: this.checkApplication },
            { name: "Secret access", fn: this.checkSecrets },
            { name: "Data integrity", fn: this.checkDataIntegrity },
            { name: "Service dependencies", fn: this.checkDependencies },
        ];

        for (const check of checks) {
            const start = Date.now();
            const result = await check.fn.call(this);
            const duration = Date.now() - start;

            this.testResults.push({
                check: check.name,
                passed: result.success,
                duration,
                details: result.details,
            });

            if (!result.success) {
                throw new Error(`Verification failed: ${check.name}`);
            }
        }
    }

    async testBusinessOperations() {
        console.log("Testing business operations...");

        const operations = [
            {
                name: "User login",
                test: async () => {
                    const response = await fetch(
                        `${this.drUrl}/api/auth/login`,
                        {
                            method: "POST",
                            headers: { "Content-Type": "application/json" },
                            body: JSON.stringify({
                                email: "test@example.com",
                                password: "test123",
                            }),
                        },
                    );
                    return response.ok;
                },
            },
            {
                name: "Create order",
                test: async () => {
                    const response = await fetch(`${this.drUrl}/api/orders`, {
                        method: "POST",
                        headers: {
                            "Content-Type": "application/json",
                            Authorization: `Bearer ${this.testToken}`,
                        },
                        body: JSON.stringify({
                            items: [{ productId: 1, quantity: 1 }],
                        }),
                    });
                    return response.ok;
                },
            },
            {
                name: "Process payment",
                test: async () => {
                    // Test payment processing
                    const response = await fetch(
                        `${this.drUrl}/api/payments/test`,
                    );
                    return response.ok;
                },
            },
        ];

        for (const op of operations) {
            const start = Date.now();
            const success = await op.test();
            const duration = Date.now() - start;

            this.testResults.push({
                operation: op.name,
                success,
                duration,
                businessImpact: success ? "none" : "critical",
            });
        }
    }

    async generateReport(recoveryTime) {
        const report = {
            testId: `dr-test-${Date.now()}`,
            scenario: this.scenario,
            startTime: this.startTime,
            endTime: new Date(),
            recoveryTime: recoveryTime / 1000, // seconds
            rtoMet: recoveryTime < 15 * 60 * 1000, // 15 min RTO
            results: this.testResults,
            events: this.events,
            recommendations: [],
        };

        // Analyze results
        if (!report.rtoMet) {
            report.recommendations.push(
                "Recovery time exceeded RTO. Consider optimizing backup restoration process.",
            );
        }

        const failedChecks = this.testResults.filter(
            (r) => !r.passed && !r.success,
        );
        if (failedChecks.length > 0) {
            report.recommendations.push(
                `${failedChecks.length} checks failed. Review and fix issues.`,
            );
        }

        // Save report
        const fs = require("fs").promises;
        await fs.writeFile(
            `dr-test-reports/${report.testId}.json`,
            JSON.stringify(report, null, 2),
        );

        // Send summary
        await this.sendTestSummary(report);

        return report;
    }

    recordEvent(event) {
        if (!this.events) this.events = [];
        this.events.push({
            timestamp: new Date(),
            event,
        });
    }
}

// Run DR test
if (require.main === module) {
    const test = new DisasterRecoveryTest();
    const scenario = process.argv[2] || "full";

    test.runDRTest(scenario)
        .then((report) => {
            console.log("\nTest Summary:");
            console.log(`Recovery Time: ${report.recoveryTime}s`);
            console.log(`RTO Met: ${report.rtoMet ? "Yes" : "No"}`);
            console.log(
                `Checks Passed: ${report.results.filter((r) => r.passed).length}/${report.results.length}`,
            );
        })
        .catch(console.error);
}

module.exports = DisasterRecoveryTest;
```

## Incident Response

### 1. Incident Management System

```javascript
// services/incident-manager.js
class IncidentManager {
    constructor() {
        this.incidents = new Map();
        this.responders = new Map();
    }

    async createIncident(details) {
        const incident = {
            id: `INC-${Date.now()}`,
            ...details,
            status: "open",
            severity: this.calculateSeverity(details),
            createdAt: new Date(),
            timeline: [],
            responders: [],
            affectedServices: [],
        };

        // Log incident creation
        this.addTimelineEvent(incident, "Incident created", details);

        // Store incident
        this.incidents.set(incident.id, incident);

        // Start response procedures
        await this.initiateResponse(incident);

        return incident;
    }

    calculateSeverity(details) {
        // SEV1: Complete outage, data loss, security breach
        if (
            details.type === "security-breach" ||
            details.impact === "complete-outage" ||
            details.dataLoss === true
        ) {
            return "SEV1";
        }

        // SEV2: Partial outage, degraded performance
        if (
            details.impact === "partial-outage" ||
            details.affectedUsers > 1000
        ) {
            return "SEV2";
        }

        // SEV3: Minor issue, limited impact
        if (details.affectedUsers < 100) {
            return "SEV3";
        }

        return "SEV2"; // Default
    }

    async initiateResponse(incident) {
        console.log(
            `🚨 Initiating response for ${incident.id} (${incident.severity})`,
        );

        // 1. Page on-call
        await this.pageOnCall(incident);

        // 2. Create communication channels
        await this.createCommsChannels(incident);

        // 3. Start automated diagnostics
        await this.runDiagnostics(incident);

        // 4. Implement immediate mitigations
        await this.applyMitigations(incident);

        // 5. Notify stakeholders
        await this.notifyStakeholders(incident);
    }

    async pageOnCall(incident) {
        const pagerDuty = require("./pagerduty-client");

        const escalationPolicy = {
            SEV1: [
                "primary-oncall",
                "secondary-oncall",
                "engineering-lead",
                "cto",
            ],
            SEV2: ["primary-oncall", "secondary-oncall"],
            SEV3: ["primary-oncall"],
        };

        const responders = escalationPolicy[incident.severity];

        for (const responder of responders) {
            const response = await pagerDuty.page({
                to: responder,
                incident: incident.id,
                severity: incident.severity,
                title: incident.title,
                urgency: incident.severity === "SEV1" ? "high" : "low",
            });

            if (response.acknowledged) {
                incident.responders.push({
                    name: responder,
                    acknowledgedAt: new Date(),
                });

                this.addTimelineEvent(incident, `${responder} acknowledged`, {
                    responseTime: response.responseTime,
                });

                break; // Stop escalation if acknowledged
            }
        }
    }

    async createCommsChannels(incident) {
        const slack = require("./slack-client");

        // Create incident channel
        const channel = await slack.createChannel({
            name: `incident-${incident.id.toLowerCase()}`,
            purpose: `Response channel for ${incident.title}`,
            private: incident.severity === "SEV1",
        });

        incident.slackChannel = channel.id;

        // Post initial status
        await slack.postMessage(channel.id, {
            text: `🚨 *${incident.id}* - ${incident.title}`,
            attachments: [
                {
                    color: incident.severity === "SEV1" ? "danger" : "warning",
                    fields: [
                        {
                            title: "Severity",
                            value: incident.severity,
                            short: true,
                        },
                        {
                            title: "Status",
                            value: incident.status,
                            short: true,
                        },
                        {
                            title: "Impact",
                            value: incident.impact,
                            short: false,
                        },
                    ],
                },
            ],
        });

        // Set up video bridge for SEV1
        if (incident.severity === "SEV1") {
            const zoom = require("./zoom-client");
            const meeting = await zoom.createMeeting({
                topic: `${incident.id} War Room`,
                duration: 240, // 4 hours
            });

            incident.warRoomUrl = meeting.join_url;

            await slack.postMessage(channel.id, {
                text: `📹 War Room: ${meeting.join_url}`,
            });
        }
    }

    async runDiagnostics(incident) {
        const diagnostics = {
            "application-error": ["logs", "metrics", "traces"],
            "database-issue": ["queries", "connections", "locks"],
            "performance-degradation": ["cpu", "memory", "latency"],
            "security-breach": ["access-logs", "changes", "network"],
        };

        const checks = diagnostics[incident.type] || ["logs", "metrics"];

        for (const check of checks) {
            const result = await this.runDiagnosticCheck(check);

            this.addTimelineEvent(incident, `Diagnostic: ${check}`, result);

            // Add findings to incident
            if (result.findings.length > 0) {
                incident.diagnosticFindings = incident.diagnosticFindings || [];
                incident.diagnosticFindings.push(...result.findings);
            }
        }
    }

    async applyMitigations(incident) {
        const mitigations = {
            "high-load": async () => {
                // Scale up
                await this.scaleService(incident.affectedServices[0], 2.0);
                return "Scaled up services";
            },

            "database-locks": async () => {
                // Kill blocking queries
                await this.killBlockingQueries();
                return "Cleared database locks";
            },

            "memory-leak": async () => {
                // Rolling restart
                await this.rollingRestart(incident.affectedServices);
                return "Performed rolling restart";
            },

            "ddos-attack": async () => {
                // Enable DDoS protection
                await this.enableDDoSProtection();
                return "Enabled DDoS protection";
            },
        };

        const mitigation = mitigations[incident.type];
        if (mitigation) {
            const result = await mitigation();
            this.addTimelineEvent(incident, "Mitigation applied", {
                action: result,
            });
        }
    }

    async updateIncident(incidentId, updates) {
        const incident = this.incidents.get(incidentId);
        if (!incident) throw new Error("Incident not found");

        // Update fields
        Object.assign(incident, updates);

        // Log update
        this.addTimelineEvent(incident, "Incident updated", updates);

        // Check if resolved
        if (updates.status === "resolved") {
            await this.resolveIncident(incident);
        }

        return incident;
    }

    async resolveIncident(incident) {
        incident.resolvedAt = new Date();
        incident.duration = incident.resolvedAt - incident.createdAt;

        // Generate postmortem template
        const postmortem = await this.generatePostmortem(incident);

        // Archive incident data
        await this.archiveIncident(incident);

        // Send resolution notification
        await this.notifyResolution(incident);

        return postmortem;
    }

    async generatePostmortem(incident) {
        const postmortem = {
            incidentId: incident.id,
            title: incident.title,
            severity: incident.severity,
            duration: incident.duration,
            timeline: incident.timeline,

            summary: {
                whatHappened: "",
                impact: incident.impact,
                rootCause: "",
                trigger: "",
                resolution: "",
                detectionMethod: "",
            },

            metrics: {
                timeToDetect: this.calculateTimeToDetect(incident),
                timeToResolve: incident.duration,
                affectedUsers: incident.affectedUsers,
                errorRate: incident.errorRate,
            },

            actionItems: [],

            prevention: {
                whatWentWell: [],
                whatWentWrong: [],
                whereWeGotLucky: [],
            },
        };

        // Save template
        const fs = require("fs").promises;
        await fs.writeFile(
            `postmortems/${incident.id}.md`,
            this.formatPostmortem(postmortem),
        );

        return postmortem;
    }

    addTimelineEvent(incident, event, details = {}) {
        incident.timeline.push({
            timestamp: new Date(),
            event,
            details,
            author: details.author || "system",
        });
    }
}

module.exports = IncidentManager;
```

### 2. Runbook Automation

```javascript
// runbooks/automated-recovery.js
class AutomatedRunbook {
    constructor() {
        this.runbooks = new Map();
        this.loadRunbooks();
    }

    loadRunbooks() {
        this.runbooks.set("database-failover", {
            name: "Database Failover",
            description: "Failover to replica database",
            steps: [
                {
                    name: "Verify replica health",
                    action: this.verifyReplicaHealth,
                    rollback: null,
                },
                {
                    name: "Stop write traffic",
                    action: this.stopWriteTraffic,
                    rollback: this.resumeWriteTraffic,
                },
                {
                    name: "Promote replica",
                    action: this.promoteReplica,
                    rollback: this.demoteReplica,
                },
                {
                    name: "Update connection strings",
                    action: this.updateConnectionStrings,
                    rollback: this.revertConnectionStrings,
                },
                {
                    name: "Verify application health",
                    action: this.verifyApplicationHealth,
                    rollback: null,
                },
            ],
        });

        this.runbooks.set("secret-rotation-emergency", {
            name: "Emergency Secret Rotation",
            description: "Rotate all secrets after compromise",
            steps: [
                {
                    name: "Disable compromised credentials",
                    action: this.disableCompromisedCreds,
                    rollback: null,
                },
                {
                    name: "Generate new secrets",
                    action: this.generateNewSecrets,
                    rollback: null,
                },
                {
                    name: "Deploy with dual secrets",
                    action: this.deployDualSecrets,
                    rollback: this.rollbackSecrets,
                },
                {
                    name: "Rotate service by service",
                    action: this.rotateServices,
                    rollback: null,
                },
                {
                    name: "Remove old secrets",
                    action: this.removeOldSecrets,
                    rollback: null,
                },
            ],
        });
    }

    async executeRunbook(runbookName, context = {}) {
        const runbook = this.runbooks.get(runbookName);
        if (!runbook) throw new Error(`Runbook not found: ${runbookName}`);

        console.log(`🔧 Executing runbook: ${runbook.name}`);

        const execution = {
            id: `exec-${Date.now()}`,
            runbook: runbookName,
            startTime: new Date(),
            steps: [],
            status: "running",
            context,
        };

        const completedSteps = [];

        try {
            for (const step of runbook.steps) {
                console.log(`▶️  ${step.name}`);

                const stepResult = {
                    name: step.name,
                    startTime: new Date(),
                    status: "running",
                };

                try {
                    const result = await step.action.call(this, context);

                    stepResult.endTime = new Date();
                    stepResult.status = "completed";
                    stepResult.result = result;

                    completedSteps.push(step);
                    execution.steps.push(stepResult);

                    console.log(`✅ ${step.name} completed`);
                } catch (error) {
                    stepResult.endTime = new Date();
                    stepResult.status = "failed";
                    stepResult.error = error.message;

                    execution.steps.push(stepResult);

                    console.error(`❌ ${step.name} failed:`, error.message);

                    // Rollback completed steps
                    await this.rollbackSteps(completedSteps, context);

                    throw error;
                }
            }

            execution.endTime = new Date();
            execution.status = "completed";

            console.log(`✅ Runbook completed successfully`);

            return execution;
        } catch (error) {
            execution.endTime = new Date();
            execution.status = "failed";
            execution.error = error.message;

            throw error;
        }
    }

    async rollbackSteps(steps, context) {
        console.log("🔄 Rolling back completed steps...");

        // Rollback in reverse order
        for (const step of steps.reverse()) {
            if (step.rollback) {
                try {
                    console.log(`↩️  Rolling back: ${step.name}`);
                    await step.rollback.call(this, context);
                    console.log(`✅ Rolled back: ${step.name}`);
                } catch (error) {
                    console.error(
                        `❌ Rollback failed for ${step.name}:`,
                        error.message,
                    );
                    // Continue with other rollbacks
                }
            }
        }
    }

    // Runbook actions
    async verifyReplicaHealth(context) {
        const health = await this.checkDatabaseHealth(context.replicaHost);
        if (!health.healthy) {
            throw new Error("Replica is not healthy");
        }

        const lag = await this.checkReplicationLag(context.replicaHost);
        if (lag > 60) {
            // 60 seconds
            throw new Error(`Replication lag too high: ${lag}s`);
        }

        return { healthy: true, lag };
    }

    async promoteReplica(context) {
        const rds = new AWS.RDS();

        const response = await rds
            .promoteReadReplica({
                DBInstanceIdentifier: context.replicaId,
                BackupRetentionPeriod: 7,
            })
            .promise();

        // Wait for promotion
        await rds
            .waitFor("dBInstanceAvailable", {
                DBInstanceIdentifier: context.replicaId,
            })
            .promise();

        return response;
    }

    async updateConnectionStrings(context) {
        const dotenv = new DotEnvClient();

        // Update production secrets
        await dotenv.updateSecret(
            "production",
            "DATABASE_URL",
            context.newDatabaseUrl,
        );

        // Trigger application restart
        await this.restartApplications();

        return { updated: true };
    }
}

module.exports = AutomatedRunbook;
```

## Recovery Procedures

### 1. Recovery Orchestrator

```javascript
// services/recovery-orchestrator.js
class RecoveryOrchestrator {
    async recover(options) {
        const {
            scenario,
            backupId,
            targetEnvironment = "production",
            verifyOnly = false,
        } = options;

        console.log("🚑 Starting recovery process");
        console.log(`Scenario: ${scenario}`);
        console.log(`Target: ${targetEnvironment}`);

        const recovery = {
            id: `recovery-${Date.now()}`,
            scenario,
            startTime: new Date(),
            steps: [],
        };

        try {
            // 1. Prepare recovery environment
            await this.prepareEnvironment(targetEnvironment);
            recovery.steps.push({ step: "prepare", status: "completed" });

            // 2. Restore data
            await this.restoreData(backupId, targetEnvironment);
            recovery.steps.push({ step: "restore-data", status: "completed" });

            // 3. Restore secrets
            await this.restoreSecrets(backupId, targetEnvironment);
            recovery.steps.push({
                step: "restore-secrets",
                status: "completed",
            });

            // 4. Restore services
            await this.restoreServices(targetEnvironment);
            recovery.steps.push({
                step: "restore-services",
                status: "completed",
            });

            // 5. Verify recovery
            await this.verifyRecovery(targetEnvironment);
            recovery.steps.push({ step: "verify", status: "completed" });

            // 6. Switch traffic (if not verify only)
            if (!verifyOnly) {
                await this.switchTraffic(targetEnvironment);
                recovery.steps.push({
                    step: "switch-traffic",
                    status: "completed",
                });
            }

            recovery.endTime = new Date();
            recovery.status = "completed";
            recovery.duration = recovery.endTime - recovery.startTime;

            console.log("✅ Recovery completed successfully");
            console.log(`Duration: ${recovery.duration / 1000}s`);

            return recovery;
        } catch (error) {
            recovery.status = "failed";
            recovery.error = error.message;

            console.error("❌ Recovery failed:", error);

            // Attempt to rollback
            await this.rollbackRecovery(recovery);

            throw error;
        }
    }

    async restoreData(backupId, environment) {
        console.log("Restoring data from backup...");

        const backup = await this.getBackup(backupId);

        // Restore database
        await this.restoreDatabase(backup.database, environment);

        // Restore files
        await this.restoreFiles(backup.files, environment);

        // Verify data integrity
        await this.verifyDataIntegrity(environment);
    }

    async restoreSecrets(backupId, environment) {
        console.log("Restoring secrets...");

        const backup = await this.getSecretBackup(backupId);
        const decrypted = await this.decryptBackup(backup);

        const dotenv = new DotEnvClient();

        for (const [project, envSecrets] of Object.entries(decrypted)) {
            for (const [env, secrets] of Object.entries(envSecrets)) {
                if (env === environment || environment === "all") {
                    await dotenv.importSecrets(project, env, secrets);
                }
            }
        }

        // Rotate critical secrets if compromise scenario
        if (this.scenario === "secret-compromise") {
            await this.rotateCompromisedSecrets();
        }
    }
}
```

## Best Practices

### 1. DR Checklist

```yaml
# disaster-recovery-checklist.yaml
preparation:
    documentation:
        - Recovery procedures documented
        - Contact information updated
        - Escalation paths defined
        - Runbooks tested

    backups:
        - Automated backup system running
        - Backups tested regularly
        - Offsite backup storage
        - Encryption keys secured

    infrastructure:
        - DR environment maintained
        - Cross-region replication active
        - Failover mechanisms tested
        - Network routes configured

    team:
        - Team trained on procedures
        - On-call rotation active
        - Communication channels ready
        - Decision makers identified

during_incident:
    immediate_actions:
        - Assess situation
        - Activate incident response
        - Notify stakeholders
        - Begin diagnostics

    communication:
        - Create incident channel
        - Send status updates
        - Document timeline
        - Coordinate response

    recovery:
        - Execute runbooks
        - Monitor progress
        - Verify each step
        - Test functionality

post_incident:
    verification:
        - All services operational
        - Data integrity confirmed
        - Performance normal
        - Security verified

    documentation:
        - Timeline documented
        - Postmortem written
        - Lessons learned captured
        - Action items assigned
```

### 2. Testing Schedule

```javascript
// config/dr-testing-schedule.js
module.exports = {
    schedule: {
        daily: {
            tests: ["backup-verification", "replica-health"],
            time: "02:00",
        },

        weekly: {
            tests: ["restore-test", "failover-test"],
            day: "sunday",
            time: "03:00",
        },

        monthly: {
            tests: ["full-dr-test", "runbook-validation"],
            day: 1,
            time: "04:00",
        },

        quarterly: {
            tests: ["region-failover", "complete-recovery"],
            months: [3, 6, 9, 12],
            coordination: "required",
        },
    },

    testDurations: {
        "backup-verification": "10m",
        "replica-health": "5m",
        "restore-test": "30m",
        "failover-test": "45m",
        "full-dr-test": "2h",
        "region-failover": "4h",
    },
};
```

## Troubleshooting

### Common DR Issues

1. **Backup Corruption**

    ```javascript
    // Verify backup integrity
    const checksum = await calculateChecksum(backupFile);
    if (checksum !== expectedChecksum) {
        // Use previous backup
        await usePreviousBackup();
    }
    ```

2. **Slow Recovery**

    ```javascript
    // Parallel restoration
    await Promise.all([restoreDatabase(), restoreFiles(), restoreSecrets()]);
    ```

3. **Secret Recovery Failures**
    ```javascript
    // Fallback to encrypted cache
    const cache = await getEncryptedCache();
    await restoreFromCache(cache);
    ```

## Resources

- [AWS Disaster Recovery](https://aws.amazon.com/disaster-recovery/)
- [NIST Contingency Planning Guide](https://csrc.nist.gov/publications/detail/sp/800-34/rev-1/final)
- [Business Continuity Planning](https://www.ready.gov/business-continuity-plan)
- [DotEnv Backup Documentation](/documentation/v1/core-concepts/security-model)

## Next Steps

- [Create DR plan](#disaster-recovery-planning)
- [Set up backup system](#backup-procedures)
- [Configure incident response](#incident-response)
- [Test recovery procedures](#recovery-procedures)
