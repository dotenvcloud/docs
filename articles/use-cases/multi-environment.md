---
title: Multi-Environment Deployment
slug: multi-environment
order: 3
tags: [use-cases, environments, deployment, workflows]
---

# Multi-Environment Deployment

Learn how to manage secrets across multiple deployment environments using DotEnv. This guide covers patterns for development, staging, production, and specialized environments like QA, demo, and regional deployments.

## Overview

Multi-environment challenges:

- Environment parity
- Progressive deployment
- Environment-specific configurations
- Secret promotion workflows
- Rollback procedures
- Regional deployments

## Environment Strategy

### Standard Environment Hierarchy

```
Development
    ↓
  Staging
    ↓
 Production
```

### Extended Environment Setup

```
Local Development
    ↓
Development (Cloud)
    ↓
    QA
    ↓
  Staging
    ↓
UAT (User Acceptance)
    ↓
 Production
    ↓
Production (Regional)
```

## Environment Configuration

### 1. Environment Definition

```yaml
# .dotenv/environments.yaml
environments:
    # Development environments
    local:
        description: "Local development"
        inherits: null
        auto_sync: true

    development:
        description: "Shared development server"
        inherits: local
        url: https://dev.example.com

    # Testing environments
    qa:
        description: "QA testing"
        inherits: development
        url: https://qa.example.com
        restrictions:
            - no_customer_data

    staging:
        description: "Pre-production staging"
        inherits: qa
        url: https://staging.example.com
        restrictions:
            - limited_real_data

    # Production environments
    uat:
        description: "User acceptance testing"
        inherits: staging
        url: https://uat.example.com

    production:
        description: "Production environment"
        inherits: null
        url: https://app.example.com
        restrictions:
            - no_direct_access
            - requires_approval

    production-eu:
        description: "EU production region"
        inherits: production
        url: https://eu.app.example.com
        region: eu-west-1

    production-us:
        description: "US production region"
        inherits: production
        url: https://us.app.example.com
        region: us-east-1
```

### 2. Environment-Specific Values

```javascript
// config/environments.js
const environments = {
    local: {
        API_URL: "http://localhost:3000",
        DATABASE_URL: "postgresql://local:local@localhost:5432/myapp_dev",
        REDIS_URL: "redis://localhost:6379/0",
        LOG_LEVEL: "debug",
        ENABLE_DEBUG: true,
    },

    development: {
        API_URL: "https://api-dev.example.com",
        DATABASE_URL: "{{SECRET}}", // Pulled from DotEnv
        REDIS_URL: "{{SECRET}}",
        LOG_LEVEL: "debug",
        ENABLE_DEBUG: true,
    },

    staging: {
        API_URL: "https://api-staging.example.com",
        DATABASE_URL: "{{SECRET}}",
        REDIS_URL: "{{SECRET}}",
        LOG_LEVEL: "info",
        ENABLE_DEBUG: false,
    },

    production: {
        API_URL: "https://api.example.com",
        DATABASE_URL: "{{SECRET}}",
        REDIS_URL: "{{SECRET}}",
        LOG_LEVEL: "error",
        ENABLE_DEBUG: false,
        SENTRY_DSN: "{{SECRET}}",
    },
};
```

## Progressive Deployment

### 1. Promotion Workflow

```bash
#!/bin/bash
# scripts/promote-secrets.sh

source_env=$1
target_env=$2
project=$3

if [ -z "$source_env" ] || [ -z "$target_env" ] || [ -z "$project" ]; then
  echo "Usage: ./promote-secrets.sh <source> <target> <project>"
  exit 1
fi

echo "Promoting secrets from $source_env to $target_env for $project"

# Validation checks
case "$target_env" in
  "production")
    if [ "$source_env" != "staging" ] && [ "$source_env" != "uat" ]; then
      echo "❌ Production can only be promoted from staging or uat"
      exit 1
    fi
    ;;
  "staging")
    if [ "$source_env" != "qa" ] && [ "$source_env" != "development" ]; then
      echo "❌ Staging can only be promoted from qa or development"
      exit 1
    fi
    ;;
esac

# Create backup of target environment
echo "Creating backup of $target_env..."
dotenv pull $target_env --project $project --format json > "backup-$target_env-$(date +%Y%m%d-%H%M%S).json"

# Get secrets from source
echo "Fetching secrets from $source_env..."
source_secrets=$(dotenv pull $source_env --project $project --format json)

# Filter environment-specific values
filtered_secrets=$(echo "$source_secrets" | jq 'del(.DATABASE_URL, .REDIS_URL, .ELASTICSEARCH_URL)')

# Merge with target environment values
echo "Merging with $target_env values..."
dotenv pull $target_env --project $project --format json > current-target.json

merged_secrets=$(jq -s '.[0] * .[1]' current-target.json <(echo "$filtered_secrets"))

# Update target environment
echo "Updating $target_env..."
echo "$merged_secrets" | dotenv push $target_env --project $project --format json

echo "✅ Successfully promoted from $source_env to $target_env"
```

