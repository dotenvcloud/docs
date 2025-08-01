---
title: Backup and Restore
slug: backup-restore
order: 2
tags: [administration, backup, restore, disaster-recovery]
---

# Backup and Restore

This guide covers backup and restore procedures for DotEnv, including data backup strategies, restoration processes, and disaster recovery planning. Learn how to protect your organization's data and ensure business continuity.

## Backup Strategy Overview

### Backup Architecture

```
┌─────────────────────────────────────────┐
│         Backup Sources                  │
│  • Database (PostgreSQL)               │
│  • File Storage (S3)                   │
│  • Configuration                       │
│  • Encryption Keys                     │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│         Backup Process                  │
│  • Incremental Backups                 │
│  • Full Backups                        │
│  • Encryption                          │
│  • Compression                         │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│         Storage Destinations            │
│  • Primary: AWS S3                     │
│  • Secondary: Azure Blob               │
│  • Tertiary: Offline Archive           │
│  • Geographic Distribution             │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│       Verification & Testing            │
│  • Integrity Checks                    │
│  • Restore Testing                     │
│  • Recovery Drills                     │
│  • Documentation                       │
└─────────────────────────────────────────┘
```

## Backup Configuration

### Backup Types

```javascript
// Backup configuration
const backupConfig = {
    database: {
        type: "postgresql",
        strategy: {
            full_backup: {
                frequency: "daily",
                retention: 30, // days
                time: "02:00 UTC",
            },
            incremental_backup: {
                frequency: "hourly",
                retention: 7, // days
            },
            point_in_time: {
                enabled: true,
                retention: 7, // days
            },
        },
    },

    files: {
        storage: "s3",
        strategy: {
            continuous: true,
            versioning: true,
            lifecycle: {
                transition_to_ia: 30, // days
                transition_to_glacier: 90, // days
                expire: 365, // days
            },
        },
    },

    encryption_keys: {
        backup_method: "secure_vault",
        key_escrow: true,
        recovery_shares: 5,
        recovery_threshold: 3,
    },
};
```

### Backup Schedule

```javascript
// Automated backup scheduler
class BackupScheduler {
    constructor() {
        this.schedules = {
            full_database: "0 2 * * *", // 2 AM daily
            incremental_database: "0 * * * *", // Every hour
            configuration: "0 0 * * 0", // Weekly
            encryption_keys: "0 0 1 * *", // Monthly
            audit_logs: "0 3 * * *", // 3 AM daily
        };
    }

    async scheduleBackups() {
        // Full database backup
        cron.schedule(this.schedules.full_database, async () => {
            await this.performFullDatabaseBackup();
        });

        // Incremental database backup
        cron.schedule(this.schedules.incremental_database, async () => {
            await this.performIncrementalBackup();
        });

        // Configuration backup
        cron.schedule(this.schedules.configuration, async () => {
            await this.backupConfiguration();
        });

        // Encryption keys backup
        cron.schedule(this.schedules.encryption_keys, async () => {
            await this.backupEncryptionKeys();
        });
    }

    async performFullDatabaseBackup() {
        const backup = {
            id: generateBackupId(),
            type: "full",
            started_at: new Date(),
            status: "in_progress",
        };

        try {
            // Create database dump
            const dump = await this.createDatabaseDump();

            // Compress dump
            const compressed = await this.compressBackup(dump);

            // Encrypt backup
            const encrypted = await this.encryptBackup(compressed);

            // Upload to storage
            const locations = await this.uploadBackup(encrypted);

            backup.completed_at = new Date();
            backup.status = "completed";
            backup.size = encrypted.size;
            backup.locations = locations;

            // Verify backup
            await this.verifyBackup(backup);
        } catch (error) {
            backup.status = "failed";
            backup.error = error.message;
            await this.notifyBackupFailure(backup);
        }

        await this.recordBackup(backup);

        return backup;
    }
}
```

## Database Backup

### PostgreSQL Backup

