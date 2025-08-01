---
title: Migration Guide
slug: migration-guide
order: 10
tags: [use-cases, migration, upgrade, transition]
---

# Migration Guide

Learn how to migrate from other secret management solutions to DotEnv, or upgrade from older versions. This guide covers migration strategies, tools, and best practices for a smooth transition.

## Overview

Migration considerations:

- Minimal downtime
- Data integrity
- Gradual rollout
- Rollback capability
- Team training
- Security preservation

## Migration Strategies

### 1. Big Bang Migration

Complete migration in a single operation:

```bash
#!/bin/bash
# scripts/big-bang-migration.sh

echo "🚀 Starting DotEnv migration..."

# Step 1: Export existing secrets
case "$SOURCE_SYSTEM" in
  "vault")
    vault kv get -format=json secret/myapp > secrets-backup.json
    ;;
  "aws-secrets-manager")
    aws secretsmanager get-secret-value --secret-id myapp \
      --query SecretString --output text > secrets-backup.json
    ;;
  "azure-keyvault")
    az keyvault secret list --vault-name myvault \
      --query "[].{name:name,value:value}" > secrets-backup.json
    ;;
  "env-files")
    env-to-json < .env > secrets-backup.json
    ;;
esac

# Step 2: Transform to DotEnv format
node scripts/transform-secrets.js secrets-backup.json > dotenv-import.json

# Step 3: Import to DotEnv
dotenv import --project myapp --environment production \
  --file dotenv-import.json

# Step 4: Verify migration
dotenv pull production --project myapp --format json > migrated-secrets.json
node scripts/verify-migration.js secrets-backup.json migrated-secrets.json

echo "✅ Migration complete!"
```

### 2. Gradual Migration

Migrate incrementally with dual-source support:

```javascript
// services/secret-manager.js
class HybridSecretManager {
    constructor(config) {
        this.dotenv = new DotEnvClient(config.dotenv);
        this.legacy = this.initLegacyClient(config.legacy);
        this.migrationMode = config.migrationMode || "read-legacy-first";
    }

    async getSecret(key) {
        switch (this.migrationMode) {
            case "read-legacy-first":
                // Try legacy first, fall back to DotEnv
                try {
                    return await this.legacy.get(key);
                } catch (error) {
                    console.log(`Legacy miss for ${key}, trying DotEnv`);
                    return await this.dotenv.get(key);
                }

            case "read-dotenv-first":
                // Try DotEnv first, fall back to legacy
                try {
                    return await this.dotenv.get(key);
                } catch (error) {
                    console.log(`DotEnv miss for ${key}, trying legacy`);
                    return await this.legacy.get(key);
                }

            case "write-through":
                // Read from legacy, write to both
                const value = await this.legacy.get(key);
                await this.dotenv.set(key, value);
                return value;

            case "dotenv-only":
                // Full migration complete
                return await this.dotenv.get(key);

            default:
                throw new Error(
                    `Unknown migration mode: ${this.migrationMode}`,
                );
        }
    }

    async setSecret(key, value) {
        if (this.migrationMode === "write-through") {
            // Write to both systems during migration
            await Promise.all([
                this.legacy.set(key, value),
                this.dotenv.set(key, value),
            ]);
        } else {
            await this.dotenv.set(key, value);
        }
    }
}
```

## From Popular Solutions

### 1. From HashiCorp Vault