### 2. Automated Promotion Pipeline

```yaml
# .github/workflows/promote-environments.yml
name: Promote Environments

on:
    workflow_dispatch:
        inputs:
            source:
                description: "Source environment"
                required: true
                type: choice
                options:
                    - development
                    - qa
                    - staging
                    - uat
            target:
                description: "Target environment"
                required: true
                type: choice
                options:
                    - qa
                    - staging
                    - uat
                    - production
            project:
                description: "Project name"
                required: true
                default: "my-app"

jobs:
    validate:
        runs-on: ubuntu-latest
        steps:
            - name: Validate promotion path
              run: |
                  case "${{ inputs.target }}" in
                    "production")
                      if [[ "${{ inputs.source }}" != "staging" && "${{ inputs.source }}" != "uat" ]]; then
                        echo "Invalid promotion path"
                        exit 1
                      fi
                      ;;
                    "staging"|"uat")
                      if [[ "${{ inputs.source }}" != "qa" && "${{ inputs.source }}" != "development" ]]; then
                        echo "Invalid promotion path"
                        exit 1
                      fi
                      ;;
                  esac

    promote:
        needs: validate
        runs-on: ubuntu-latest
        environment: ${{ inputs.target }}
        steps:
            - uses: actions/checkout@v3

            - name: Install DotEnv CLI
              run: |
                  curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                  echo "$HOME/.dotenv/bin" >> $GITHUB_PATH

            - name: Promote secrets
              env:
                  DOTENV_API_KEY: ${{ secrets.DOTENV_API_KEY }}
              run: |
                  dotenv auth login --api-key $DOTENV_API_KEY
                  ./scripts/promote-secrets.sh ${{ inputs.source }} ${{ inputs.target }} ${{ inputs.project }}

            - name: Run smoke tests
              run: |
                  ./scripts/smoke-test.sh ${{ inputs.target }}
```

## Environment Isolation

### 1. Database Isolation

```javascript
// config/database.js
const knex = require("knex");

function getDatabaseConfig(environment) {
    const baseConfig = {
        client: "postgresql",
        connection: process.env.DATABASE_URL,
        pool: {
            min: 2,
            max: 10,
        },
    };

    // Environment-specific configurations
    switch (environment) {
        case "local":
            return {
                ...baseConfig,
                connection: {
                    host: "localhost",
                    port: 5432,
                    database: "myapp_local",
                    user: "postgres",
                    password: "postgres",
                },
            };

        case "development":
        case "qa":
            return {
                ...baseConfig,
                connection: process.env.DATABASE_URL,
                pool: {
                    min: 2,
                    max: 20,
                },
            };

        case "staging":
        case "uat":
            return {
                ...baseConfig,
                connection: process.env.DATABASE_URL,
                pool: {
                    min: 5,
                    max: 30,
                },
                // Read replica for reporting
                readReplica: {
                    connection: process.env.DATABASE_READ_URL,
                },
            };

        case "production":
            return {
                ...baseConfig,
                connection: process.env.DATABASE_URL,
                pool: {
                    min: 10,
                    max: 50,
                },
                // Multiple read replicas
                readReplicas: [
                    { connection: process.env.DATABASE_READ_URL_1 },
                    { connection: process.env.DATABASE_READ_URL_2 },
                ],
            };

        default:
            throw new Error(`Unknown environment: ${environment}`);
    }
}

module.exports = getDatabaseConfig(process.env.NODE_ENV);
```

### 2. Feature Flags