```bash
#!/bin/bash
# database-backup.sh

set -euo pipefail

# Configuration
DB_HOST="${DB_HOST}"
DB_NAME="${DB_NAME}"
DB_USER="${DB_USER}"
BACKUP_DIR="/backups/database"
S3_BUCKET="${BACKUP_S3_BUCKET}"
RETENTION_DAYS=30

# Create backup filename
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/dotenv_${DB_NAME}_${TIMESTAMP}.sql.gz"

# Perform backup
echo "Starting database backup..."
PGPASSWORD="${DB_PASSWORD}" pg_dump \
  -h "${DB_HOST}" \
  -U "${DB_USER}" \
  -d "${DB_NAME}" \
  --no-owner \
  --no-privileges \
  --clean \
  --if-exists \
  | gzip -9 > "${BACKUP_FILE}"

# Calculate checksum
CHECKSUM=$(sha256sum "${BACKUP_FILE}" | awk '{print $1}')
echo "${CHECKSUM}" > "${BACKUP_FILE}.sha256"

# Encrypt backup
echo "Encrypting backup..."
gpg --cipher-algo AES256 \
    --compress-algo zlib \
    --recipient backup@dotenv.cloud \
    --encrypt "${BACKUP_FILE}"

# Upload to S3
echo "Uploading to S3..."
aws s3 cp "${BACKUP_FILE}.gpg" "s3://${S3_BUCKET}/database/${TIMESTAMP}/" \
  --storage-class STANDARD_IA \
  --metadata "checksum=${CHECKSUM},timestamp=${TIMESTAMP}"

# Upload to secondary location
echo "Uploading to secondary storage..."
az storage blob upload \
  --container-name backups \
  --name "database/${TIMESTAMP}/backup.sql.gz.gpg" \
  --file "${BACKUP_FILE}.gpg"

# Clean up old backups
echo "Cleaning up old backups..."
find "${BACKUP_DIR}" -name "*.sql.gz*" -mtime +${RETENTION_DAYS} -delete

# Verify backup
echo "Verifying backup..."
aws s3api head-object \
  --bucket "${S3_BUCKET}" \
  --key "database/${TIMESTAMP}/${BACKUP_FILE}.gpg" \
  --query "Metadata.checksum" \
  --output text | grep -q "${CHECKSUM}"

echo "Backup completed successfully"
```

### Point-in-Time Recovery

```javascript
// PITR configuration
class PointInTimeRecovery {
    constructor() {
        this.config = {
            wal_level: "replica",
            archive_mode: "on",
            archive_command: "aws s3 cp %p s3://backup-bucket/wal/%f",
            max_wal_senders: 3,
            wal_keep_segments: 64,
        };
    }

    async configureWALArchiving() {
        // PostgreSQL configuration
        const pgConfig = `
      # Write Ahead Log settings
      wal_level = ${this.config.wal_level}
      archive_mode = ${this.config.archive_mode}
      archive_command = '${this.config.archive_command}'
      archive_timeout = 300
      
      # Replication settings
      max_wal_senders = ${this.config.max_wal_senders}
      wal_keep_segments = ${this.config.wal_keep_segments}
      
      # Checkpoint settings
      checkpoint_timeout = 15min
      checkpoint_completion_target = 0.9
    `;

        await this.updatePostgreSQLConfig(pgConfig);
    }

    async restoreToPointInTime(targetTime) {
        const recovery = {
            target_time: targetTime,
            base_backup: await this.findNearestBaseBackup(targetTime),
            wal_files: await this.getRequiredWALFiles(targetTime),
        };

        // Stop database
        await this.stopDatabase();

        // Restore base backup
        await this.restoreBaseBackup(recovery.base_backup);

        // Create recovery configuration
        const recoveryConf = `
      restore_command = 'aws s3 cp s3://backup-bucket/wal/%f %p'
      recovery_target_time = '${targetTime}'
      recovery_target_action = 'promote'
    `;

        await this.writeRecoveryConfig(recoveryConf);

        // Start recovery
        await this.startDatabase();

        // Wait for recovery to complete
        await this.waitForRecovery();

        return recovery;
    }
}
```

## File Storage Backup

### S3 Backup Strategy