```javascript
// migration/from-vault.js
const vault = require("node-vault");
const { DotEnvClient } = require("@dotenv/sdk");

class VaultToDotEnvMigrator {
    constructor(config) {
        this.vault = vault({
            endpoint: config.vaultEndpoint,
            token: config.vaultToken,
        });
        this.dotenv = new DotEnvClient({
            apiKey: config.dotenvApiKey,
        });
    }

    async migrate(vaultPath, dotenvProject, dotenvEnvironment) {
        console.log(`Migrating from Vault path: ${vaultPath}`);

        // Read all secrets from Vault
        const vaultSecrets = await this.readVaultSecrets(vaultPath);

        // Transform Vault format to DotEnv format
        const dotenvSecrets = this.transformSecrets(vaultSecrets);

        // Import to DotEnv
        await this.importToDotEnv(
            dotenvSecrets,
            dotenvProject,
            dotenvEnvironment,
        );

        // Verify migration
        await this.verifyMigration(
            vaultSecrets,
            dotenvProject,
            dotenvEnvironment,
        );

        console.log("✅ Migration complete");
    }

    async readVaultSecrets(path) {
        const secrets = {};

        // List all secrets at path
        const list = await this.vault.list(path);

        for (const key of list.data.keys) {
            if (key.endsWith("/")) {
                // Recursive read for nested paths
                const nestedSecrets = await this.readVaultSecrets(
                    `${path}/${key}`,
                );
                Object.assign(secrets, nestedSecrets);
            } else {
                // Read individual secret
                const secret = await this.vault.read(`${path}/${key}`);
                secrets[key] = secret.data;
            }
        }

        return secrets;
    }

    transformSecrets(vaultSecrets) {
        const transformed = {};

        for (const [key, data] of Object.entries(vaultSecrets)) {
            // Flatten nested Vault structure
            if (typeof data === "object" && data.data) {
                for (const [subKey, value] of Object.entries(data.data)) {
                    transformed[`${key}_${subKey}`.toUpperCase()] = value;
                }
            } else {
                transformed[key.toUpperCase()] = data;
            }
        }

        return transformed;
    }

    async importToDotEnv(secrets, project, environment) {
        const batch = [];

        for (const [key, value] of Object.entries(secrets)) {
            batch.push({
                key,
                value: String(value),
                encrypted: this.shouldEncrypt(key),
            });

            // Import in batches of 50
            if (batch.length >= 50) {
                await this.dotenv.secrets.createBatch(
                    project,
                    environment,
                    batch,
                );
                batch.length = 0;
            }
        }

        // Import remaining secrets
        if (batch.length > 0) {
            await this.dotenv.secrets.createBatch(project, environment, batch);
        }
    }

    shouldEncrypt(key) {
        const encryptPatterns = [
            "PASSWORD",
            "SECRET",
            "KEY",
            "TOKEN",
            "PRIVATE",
        ];
        return encryptPatterns.some((pattern) => key.includes(pattern));
    }
}
```

### 2. From AWS Secrets Manager

```python
# migration/from_aws_secrets.py
import boto3
import json
import requests
from typing import Dict, List

class AWSSecretsToDotEnvMigrator:
    def __init__(self, aws_region: str, dotenv_api_key: str):
        self.secrets_client = boto3.client(
            'secretsmanager',
            region_name=aws_region
        )
        self.dotenv_api_key = dotenv_api_key
        self.dotenv_api_url = "https://api.dotenv.cloud/v1"

    def migrate(self, secret_names: List[str], project: str, environment: str):
        """Migrate AWS Secrets to DotEnv"""
        print(f"Migrating {len(secret_names)} secrets to DotEnv")

        # Collect all secrets
        all_secrets = {}
        for secret_name in secret_names:
            secrets = self.read_aws_secret(secret_name)
            all_secrets.update(secrets)

        # Import to DotEnv
        self.import_to_dotenv(all_secrets, project, environment)

        # Generate migration report
        self.generate_report(all_secrets, project, environment)

    def read_aws_secret(self, secret_name: str) -> Dict[str, str]:
        """Read secret from AWS Secrets Manager"""
        try:
            response = self.secrets_client.get_secret_value(
                SecretId=secret_name
            )

            # Parse JSON secrets
            if 'SecretString' in response:
                secret_data = json.loads(response['SecretString'])
                if isinstance(secret_data, dict):
                    return secret_data
                else:
                    return {secret_name: secret_data}
            else:
                # Binary secret
                return {secret_name: response['SecretBinary'].decode('utf-8')}

        except Exception as e:
            print(f"Error reading secret {secret_name}: {e}")
            return {}

    def import_to_dotenv(self, secrets: Dict[str, str],
                        project: str, environment: str):
        """Import secrets to DotEnv"""
        headers = {
            'Authorization': f'Bearer {self.dotenv_api_key}',
            'Content-Type': 'application/json'
        }

        # Prepare batch import
        batch = []
        for key, value in secrets.items():
            batch.append({
                'key': key.upper().replace('-', '_'),
                'value': str(value),
                'encrypted': self._should_encrypt(key)
            })

        # Import secrets
        response = requests.post(
            f'{self.dotenv_api_url}/projects/{project}/'
            f'environments/{environment}/secrets/batch',
            headers=headers,
            json={'secrets': batch}
        )

        if response.status_code == 200:
            print(f"✅ Imported {len(batch)} secrets")
        else:
            print(f"❌ Import failed: {response.text}")

    def _should_encrypt(self, key: str) -> bool:
        """Determine if secret should be encrypted"""
        sensitive_patterns = [
            'password', 'secret', 'key', 'token',
            'certificate', 'private'
        ]
        key_lower = key.lower()
        return any(pattern in key_lower for pattern in sensitive_patterns)
```