```typescript
// services/feature-flags.ts
interface FeatureFlag {
    key: string;
    defaultValue: boolean;
    environments: {
        [env: string]: boolean;
    };
}

const featureFlags: FeatureFlag[] = [
    {
        key: "NEW_DASHBOARD",
        defaultValue: false,
        environments: {
            local: true,
            development: true,
            qa: true,
            staging: true,
            uat: true,
            production: false,
        },
    },
    {
        key: "BETA_FEATURES",
        defaultValue: false,
        environments: {
            local: true,
            development: true,
            qa: true,
            staging: false,
            uat: false,
            production: false,
        },
    },
    {
        key: "MAINTENANCE_MODE",
        defaultValue: false,
        environments: {
            local: false,
            development: false,
            qa: false,
            staging: false,
            uat: false,
            production: false,
        },
    },
];

export class FeatureFlags {
    private environment: string;

    constructor(environment: string = process.env.NODE_ENV || "development") {
        this.environment = environment;
    }

    isEnabled(flagKey: string): boolean {
        const flag = featureFlags.find((f) => f.key === flagKey);
        if (!flag) return false;

        // Check environment-specific value
        if (flag.environments[this.environment] !== undefined) {
            return flag.environments[this.environment];
        }

        // Fall back to default
        return flag.defaultValue;
    }

    getAllFlags(): Record<string, boolean> {
        return featureFlags.reduce(
            (acc, flag) => {
                acc[flag.key] = this.isEnabled(flag.key);
                return acc;
            },
            {} as Record<string, boolean>,
        );
    }
}
```

## Regional Deployments

### 1. Region Configuration

```javascript
// config/regions.js
const regions = {
    "us-east-1": {
        name: "US East",
        endpoint: "https://us-east.api.example.com",
        database: {
            write: process.env.US_EAST_DATABASE_URL,
            read: process.env.US_EAST_DATABASE_READ_URL,
        },
        redis: process.env.US_EAST_REDIS_URL,
        cdn: "https://us-east-cdn.example.com",
    },

    "eu-west-1": {
        name: "EU West",
        endpoint: "https://eu-west.api.example.com",
        database: {
            write: process.env.EU_WEST_DATABASE_URL,
            read: process.env.EU_WEST_DATABASE_READ_URL,
        },
        redis: process.env.EU_WEST_REDIS_URL,
        cdn: "https://eu-west-cdn.example.com",
        compliance: ["GDPR"],
    },

    "ap-southeast-1": {
        name: "Asia Pacific",
        endpoint: "https://ap-southeast.api.example.com",
        database: {
            write: process.env.AP_SOUTHEAST_DATABASE_URL,
            read: process.env.AP_SOUTHEAST_DATABASE_READ_URL,
        },
        redis: process.env.AP_SOUTHEAST_REDIS_URL,
        cdn: "https://ap-southeast-cdn.example.com",
    },
};

function getRegionConfig() {
    const region = process.env.AWS_REGION || "us-east-1";
    return regions[region] || regions["us-east-1"];
}

module.exports = { regions, getRegionConfig };
```

### 2. Multi-Region Deployment

```yaml
# k8s/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: api-server
spec:
    replicas: 3
    template:
        spec:
            containers:
                - name: api
                  image: myapp:latest
                  env:
                      - name: REGION
                        value: "$(REGION)"
                      - name: DOTENV_PROJECT
                        value: "myapp-$(REGION)"
                      - name: DOTENV_ENVIRONMENT
                        value: "production-$(REGION)"
---
# k8s/overlays/us-east-1/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
    - ../../base

patchesStrategicMerge:
    - deployment-patch.yaml

configMapGenerator:
    - name: region-config
      literals:
          - REGION=us-east-1
          - TIMEZONE=America/New_York
---
# k8s/overlays/eu-west-1/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
    - ../../base

patchesStrategicMerge:
    - deployment-patch.yaml

configMapGenerator:
    - name: region-config
      literals:
          - REGION=eu-west-1
          - TIMEZONE=Europe/London
          - GDPR_COMPLIANT=true
```

## Environment Validation

### 1. Configuration Validator