```javascript
// S3 backup implementation
class S3BackupManager {
    constructor() {
        this.config = {
            versioning: true,
            lifecycle_rules: [
                {
                    id: "transition-to-ia",
                    status: "Enabled",
                    transitions: [
                        {
                            days: 30,
                            storage_class: "STANDARD_IA",
                        },
                    ],
                },
                {
                    id: "transition-to-glacier",
                    status: "Enabled",
                    transitions: [
                        {
                            days: 90,
                            storage_class: "GLACIER",
                        },
                    ],
                },
                {
                    id: "expire-old-versions",
                    status: "Enabled",
                    noncurrent_version_expiration: {
                        days: 365,
                    },
                },
            ],
            replication: {
                role: "arn:aws:iam::123456789012:role/replication-role",
                rules: [
                    {
                        id: "replicate-all",
                        status: "Enabled",
                        priority: 1,
                        destination: {
                            bucket: "arn:aws:s3:::backup-replica-bucket",
                            storage_class: "STANDARD_IA",
                        },
                    },
                ],
            },
        };
    }

    async configureBucketBackup(bucketName) {
        // Enable versioning
        await s3
            .putBucketVersioning({
                Bucket: bucketName,
                VersioningConfiguration: {
                    Status: "Enabled",
                },
            })
            .promise();

        // Configure lifecycle rules
        await s3
            .putBucketLifecycleConfiguration({
                Bucket: bucketName,
                LifecycleConfiguration: {
                    Rules: this.config.lifecycle_rules,
                },
            })
            .promise();

        // Configure replication
        await s3
            .putBucketReplication({
                Bucket: bucketName,
                ReplicationConfiguration: {
                    Role: this.config.replication.role,
                    Rules: this.config.replication.rules,
                },
            })
            .promise();

        // Enable object lock for compliance
        await s3
            .putObjectLockConfiguration({
                Bucket: bucketName,
                ObjectLockConfiguration: {
                    ObjectLockEnabled: "Enabled",
                    Rule: {
                        DefaultRetention: {
                            Mode: "COMPLIANCE",
                            Days: 7,
                        },
                    },
                },
            })
            .promise();
    }

    async createSnapshot(bucketName) {
        const snapshot = {
            id: generateSnapshotId(),
            bucket: bucketName,
            timestamp: new Date(),
            objects: [],
        };

        // List all objects
        let continuationToken;
        do {
            const response = await s3
                .listObjectsV2({
                    Bucket: bucketName,
                    ContinuationToken: continuationToken,
                })
                .promise();

            snapshot.objects.push(...response.Contents);
            continuationToken = response.NextContinuationToken;
        } while (continuationToken);

        // Save snapshot metadata
        await this.saveSnapshotMetadata(snapshot);

        return snapshot;
    }
}
```

## Encryption Key Backup

### Key Escrow System

```javascript
// Secure key backup system
class KeyEscrowSystem {
    constructor() {
        this.config = {
            shares: 5,
            threshold: 3,
            key_custodians: [
                { name: "CTO", email: "cto@company.com" },
                { name: "Security Officer", email: "security@company.com" },
                { name: "VP Engineering", email: "vp-eng@company.com" },
                { name: "Legal Counsel", email: "legal@company.com" },
                { name: "CEO", email: "ceo@company.com" },
            ],
        };
    }

    async backupMasterKey(masterKey) {
        // Split key using Shamir's Secret Sharing
        const shares = await this.splitKey(masterKey, {
            shares: this.config.shares,
            threshold: this.config.threshold,
        });

        // Distribute shares to custodians
        for (let i = 0; i < shares.length; i++) {
            const custodian = this.config.key_custodians[i];
            const encryptedShare = await this.encryptShareForCustodian(
                shares[i],
                custodian,
            );

            // Store encrypted share
            await this.storeKeyShare({
                custodian_id: custodian.email,
                share_index: i,
                encrypted_share: encryptedShare,
                created_at: new Date(),
            });

            // Notify custodian
            await this.notifyCustodian(custodian, {
                action: "key_share_assigned",
                instructions: this.getRecoveryInstructions(),
            });
        }

        // Create backup verification
        const verification = await this.createVerificationData(masterKey);
        await this.storeVerification(verification);

        return {
            shares_created: shares.length,
            threshold: this.config.threshold,
            custodians: this.config.key_custodians.map((c) => c.email),
            verification_hash: verification.hash,
        };
    }

    async recoverMasterKey(shares) {
        if (shares.length < this.config.threshold) {
            throw new Error(`Need at least ${this.config.threshold} shares`);
        }

        // Verify shares
        for (const share of shares) {
            await this.verifyShare(share);
        }

        // Reconstruct key
        const masterKey = await this.reconstructKey(shares);

        // Verify recovered key
        const verification = await this.getVerification();
        if (!(await this.verifyKey(masterKey, verification))) {
            throw new Error("Recovered key verification failed");
        }

        // Audit recovery
        await this.auditKeyRecovery({
            recovered_by: shares.map((s) => s.custodian_id),
            recovered_at: new Date(),
            purpose: shares[0].recovery_reason,
        });

        return masterKey;
    }
}
```

### Encryption Key Rotation Backup