### 3. From .env Files

```javascript
// migration/from-env-files.js
const fs = require("fs").promises;
const path = require("path");
const dotenv = require("dotenv");

class EnvFileMigrator {
    constructor(dotenvClient) {
        this.dotenvClient = dotenvClient;
    }

    async migrateEnvFile(envFilePath, project, environment) {
        console.log(`Migrating ${envFilePath} to DotEnv`);

        // Parse .env file
        const content = await fs.readFile(envFilePath, "utf8");
        const parsed = dotenv.parse(content);

        // Analyze and categorize secrets
        const categorized = this.categorizeSecrets(parsed);

        // Import to DotEnv with metadata
        await this.importWithMetadata(categorized, project, environment);

        // Create migration documentation
        await this.createMigrationDocs(envFilePath, categorized);
    }

    categorizeSecrets(secrets) {
        const categories = {
            database: [],
            api_keys: [],
            urls: [],
            features: [],
            security: [],
            other: [],
        };

        for (const [key, value] of Object.entries(secrets)) {
            if (key.includes("DATABASE") || key.includes("DB_")) {
                categories.database.push({ key, value });
            } else if (key.includes("API") || key.includes("KEY")) {
                categories.api_keys.push({ key, value });
            } else if (key.includes("URL") || key.includes("URI")) {
                categories.urls.push({ key, value });
            } else if (key.includes("FEATURE") || key.includes("ENABLE")) {
                categories.features.push({ key, value });
            } else if (key.includes("SECRET") || key.includes("TOKEN")) {
                categories.security.push({ key, value });
            } else {
                categories.other.push({ key, value });
            }
        }

        return categories;
    }

    async importWithMetadata(categorized, project, environment) {
        const allSecrets = [];

        for (const [category, secrets] of Object.entries(categorized)) {
            for (const { key, value } of secrets) {
                allSecrets.push({
                    key,
                    value,
                    category,
                    encrypted: this.shouldEncrypt(key, category),
                    metadata: {
                        migrated_from: "env_file",
                        migration_date: new Date().toISOString(),
                        original_category: category,
                    },
                });
            }
        }

        // Import in batches
        const batchSize = 50;
        for (let i = 0; i < allSecrets.length; i += batchSize) {
            const batch = allSecrets.slice(i, i + batchSize);
            await this.dotenvClient.secrets.createBatch(
                project,
                environment,
                batch,
            );
        }
    }

    shouldEncrypt(key, category) {
        // Always encrypt security category
        if (category === "security") return true;

        // Check for sensitive patterns
        const sensitivePatterns = [
            "PASSWORD",
            "SECRET",
            "PRIVATE",
            "TOKEN",
            "KEY",
        ];

        return sensitivePatterns.some((pattern) =>
            key.toUpperCase().includes(pattern),
        );
    }
}
```

## Version Upgrades

### 1. DotEnv v1 to v2

```javascript
// migration/v1-to-v2.js
class DotEnvV1ToV2Migrator {
    async migrate(v1Config, v2Config) {
        console.log("Starting DotEnv v1 to v2 migration...");

        // Step 1: Export v1 data
        const v1Data = await this.exportV1Data(v1Config);

        // Step 2: Transform to v2 format
        const v2Data = this.transformToV2Format(v1Data);

        // Step 3: Import to v2
        await this.importToV2(v2Data, v2Config);

        // Step 4: Verify migration
        await this.verifyMigration(v1Data, v2Config);
    }

    async exportV1Data(v1Config) {
        const data = {
            organizations: [],
            projects: [],
            environments: [],
            secrets: [],
        };

        // Export organizations
        const orgs = await this.v1Client.organizations.list();
        for (const org of orgs) {
            data.organizations.push(org);

            // Export projects
            const projects = await this.v1Client.projects.list(org.id);
            for (const project of projects) {
                data.projects.push(project);

                // Export environments and secrets
                const environments = await this.v1Client.environments.list(
                    org.id,
                    project.id,
                );

                for (const env of environments) {
                    data.environments.push(env);

                    const secrets = await this.v1Client.secrets.list(
                        org.id,
                        project.id,
                        env.id,
                    );

                    data.secrets.push(
                        ...secrets.map((s) => ({
                            ...s,
                            organization_id: org.id,
                            project_id: project.id,
                            environment_id: env.id,
                        })),
                    );
                }
            }
        }

        return data;
    }

    transformToV2Format(v1Data) {
        // V2 changes:
        // - Flattened organization structure
        // - New permission model
        // - Enhanced encryption options
        // - Improved audit logging

        return {
            organizations: v1Data.organizations.map((org) => ({
                ...org,
                settings: {
                    ...org.settings,
                    encryption: {
                        algorithm: "AES-256-GCM",
                        key_rotation_enabled: true,
                        key_rotation_period_days: 90,
                    },
                },
            })),

            projects: v1Data.projects.map((project) => ({
                ...project,
                settings: {
                    ...project.settings,
                    access_control: {
                        mode: "rbac",
                        default_permission: "read",
                    },
                },
            })),

            secrets: v1Data.secrets.map((secret) => ({
                ...secret,
                encryption: {
                    enabled: true,
                    algorithm: "AES-256-GCM",
                },
                audit: {
                    created_at: secret.created_at,
                    created_by: secret.created_by || "migration",
                    last_modified_at: secret.updated_at,
                    last_modified_by: secret.updated_by || "migration",
                },
            })),
        };
    }
}
```