```typescript
// utils/env-validator.ts
import Joi from "joi";

interface EnvironmentSchema {
    [key: string]: Joi.Schema;
}

const commonSchema: EnvironmentSchema = {
    NODE_ENV: Joi.string()
        .valid("local", "development", "qa", "staging", "uat", "production")
        .required(),
    LOG_LEVEL: Joi.string().valid("debug", "info", "warn", "error").required(),
    PORT: Joi.number().port().default(3000),
    API_KEY: Joi.string().min(32).required(),
};

const environmentSchemas: Record<string, EnvironmentSchema> = {
    local: {
        ...commonSchema,
        DATABASE_URL: Joi.string().uri({ scheme: "postgresql" }).required(),
        REDIS_URL: Joi.string().uri({ scheme: "redis" }).required(),
    },

    development: {
        ...commonSchema,
        DATABASE_URL: Joi.string().uri({ scheme: "postgresql" }).required(),
        REDIS_URL: Joi.string().uri({ scheme: "redis" }).required(),
        SENTRY_DSN: Joi.string().uri().optional(),
    },

    production: {
        ...commonSchema,
        DATABASE_URL: Joi.string().uri({ scheme: "postgresql" }).required(),
        DATABASE_READ_URL: Joi.string()
            .uri({ scheme: "postgresql" })
            .required(),
        REDIS_URL: Joi.string().uri({ scheme: "redis" }).required(),
        SENTRY_DSN: Joi.string().uri().required(),
        NEW_RELIC_LICENSE_KEY: Joi.string().required(),
        CDN_URL: Joi.string().uri().required(),
    },
};

export function validateEnvironment(
    environment: string = process.env.NODE_ENV || "development",
) {
    const schema = environmentSchemas[environment];
    if (!schema) {
        throw new Error(`No schema defined for environment: ${environment}`);
    }

    const validation = Joi.object(schema).validate(process.env, {
        allowUnknown: true,
        abortEarly: false,
    });

    if (validation.error) {
        console.error(`Environment validation failed for ${environment}:`);
        validation.error.details.forEach((detail) => {
            console.error(`  - ${detail.message}`);
        });
        throw new Error("Invalid environment configuration");
    }

    return validation.value;
}
```

### 2. Health Checks

```javascript
// routes/health.js
const express = require("express");
const router = express.Router();

router.get("/health", (req, res) => {
    res.json({
        status: "healthy",
        environment: process.env.NODE_ENV,
        version: process.env.APP_VERSION || "unknown",
        timestamp: new Date().toISOString(),
    });
});

router.get("/health/detailed", async (req, res) => {
    const checks = {
        environment: process.env.NODE_ENV,
        version: process.env.APP_VERSION || "unknown",
        services: {},
    };

    // Database check
    try {
        await db.raw("SELECT 1");
        checks.services.database = "healthy";
    } catch (error) {
        checks.services.database = "unhealthy";
    }

    // Redis check
    try {
        await redis.ping();
        checks.services.redis = "healthy";
    } catch (error) {
        checks.services.redis = "unhealthy";
    }

    // External API check
    try {
        const response = await fetch(process.env.API_URL + "/health");
        checks.services.api = response.ok ? "healthy" : "unhealthy";
    } catch (error) {
        checks.services.api = "unhealthy";
    }

    const allHealthy = Object.values(checks.services).every(
        (status) => status === "healthy",
    );

    res.status(allHealthy ? 200 : 503).json(checks);
});

module.exports = router;
```

## Rollback Procedures

### 1. Automated Rollback

```bash
#!/bin/bash
# scripts/rollback.sh

environment=$1
project=$2
backup_file=$3

if [ -z "$environment" ] || [ -z "$project" ]; then
  echo "Usage: ./rollback.sh <environment> <project> [backup-file]"
  exit 1
fi

# Find latest backup if not specified
if [ -z "$backup_file" ]; then
  backup_file=$(ls -t backup-$environment-*.json | head -1)
  if [ -z "$backup_file" ]; then
    echo "❌ No backup file found for $environment"
    exit 1
  fi
fi

echo "Rolling back $environment for $project using $backup_file"

# Verify backup file
if [ ! -f "$backup_file" ]; then
  echo "❌ Backup file not found: $backup_file"
  exit 1
fi

# Create current state backup
echo "Backing up current state..."
dotenv pull $environment --project $project --format json > "rollback-current-$(date +%Y%m%d-%H%M%S).json"

# Restore from backup
echo "Restoring from backup..."
cat "$backup_file" | dotenv push $environment --project $project --format json

echo "✅ Successfully rolled back $environment"

# Trigger deployment restart
echo "Restarting services..."
case "$environment" in
  "production")
    kubectl rollout restart deployment/api-server -n production
    ;;
  "staging")
    kubectl rollout restart deployment/api-server -n staging
    ;;
  *)
    echo "Manual service restart required for $environment"
    ;;
esac
```

### 2. Blue-Green Deployment