```javascript
// Key rotation backup process
class KeyRotationBackup {
    async backupBeforeRotation(project) {
        const backup = {
            project_id: project.id,
            timestamp: new Date(),
            keys: {
                current: await this.backupCurrentKey(project),
                previous: await this.backupPreviousKeys(project),
            },
            secrets: await this.backupSecretHashes(project),
        };

        // Create encrypted backup
        const encryptedBackup = await this.encryptBackup(backup);

        // Store in multiple locations
        const locations = await Promise.all([
            this.storeInVault(encryptedBackup),
            this.storeInS3(encryptedBackup),
            this.storeInColdStorage(encryptedBackup),
        ]);

        // Create recovery package
        const recoveryPackage = {
            backup_id: backup.id,
            locations: locations,
            recovery_instructions: this.generateRecoveryInstructions(backup),
            verification_data: await this.createVerificationData(backup),
        };

        return recoveryPackage;
    }

    generateRecoveryInstructions(backup) {
        return {
            steps: [
                "Retrieve backup from any storage location",
                "Decrypt using master recovery key",
                "Verify backup integrity using checksums",
                "Restore keys in reverse chronological order",
                "Re-encrypt secrets with restored keys",
                "Verify all secrets are accessible",
            ],
            required_tools: [
                "Master recovery key",
                "Backup decryption tool",
                "Key restoration utility",
            ],
            estimated_time: "2-4 hours",
            contacts: {
                primary: "security-team@company.com",
                emergency: "cto@company.com",
            },
        };
    }
}
```

## Restoration Procedures

### Database Restoration

```bash
#!/bin/bash
# database-restore.sh

set -euo pipefail

# Configuration
BACKUP_FILE="${1}"
DB_HOST="${DB_HOST}"
DB_NAME="${DB_NAME}_restore"
DB_USER="${DB_USER}"

echo "Starting database restoration..."

# Verify backup file
if [ ! -f "${BACKUP_FILE}" ]; then
    echo "Backup file not found: ${BACKUP_FILE}"
    exit 1
fi

# Verify checksum
CHECKSUM_FILE="${BACKUP_FILE}.sha256"
if [ -f "${CHECKSUM_FILE}" ]; then
    echo "Verifying backup integrity..."
    sha256sum -c "${CHECKSUM_FILE}" || exit 1
fi

# Decrypt backup
echo "Decrypting backup..."
gpg --decrypt "${BACKUP_FILE}.gpg" > "${BACKUP_FILE}.decrypted"

# Create restore database
echo "Creating restore database..."
PGPASSWORD="${DB_PASSWORD}" createdb \
    -h "${DB_HOST}" \
    -U "${DB_USER}" \
    "${DB_NAME}"

# Restore backup
echo "Restoring database..."
gunzip -c "${BACKUP_FILE}.decrypted" | \
    PGPASSWORD="${DB_PASSWORD}" psql \
    -h "${DB_HOST}" \
    -U "${DB_USER}" \
    -d "${DB_NAME}" \
    --single-transaction \
    --set ON_ERROR_STOP=on

# Verify restoration
echo "Verifying restoration..."
TABLE_COUNT=$(PGPASSWORD="${DB_PASSWORD}" psql \
    -h "${DB_HOST}" \
    -U "${DB_USER}" \
    -d "${DB_NAME}" \
    -t -c "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'public'")

echo "Restored ${TABLE_COUNT} tables"

# Run integrity checks
echo "Running integrity checks..."
PGPASSWORD="${DB_PASSWORD}" psql \
    -h "${DB_HOST}" \
    -U "${DB_USER}" \
    -d "${DB_NAME}" \
    -f /scripts/integrity-checks.sql

echo "Restoration completed successfully"
```

### Application Restoration