## Migration Tools

### 1. Migration CLI

```bash
#!/bin/bash
# dotenv-migrate.sh

# DotEnv Migration Tool
VERSION="1.0.0"

function show_help() {
  cat << EOF
DotEnv Migration Tool v${VERSION}

Usage: dotenv-migrate [command] [options]

Commands:
  from-vault        Migrate from HashiCorp Vault
  from-aws          Migrate from AWS Secrets Manager
  from-azure        Migrate from Azure Key Vault
  from-env          Migrate from .env files
  from-k8s          Migrate from Kubernetes secrets
  validate          Validate migration
  rollback          Rollback migration

Options:
  --source          Source system configuration
  --target          Target DotEnv configuration
  --dry-run         Preview migration without changes
  --parallel        Number of parallel workers
  --verify          Verify after migration
  --backup          Create backup before migration

Examples:
  dotenv-migrate from-vault --source vault.json --target dotenv.json
  dotenv-migrate from-env --source .env.production --target production
  dotenv-migrate validate --config migration.json
EOF
}

function migrate_from_vault() {
  local source=$1
  local target=$2

  echo "Migrating from HashiCorp Vault..."

  # Export from Vault
  vault kv get -format=json secret/data > vault-export.json

  # Transform data
  jq '.data.data | to_entries | map({key: .key, value: .value})' \
    vault-export.json > transformed.json

  # Import to DotEnv
  dotenv import --file transformed.json \
    --project "$target" \
    --environment production

  echo "✅ Migration complete"
}

function migrate_from_env() {
  local source=$1
  local target=$2

  echo "Migrating from .env file: $source"

  # Parse .env file
  node -e "
    const fs = require('fs');
    const dotenv = require('dotenv');
    const parsed = dotenv.parse(fs.readFileSync('$source'));
    console.log(JSON.stringify(parsed, null, 2));
  " > env-export.json

  # Import to DotEnv
  dotenv import --file env-export.json \
    --project "$target" \
    --environment production

  echo "✅ Migration complete"
}

# Main script
case "$1" in
  from-vault)
    migrate_from_vault "$@"
    ;;
  from-env)
    migrate_from_env "$@"
    ;;
  *)
    show_help
    ;;
esac
```

### 2. Migration Validator