```javascript
// deployment/blue-green.js
const AWS = require("aws-sdk");
const elbv2 = new AWS.ELBv2();

class BlueGreenDeployment {
    constructor(config) {
        this.config = config;
        this.targetGroupArns = {
            blue: config.blueTargetGroupArn,
            green: config.greenTargetGroupArn,
        };
    }

    async getCurrentEnvironment() {
        const params = {
            ListenerArn: this.config.listenerArn,
        };

        const data = await elbv2.describeListenerRules(params).promise();
        const defaultRule = data.Rules.find((r) => r.IsDefault);

        const activeTargetGroup = defaultRule.Actions[0].TargetGroupArn;
        return activeTargetGroup === this.targetGroupArns.blue
            ? "blue"
            : "green";
    }

    async switchEnvironment() {
        const current = await this.getCurrentEnvironment();
        const target = current === "blue" ? "green" : "blue";

        console.log(`Switching from ${current} to ${target}`);

        // Update listener rule
        const params = {
            ListenerArn: this.config.listenerArn,
            Rules: [
                {
                    RuleArn: this.config.defaultRuleArn,
                    Actions: [
                        {
                            Type: "forward",
                            TargetGroupArn: this.targetGroupArns[target],
                        },
                    ],
                },
            ],
        };

        await elbv2.modifyRule(params).promise();

        // Wait for target group to be healthy
        await this.waitForHealthy(this.targetGroupArns[target]);

        console.log(`Successfully switched to ${target}`);
        return target;
    }

    async rollback() {
        console.log("Initiating rollback...");
        return await this.switchEnvironment();
    }

    async waitForHealthy(targetGroupArn, maxAttempts = 60) {
        for (let i = 0; i < maxAttempts; i++) {
            const health = await elbv2
                .describeTargetHealth({
                    TargetGroupArn: targetGroupArn,
                })
                .promise();

            const healthy = health.TargetHealthDescriptions.every(
                (t) => t.TargetHealth.State === "healthy",
            );

            if (healthy) return true;

            console.log(
                `Waiting for targets to be healthy... (${i + 1}/${maxAttempts})`,
            );
            await new Promise((resolve) => setTimeout(resolve, 5000));
        }

        throw new Error("Targets did not become healthy in time");
    }
}
```

## Monitoring and Alerts

### 1. Environment Metrics

```javascript
// monitoring/metrics.js
const prometheus = require("prom-client");

// Environment-specific metrics
const environmentInfo = new prometheus.Gauge({
    name: "app_environment_info",
    help: "Application environment information",
    labelNames: ["environment", "version", "region"],
});

const secretLoadDuration = new prometheus.Histogram({
    name: "dotenv_secret_load_duration_seconds",
    help: "Time taken to load secrets from DotEnv",
    labelNames: ["environment", "status"],
    buckets: [0.1, 0.5, 1, 2, 5, 10],
});

const environmentMismatch = new prometheus.Counter({
    name: "app_environment_mismatch_total",
    help: "Count of environment mismatches detected",
    labelNames: ["expected", "actual"],
});

// Initialize metrics
environmentInfo.set(
    {
        environment: process.env.NODE_ENV,
        version: process.env.APP_VERSION || "unknown",
        region: process.env.AWS_REGION || "unknown",
    },
    1,
);

module.exports = {
    environmentInfo,
    secretLoadDuration,
    environmentMismatch,
    register: prometheus.register,
};
```

## Best Practices

### 1. Environment Naming

```javascript
// Good naming conventions
const environments = [
    "development", // Shared dev environment
    "staging", // Pre-production testing
    "production", // Production
    "production-eu", // Regional production
    "production-us", // Regional production
    "feature-xyz", // Feature branch environment
    "hotfix-123", // Hotfix environment
];

// Bad naming conventions
const badEnvironments = [
    "dev", // Too abbreviated
    "prod", // Too abbreviated
    "test", // Ambiguous
    "new-prod", // Unclear purpose
    "prod-backup", // Confusing
];
```

### 2. Secret Organization

```yaml
# Environment-specific secret organization
secrets:
    # Shared across all environments
    shared:
        - APP_NAME
        - PUBLIC_API_VERSION

    # Environment-specific
    development:
        - DATABASE_URL
        - REDIS_URL
        - DEBUG_MODE=true

    staging:
        - DATABASE_URL
        - REDIS_URL
        - DEBUG_MODE=false
        - FEATURE_FLAGS

    production:
        - DATABASE_URL
        - DATABASE_READ_URL
        - REDIS_URL
        - REDIS_CACHE_URL
        - CDN_URL
        - MONITORING_KEY
        - DEBUG_MODE=false
```

## Resources

- [12-Factor App: Config](https://12factor.net/config)
- [Environment Promotion Best Practices](https://www.atlassian.com/continuous-delivery/principles/deployment-strategies)
- [Blue-Green Deployments](https://martinfowler.com/bliki/BlueGreenDeployment.html)
- [DotEnv Environment Management](/documentation/v1/core-concepts/environments-concept)

## Next Steps

- [Define environment strategy](#environment-strategy)
- [Set up promotion workflows](#progressive-deployment)
- [Configure regional deployments](#regional-deployments)
- [Implement rollback procedures](#rollback-procedures)