```javascript
// Application restoration manager
class ApplicationRestorer {
    async restoreApplication(backupId) {
        const restoration = {
            id: generateRestorationId(),
            backup_id: backupId,
            started_at: new Date(),
            steps: [],
        };

        try {
            // Step 1: Retrieve backup
            restoration.steps.push(await this.retrieveBackup(backupId));

            // Step 2: Verify backup integrity
            restoration.steps.push(await this.verifyBackupIntegrity(backupId));

            // Step 3: Prepare restoration environment
            restoration.steps.push(await this.prepareEnvironment());

            // Step 4: Restore database
            restoration.steps.push(await this.restoreDatabase(backupId));

            // Step 5: Restore files
            restoration.steps.push(await this.restoreFiles(backupId));

            // Step 6: Restore configuration
            restoration.steps.push(await this.restoreConfiguration(backupId));

            // Step 7: Restore encryption keys
            restoration.steps.push(await this.restoreEncryptionKeys(backupId));

            // Step 8: Verify restoration
            restoration.steps.push(await this.verifyRestoration());

            // Step 9: Switch traffic
            restoration.steps.push(await this.switchTraffic());

            restoration.completed_at = new Date();
            restoration.status = "completed";
        } catch (error) {
            restoration.status = "failed";
            restoration.error = error.message;

            // Rollback
            await this.rollbackRestoration(restoration);
        }

        await this.recordRestoration(restoration);

        return restoration;
    }

    async verifyRestoration() {
        const checks = {
            database: await this.checkDatabaseIntegrity(),
            files: await this.checkFileIntegrity(),
            encryption: await this.checkEncryptionKeys(),
            api: await this.checkAPIHealth(),
            data_integrity: await this.checkDataIntegrity(),
        };

        const failed = Object.entries(checks)
            .filter(([_, result]) => !result.passed)
            .map(([check, _]) => check);

        if (failed.length > 0) {
            throw new Error(`Verification failed for: ${failed.join(", ")}`);
        }

        return {
            step: "verify_restoration",
            status: "completed",
            checks: checks,
        };
    }
}
```

## Disaster Recovery

### DR Planning

```javascript
// Disaster recovery plan
const disasterRecoveryPlan = {
    objectives: {
        rto: 4, // Recovery Time Objective: 4 hours
        rpo: 1, // Recovery Point Objective: 1 hour
        mtd: 24, // Maximum Tolerable Downtime: 24 hours
    },

    scenarios: {
        data_center_failure: {
            probability: "low",
            impact: "critical",
            response: "failover_to_secondary_region",
        },

        database_corruption: {
            probability: "medium",
            impact: "high",
            response: "restore_from_backup",
        },

        ransomware_attack: {
            probability: "medium",
            impact: "critical",
            response: "isolate_and_restore",
        },

        key_compromise: {
            probability: "low",
            impact: "critical",
            response: "rotate_all_keys",
        },
    },

    procedures: {
        initial_response: [
            "Activate incident response team",
            "Assess scope of disaster",
            "Initiate communication plan",
            "Begin recovery procedures",
        ],

        recovery_priorities: [
            "Restore authentication services",
            "Restore database access",
            "Restore API services",
            "Restore web interface",
            "Restore background jobs",
        ],

        communication_plan: {
            internal: ["CTO", "Security Team", "Operations"],
            external: ["Status Page", "Email", "Support Tickets"],
            stakeholders: ["Customers", "Partners", "Regulators"],
        },
    },
};
```

### Automated DR Testing

```javascript
// DR testing automation
class DisasterRecoveryTester {
    async runDRTest(scenario) {
        const test = {
            id: generateTestId(),
            scenario: scenario,
            started_at: new Date(),
            results: [],
        };

        try {
            // Prepare test environment
            const testEnv = await this.prepareTestEnvironment();

            // Simulate disaster
            await this.simulateDisaster(scenario, testEnv);

            // Execute recovery
            const recoveryStart = new Date();
            await this.executeRecoveryPlan(scenario, testEnv);
            const recoveryEnd = new Date();

            // Verify recovery
            const verification = await this.verifyRecovery(testEnv);

            test.results = {
                recovery_time: recoveryEnd - recoveryStart,
                rto_met: recoveryEnd - recoveryStart < scenario.rto * 3600000,
                data_loss: verification.data_loss,
                rpo_met: verification.data_loss < scenario.rpo * 3600000,
                functionality_restored: verification.functionality_check,
                issues_found: verification.issues,
            };

            test.status = "completed";
        } catch (error) {
            test.status = "failed";
            test.error = error.message;
        } finally {
            // Clean up test environment
            await this.cleanupTestEnvironment(testEnv);
        }

        test.completed_at = new Date();
        await this.recordDRTest(test);

        return test;
    }

    async simulateDisaster(scenario, environment) {
        switch (scenario.type) {
            case "database_failure":
                await this.stopDatabase(environment);
                break;

            case "region_failure":
                await this.blockRegionAccess(environment);
                break;

            case "corruption":
                await this.corruptTestData(environment);
                break;

            case "key_loss":
                await this.removeEncryptionKeys(environment);
                break;
        }
    }
}
```

## Backup Monitoring

### Backup Health Dashboard