```javascript
// tools/migration-validator.js
class MigrationValidator {
    constructor(sourceData, targetClient) {
        this.sourceData = sourceData;
        this.targetClient = targetClient;
        this.errors = [];
        this.warnings = [];
    }

    async validate() {
        console.log("Validating migration...");

        // Check completeness
        await this.validateCompleteness();

        // Check data integrity
        await this.validateIntegrity();

        // Check functionality
        await this.validateFunctionality();

        // Generate report
        return this.generateReport();
    }

    async validateCompleteness() {
        const sourceKeys = Object.keys(this.sourceData);
        const targetSecrets = await this.targetClient.secrets.list();
        const targetKeys = targetSecrets.map((s) => s.key);

        // Check for missing secrets
        const missing = sourceKeys.filter((key) => !targetKeys.includes(key));
        if (missing.length > 0) {
            this.errors.push({
                type: "missing_secrets",
                message: `Missing ${missing.length} secrets`,
                details: missing,
            });
        }

        // Check for extra secrets
        const extra = targetKeys.filter((key) => !sourceKeys.includes(key));
        if (extra.length > 0) {
            this.warnings.push({
                type: "extra_secrets",
                message: `Found ${extra.length} extra secrets`,
                details: extra,
            });
        }
    }

    async validateIntegrity() {
        for (const [key, sourceValue] of Object.entries(this.sourceData)) {
            try {
                const targetValue = await this.targetClient.secrets.get(key);

                if (sourceValue !== targetValue) {
                    this.errors.push({
                        type: "value_mismatch",
                        message: `Value mismatch for ${key}`,
                        details: {
                            key,
                            source_length: sourceValue.length,
                            target_length: targetValue.length,
                        },
                    });
                }
            } catch (error) {
                this.errors.push({
                    type: "retrieval_error",
                    message: `Failed to retrieve ${key}`,
                    details: error.message,
                });
            }
        }
    }

    async validateFunctionality() {
        // Test basic operations
        const testKey = "_MIGRATION_TEST_" + Date.now();
        const testValue = "test_value_" + Math.random();

        try {
            // Test write
            await this.targetClient.secrets.create(testKey, testValue);

            // Test read
            const retrieved = await this.targetClient.secrets.get(testKey);
            if (retrieved !== testValue) {
                this.errors.push({
                    type: "functionality_error",
                    message: "Read/write test failed",
                    details: "Retrieved value does not match written value",
                });
            }

            // Test delete
            await this.targetClient.secrets.delete(testKey);
        } catch (error) {
            this.errors.push({
                type: "functionality_error",
                message: "Basic operations test failed",
                details: error.message,
            });
        }
    }

    generateReport() {
        const report = {
            timestamp: new Date().toISOString(),
            status: this.errors.length === 0 ? "success" : "failed",
            summary: {
                errors: this.errors.length,
                warnings: this.warnings.length,
                source_count: Object.keys(this.sourceData).length,
            },
            errors: this.errors,
            warnings: this.warnings,
        };

        // Save report
        require("fs").writeFileSync(
            `migration-report-${Date.now()}.json`,
            JSON.stringify(report, null, 2),
        );

        return report;
    }
}
```

## Testing Migration

### 1. Pre-Migration Testing

```javascript
// test/pre-migration.test.js
describe("Pre-Migration Tests", () => {
    let sourceSystem;
    let dotenvClient;

    beforeAll(async () => {
        sourceSystem = await initializeSourceSystem();
        dotenvClient = await initializeDotEnvClient();
    });

    test("Source system connectivity", async () => {
        const health = await sourceSystem.healthCheck();
        expect(health.status).toBe("healthy");
    });

    test("DotEnv connectivity", async () => {
        const health = await dotenvClient.healthCheck();
        expect(health.status).toBe("healthy");
    });

    test("Secret count verification", async () => {
        const sourceCount = await sourceSystem.getSecretCount();
        expect(sourceCount).toBeGreaterThan(0);
        console.log(`Found ${sourceCount} secrets to migrate`);
    });

    test("Permission verification", async () => {
        // Test read permissions on source
        const canRead = await sourceSystem.testPermissions("read");
        expect(canRead).toBe(true);

        // Test write permissions on target
        const canWrite = await dotenvClient.testPermissions("write");
        expect(canWrite).toBe(true);
    });

    test("Dry run migration", async () => {
        const migrator = new Migrator(sourceSystem, dotenvClient);
        const result = await migrator.dryRun();

        expect(result.errors).toHaveLength(0);
        expect(result.secretCount).toBeGreaterThan(0);
    });
});
```

### 2. Post-Migration Testing

```javascript
// test/post-migration.test.js
describe("Post-Migration Tests", () => {
    let validator;
    let applicationTests;

    beforeAll(async () => {
        validator = new MigrationValidator();
        applicationTests = new ApplicationTests();
    });

    test("Secret completeness", async () => {
        const result = await validator.validateCompleteness();
        expect(result.missing).toHaveLength(0);
    });

    test("Secret integrity", async () => {
        const result = await validator.validateIntegrity();
        expect(result.mismatches).toHaveLength(0);
    });

    test("Application startup", async () => {
        const app = await applicationTests.startApplication();
        expect(app.status).toBe("running");
    });

    test("Database connectivity", async () => {
        const db = await applicationTests.testDatabase();
        expect(db.connected).toBe(true);
    });

    test("External API connectivity", async () => {
        const apis = await applicationTests.testExternalAPIs();
        apis.forEach((api) => {
            expect(api.status).toBe("connected");
        });
    });

    test("Feature flag loading", async () => {
        const flags = await applicationTests.getFeatureFlags();
        expect(flags).toBeDefined();
        expect(Object.keys(flags).length).toBeGreaterThan(0);
    });
});
```

## Rollback Procedures

### 1. Automated Rollback

```bash
#!/bin/bash
# scripts/migration-rollback.sh

BACKUP_FILE=$1
TARGET_SYSTEM=$2

if [ -z "$BACKUP_FILE" ] || [ -z "$TARGET_SYSTEM" ]; then
  echo "Usage: ./migration-rollback.sh <backup-file> <target-system>"
  exit 1
fi

echo "⚠️  Starting rollback to $TARGET_SYSTEM using $BACKUP_FILE"

# Confirm rollback
read -p "Are you sure you want to rollback? (yes/no): " confirm
if [ "$confirm" != "yes" ]; then
  echo "Rollback cancelled"
  exit 0
fi

# Stop applications
echo "Stopping applications..."
kubectl scale deployment/api-server --replicas=0

# Restore secrets
case "$TARGET_SYSTEM" in
  "vault")
    echo "Restoring to HashiCorp Vault..."
    cat "$BACKUP_FILE" | jq -r '.[] | "\(.key)=\(.value)"' | \
      while IFS='=' read -r key value; do
        vault kv put "secret/$key" value="$value"
      done
    ;;

  "aws")
    echo "Restoring to AWS Secrets Manager..."
    cat "$BACKUP_FILE" | jq -c '.[]' | \
      while read -r secret; do
        key=$(echo "$secret" | jq -r '.key')
        value=$(echo "$secret" | jq -r '.value')
        aws secretsmanager put-secret-value \
          --secret-id "$key" \
          --secret-string "$value"
      done
    ;;

  "dotenv-v1")
    echo "Restoring to DotEnv v1..."
    dotenv import --file "$BACKUP_FILE" --force
    ;;
esac

# Update application configuration
echo "Updating application configuration..."
kubectl delete configmap app-config
kubectl create configmap app-config \
  --from-literal=SECRET_SOURCE="$TARGET_SYSTEM"

# Restart applications
echo "Restarting applications..."
kubectl scale deployment/api-server --replicas=3

# Verify rollback
echo "Verifying rollback..."
./scripts/health-check.sh

echo "✅ Rollback complete"
```

## Best Practices

### 1. Migration Checklist

```markdown
# DotEnv Migration Checklist

## Pre-Migration

- [ ] Inventory all secrets and configurations
- [ ] Document current secret management process
- [ ] Identify sensitive vs non-sensitive data
- [ ] Create comprehensive backup
- [ ] Set up DotEnv organization and projects
- [ ] Configure access controls
- [ ] Test migration scripts in non-production

## During Migration

- [ ] Run migration in maintenance window
- [ ] Monitor migration progress
- [ ] Validate each batch of secrets
- [ ] Test application functionality
- [ ] Document any issues or deviations

## Post-Migration

- [ ] Verify all secrets migrated
- [ ] Run application smoke tests
- [ ] Monitor application performance
- [ ] Update documentation
- [ ] Train team on new processes
- [ ] Decommission old system (after stability period)
```

### 2. Common Pitfalls

```javascript
// ❌ Bad: No validation
async function migrate() {
    const secrets = await source.getAll();
    await target.importAll(secrets);
}

// ✅ Good: Comprehensive validation
async function migrate() {
    const secrets = await source.getAll();

    // Validate before migration
    const validation = await validateSecrets(secrets);
    if (!validation.isValid) {
        throw new Error("Validation failed: " + validation.errors);
    }

    // Migrate in batches with verification
    const batches = chunk(secrets, 50);
    for (const batch of batches) {
        await target.importBatch(batch);
        await verifyBatch(batch);
    }

    // Final validation
    const finalCheck = await validateMigration(source, target);
    if (!finalCheck.success) {
        await rollback();
        throw new Error("Migration verification failed");
    }
}
```

## Resources

- [Migration Planning Guide](https://www.atlassian.com/incident-management/kpis/common-metrics)
- [Zero-Downtime Migrations](https://martinfowler.com/articles/evodb.html)
- [Secret Rotation Best Practices](https://cloud.google.com/secret-manager/docs/best-practices)
- [DotEnv Import Documentation](/documentation/v1/cli/commands#import)

## Next Steps

- [Assess current secret management](#overview)
- [Choose migration strategy](#migration-strategies)
- [Set up DotEnv environment](/documentation/v1/getting-started/first-project)
- [Execute migration plan](#from-popular-solutions)