```javascript
// Backup monitoring system
class BackupMonitor {
    async getBackupHealth() {
        const health = {
            overall_status: "healthy",
            last_check: new Date(),
            components: {},
        };

        // Check database backups
        health.components.database = await this.checkDatabaseBackups();

        // Check file backups
        health.components.files = await this.checkFileBackups();

        // Check encryption key backups
        health.components.encryption_keys = await this.checkKeyBackups();

        // Check backup storage
        health.components.storage = await this.checkBackupStorage();

        // Calculate overall status
        const unhealthy = Object.values(health.components).filter(
            (c) => c.status !== "healthy",
        );

        if (unhealthy.length > 0) {
            health.overall_status = unhealthy.some(
                (c) => c.status === "critical",
            )
                ? "critical"
                : "warning";
        }

        return health;
    }

    async checkDatabaseBackups() {
        const check = {
            status: "healthy",
            metrics: {},
        };

        // Check last successful backup
        const lastBackup = await this.getLastSuccessfulBackup("database");
        const hoursSinceBackup =
            (Date.now() - lastBackup.completed_at) / 3600000;

        check.metrics.last_backup_age = hoursSinceBackup;
        check.metrics.backup_size = lastBackup.size;

        if (hoursSinceBackup > 24) {
            check.status = "critical";
            check.message = "No successful backup in 24 hours";
        } else if (hoursSinceBackup > 12) {
            check.status = "warning";
            check.message = "No successful backup in 12 hours";
        }

        // Check backup success rate
        const successRate = await this.calculateBackupSuccessRate(
            "database",
            7,
        );
        check.metrics.success_rate = successRate;

        if (successRate < 0.95) {
            check.status = "warning";
            check.message = "Backup success rate below 95%";
        }

        return check;
    }
}
```

### Backup Alerts

```javascript
// Backup alerting configuration
const backupAlerts = {
    rules: [
        {
            name: "backup_failure",
            condition: 'backup.status === "failed"',
            severity: "critical",
            notification: {
                channels: ["email", "slack", "pagerduty"],
                message: "Backup failed: {{backup.type}} - {{backup.error}}",
            },
        },
        {
            name: "backup_age",
            condition: "hours_since_last_backup > 24",
            severity: "high",
            notification: {
                channels: ["email", "slack"],
                message: "No successful backup in {{hours}} hours",
            },
        },
        {
            name: "storage_space",
            condition: "backup_storage_usage > 0.9",
            severity: "warning",
            notification: {
                channels: ["email"],
                message: "Backup storage at {{usage_percent}}% capacity",
            },
        },
        {
            name: "verification_failure",
            condition: "backup.verification_failed === true",
            severity: "high",
            notification: {
                channels: ["email", "slack"],
                message: "Backup verification failed for {{backup.id}}",
            },
        },
    ],
};
```

## Best Practices

### 1. 3-2-1 Backup Rule

```javascript
// ✅ Good: Following 3-2-1 rule
const backupStrategy = {
    copies: 3, // Three copies of data
    media_types: 2, // Two different storage types
    offsite: 1, // One offsite backup

    implementation: {
        primary: "AWS S3",
        secondary: "Azure Blob Storage",
        tertiary: "Offline tape archive",
    },
};

// ❌ Bad: Single backup location
const backupStrategy = {
    copies: 1,
    location: "same_datacenter",
};
```

### 2. Regular Testing

```javascript
// ✅ Good: Automated restore testing
schedule.every("month").do(async () => {
    const testResult = await drTester.runDRTest({
        type: "full_restore",
        environment: "test",
    });

    if (!testResult.success) {
        await notifyOpsTeam(testResult);
    }
});

// ❌ Bad: Never testing restores
// "Schrodinger's backup" - unknown if it works
```

### 3. Encryption and Security

```javascript
// ✅ Good: Encrypted backups
const backup = await createBackup(data);
const encrypted = await encrypt(backup, {
    algorithm: "AES-256-GCM",
    key: backupEncryptionKey,
});
await store(encrypted);

// ❌ Bad: Unencrypted backups
await store(data); // Plain text backup
```

## Resources

- [NIST Contingency Planning Guide](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-34r1.pdf)
- [AWS Backup Best Practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/backup-recovery/welcome.html)
- [PostgreSQL Backup Documentation](https://www.postgresql.org/docs/current/backup.html)
- [DotEnv Backup API](/documentation/v1/api/backup)

## Next Steps

- [Configure disaster recovery](/documentation/v1/use-cases/disaster-recovery)
- [Set up monitoring](/documentation/v1/administration/monitoring)
- [Plan capacity scaling](/documentation/v1/administration/scaling)
- [Review security compliance](/documentation/v1/security-compliance/compliance)
