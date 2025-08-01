---
title: Production Deployment
slug: production-deployment
order: 8
tags: [use-cases, production, deployment, security]
---

# Production Deployment

Learn how to securely deploy and manage production environments using DotEnv. This guide covers best practices for zero-downtime deployments, secret rotation, monitoring, and maintaining production security.

## Overview

Production deployment challenges:

- Zero-downtime deployments
- Secret security and rotation
- Performance at scale
- Disaster recovery
- Compliance requirements
- Audit trails

## Production Architecture

### 1. High Availability Setup

```yaml
# infrastructure/production/ha-config.yaml
production:
    regions:
        primary:
            name: us-east-1
            components:
                - api_servers: 10
                - database:
                      primary: 1
                      replicas: 2
                - cache_nodes: 3
                - load_balancers: 2

        secondary:
            name: eu-west-1
            components:
                - api_servers: 8
                - database:
                      replica: 1
                - cache_nodes: 2
                - load_balancers: 2

    failover:
        strategy: active-passive
        health_check_interval: 10s
        failover_threshold: 3
        automatic_failover: true

    backup:
        database:
            frequency: hourly
            retention: 30d
            point_in_time_recovery: true

        secrets:
            frequency: daily
            encryption: AES-256-GCM
            storage:
                - s3://prod-backups/secrets/
                - glacier://long-term-backups/
```

### 2. Infrastructure as Code

```terraform
# terraform/production/main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
    dotenv = {
      source  = "dotenv/dotenv"
      version = "~> 1.0"
    }
  }

  backend "s3" {
    bucket         = "terraform-state-prod"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

# Production VPC
module "vpc" {
  source = "./modules/vpc"

  cidr_block           = "10.0.0.0/16"
  availability_zones   = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnet_cidrs  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  enable_vpn_gateway = true

  tags = {
    Environment = "production"
    Terraform   = "true"
  }
}

# EKS Cluster
module "eks" {
  source = "./modules/eks"

  cluster_name    = "prod-cluster"
  cluster_version = "1.27"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids

  node_groups = {
    general = {
      desired_capacity = 10
      max_capacity     = 20
      min_capacity     = 5

      instance_types = ["t3.xlarge"]

      k8s_labels = {
        Environment = "production"
        NodeType    = "general"
      }
    }

    memory_optimized = {
      desired_capacity = 3
      max_capacity     = 6
      min_capacity     = 2

      instance_types = ["r5.2xlarge"]

      k8s_labels = {
        Environment = "production"
        NodeType    = "memory-optimized"
      }

      taints = [{
        key    = "memory-optimized"
        value  = "true"
        effect = "NO_SCHEDULE"
      }]
    }
  }
}

# RDS Multi-AZ
module "rds" {
  source = "./modules/rds"

  identifier = "prod-database"

  engine         = "postgres"
  engine_version = "14.7"
  instance_class = "db.r5.2xlarge"

  allocated_storage     = 1000
  max_allocated_storage = 5000
  storage_encrypted     = true

  multi_az               = true
  backup_retention_period = 30
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids

  performance_insights_enabled = true
  monitoring_interval         = 60

  tags = {
    Environment = "production"
    Critical    = "true"
  }
}

# DotEnv Integration
resource "dotenv_project" "production" {
  name        = "my-app-production"
  description = "Production environment secrets"

  environment {
    name = "production"

    secret {
      key   = "DATABASE_URL"
      value = module.rds.connection_string
    }

    secret {
      key   = "REDIS_URL"
      value = module.elasticache.connection_string
    }

    secret {
      key   = "AWS_REGION"
      value = var.aws_region
    }
  }

  access_control {
    require_mfa = true

    ip_whitelist = [
      "10.0.0.0/16",  # VPC CIDR
      var.office_ip   # Office IP
    ]

    approval_required = true
    approvers        = ["ops-team@company.com"]
  }
}
```

## Deployment Pipeline

### 1. Blue-Green Deployment

```javascript
// deploy/blue-green.js
const AWS = require("aws-sdk");
const k8s = require("@kubernetes/client-node");

class BlueGreenDeployment {
    constructor(config) {
        this.config = config;
        this.elbv2 = new AWS.ELBv2();
        this.cloudwatch = new AWS.CloudWatch();

        const kc = new k8s.KubeConfig();
        kc.loadFromDefault();
        this.k8sApi = kc.makeApiClient(k8s.AppsV1Api);
    }

    async deploy(version, options = {}) {
        const {
            canaryPercent = 10,
            canaryDuration = 300000, // 5 minutes
            autoRollback = true,
        } = options;

        console.log(`Starting blue-green deployment of version ${version}`);

        try {
            // Phase 1: Deploy to green environment
            await this.deployToGreen(version);

            // Phase 2: Health checks
            await this.performHealthChecks("green");

            // Phase 3: Canary deployment
            await this.canaryDeploy(canaryPercent);

            // Phase 4: Monitor canary
            const canaryHealthy = await this.monitorCanary(canaryDuration);

            if (!canaryHealthy && autoRollback) {
                await this.rollback();
                throw new Error("Canary deployment failed, rolled back");
            }

            // Phase 5: Full switch
            await this.switchToGreen();

            // Phase 6: Final validation
            await this.validateDeployment();

            // Phase 7: Cleanup old version
            await this.cleanupBlue();

            console.log("✅ Deployment completed successfully");

            return {
                version,
                deployedAt: new Date(),
                status: "success",
            };
        } catch (error) {
            console.error("❌ Deployment failed:", error);

            if (autoRollback) {
                await this.rollback();
            }

            throw error;
        }
    }

    async deployToGreen(version) {
        console.log("Deploying to green environment...");

        // Update Kubernetes deployment
        const deployment = await this.k8sApi.readNamespacedDeployment(
            "app-green",
            "production",
        );

        deployment.body.spec.template.spec.containers[0].image = `myapp:${version}`;

        await this.k8sApi.replaceNamespacedDeployment(
            "app-green",
            "production",
            deployment.body,
        );

        // Wait for rollout
        await this.waitForRollout("app-green");
    }

    async performHealthChecks(environment) {
        console.log(`Performing health checks on ${environment}...`);

        const checks = [
            this.checkEndpoint(`http://${environment}.internal/health`),
            this.checkDatabase(environment),
            this.checkDependencies(environment),
        ];

        const results = await Promise.all(checks);

        if (results.some((r) => !r.healthy)) {
            throw new Error(`Health checks failed for ${environment}`);
        }
    }

    async canaryDeploy(percentage) {
        console.log(`Starting canary deployment (${percentage}%)...`);

        // Update ALB target group weights
        await this.elbv2
            .modifyRule({
                RuleArn: this.config.albRuleArn,
                Actions: [
                    {
                        Type: "forward",
                        ForwardConfig: {
                            TargetGroups: [
                                {
                                    TargetGroupArn: this.config.blueTargetGroup,
                                    Weight: 100 - percentage,
                                },
                                {
                                    TargetGroupArn:
                                        this.config.greenTargetGroup,
                                    Weight: percentage,
                                },
                            ],
                        },
                    },
                ],
            })
            .promise();
    }

    async monitorCanary(duration) {
        console.log(`Monitoring canary for ${duration / 1000}s...`);

        const startTime = Date.now();
        const metrics = {
            errors: 0,
            latency: [],
            success: 0,
        };

        while (Date.now() - startTime < duration) {
            const health = await this.checkCanaryHealth();

            metrics.errors += health.errors;
            metrics.success += health.success;
            metrics.latency.push(health.latency);

            // Check error rate
            const errorRate =
                metrics.errors / (metrics.errors + metrics.success);
            if (errorRate > 0.05) {
                // 5% error threshold
                console.error(
                    `High error rate detected: ${(errorRate * 100).toFixed(2)}%`,
                );
                return false;
            }

            // Check latency
            const avgLatency =
                metrics.latency.reduce((a, b) => a + b, 0) /
                metrics.latency.length;
            if (avgLatency > 1000) {
                // 1s threshold
                console.error(`High latency detected: ${avgLatency}ms`);
                return false;
            }

            await new Promise((resolve) => setTimeout(resolve, 10000)); // Check every 10s
        }

        return true;
    }

    async switchToGreen() {
        console.log("Switching all traffic to green...");

        await this.elbv2
            .modifyRule({
                RuleArn: this.config.albRuleArn,
                Actions: [
                    {
                        Type: "forward",
                        TargetGroupArn: this.config.greenTargetGroup,
                    },
                ],
            })
            .promise();

        // Update DNS if using Route53
        await this.updateDNS("green");
    }

    async rollback() {
        console.log("⚠️  Rolling back deployment...");

        // Switch all traffic back to blue
        await this.elbv2
            .modifyRule({
                RuleArn: this.config.albRuleArn,
                Actions: [
                    {
                        Type: "forward",
                        TargetGroupArn: this.config.blueTargetGroup,
                    },
                ],
            })
            .promise();

        // Scale down green
        await this.k8sApi.patchNamespacedDeployment("app-green", "production", {
            spec: { replicas: 0 },
        });
    }

    async checkCanaryHealth() {
        // Get metrics from CloudWatch
        const params = {
            MetricName: "RequestCount",
            Namespace: "AWS/ApplicationELB",
            StartTime: new Date(Date.now() - 60000),
            EndTime: new Date(),
            Period: 60,
            Statistics: ["Sum"],
            Dimensions: [
                {
                    Name: "TargetGroup",
                    Value: this.config.greenTargetGroup.split(":").pop(),
                },
            ],
        };

        const data = await this.cloudwatch
            .getMetricStatistics(params)
            .promise();

        // Parse metrics
        return {
            errors: data.Datapoints[0]?.Sum || 0,
            success: data.Datapoints[0]?.Sum || 0,
            latency: data.Datapoints[0]?.Average || 0,
        };
    }
}

module.exports = BlueGreenDeployment;
```

### 2. Deployment Automation

```yaml
# .github/workflows/production-deploy.yml
name: Production Deployment

on:
    push:
        tags:
            - "v*"

env:
    AWS_REGION: us-east-1
    ECR_REPOSITORY: myapp
    EKS_CLUSTER: prod-cluster

jobs:
    validate:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v3

            - name: Validate tag format
              run: |
                  if ! [[ "${{ github.ref }}" =~ ^refs/tags/v[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
                    echo "Invalid tag format. Must be vX.Y.Z"
                    exit 1
                  fi

            - name: Security scan
              uses: aquasecurity/trivy-action@master
              with:
                  image-ref: "."
                  format: "sarif"
                  output: "trivy-results.sarif"

            - name: Upload security results
              uses: github/codeql-action/upload-sarif@v2
              with:
                  sarif_file: "trivy-results.sarif"

    build:
        needs: validate
        runs-on: ubuntu-latest
        outputs:
            image: ${{ steps.image.outputs.image }}
        steps:
            - uses: actions/checkout@v3

            - name: Configure AWS credentials
              uses: aws-actions/configure-aws-credentials@v1
              with:
                  aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
                  aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
                  aws-region: ${{ env.AWS_REGION }}

            - name: Login to ECR
              id: login-ecr
              uses: aws-actions/amazon-ecr-login@v1

            - name: Build and push image
              env:
                  ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
                  IMAGE_TAG: ${{ github.ref_name }}
              run: |
                  docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
                  docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
                  echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

    deploy:
        needs: build
        runs-on: ubuntu-latest
        environment: production
        steps:
            - uses: actions/checkout@v3

            - name: Configure AWS credentials
              uses: aws-actions/configure-aws-credentials@v1
              with:
                  aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
                  aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
                  aws-region: ${{ env.AWS_REGION }}

            - name: Update kubeconfig
              run: |
                  aws eks update-kubeconfig --name ${{ env.EKS_CLUSTER }} --region ${{ env.AWS_REGION }}

            - name: Load secrets from DotEnv
              env:
                  DOTENV_API_KEY: ${{ secrets.DOTENV_API_KEY }}
              run: |
                  curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                  export PATH="$HOME/.dotenv/bin:$PATH"

                  dotenv auth login --api-key $DOTENV_API_KEY
                  dotenv pull production --project my-app-production --format kubernetes > secrets.yaml

                  kubectl apply -f secrets.yaml -n production

            - name: Deploy using blue-green strategy
              run: |
                  node deploy/blue-green.js \
                    --version ${{ github.ref_name }} \
                    --canary-percent 10 \
                    --canary-duration 600000 \
                    --auto-rollback true

            - name: Run smoke tests
              run: |
                  npm run test:production:smoke

            - name: Update deployment status
              if: always()
              run: |
                  STATUS="${{ job.status }}"
                  curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
                    -H 'Content-Type: application/json' \
                    -d "{
                      \"text\": \"Production deployment ${{ github.ref_name }}: ${STATUS}\",
                      \"color\": \"${STATUS == 'success' ? 'good' : 'danger'}\"
                    }"
```

## Secret Management

### 1. Production Secret Rotation

```javascript
// scripts/rotate-production-secrets.js
const { DotEnvClient } = require("@dotenv/sdk");
const AWS = require("aws-sdk");
const crypto = require("crypto");

class ProductionSecretRotation {
    constructor() {
        this.dotenv = new DotEnvClient({
            apiKey: process.env.DOTENV_API_KEY,
        });

        this.secretsManager = new AWS.SecretsManager({
            region: "us-east-1",
        });

        this.rotationSchedule = {
            DATABASE_PASSWORD: { frequency: "30d", type: "database" },
            JWT_SECRET: { frequency: "90d", type: "token" },
            API_KEY: { frequency: "180d", type: "api" },
            ENCRYPTION_KEY: { frequency: "365d", type: "encryption" },
        };
    }

    async rotateSecrets(options = {}) {
        const { force = false, dryRun = false, notify = true } = options;

        console.log("🔐 Starting production secret rotation...");

        const secretsToRotate = await this.getSecretsForRotation(force);

        if (secretsToRotate.length === 0) {
            console.log("✅ No secrets need rotation");
            return;
        }

        console.log(`Found ${secretsToRotate.length} secrets to rotate`);

        if (dryRun) {
            console.log("DRY RUN - No changes will be made");
            secretsToRotate.forEach((s) => console.log(`  - ${s.key}`));
            return;
        }

        // Create backup
        await this.backupCurrentSecrets();

        try {
            for (const secret of secretsToRotate) {
                await this.rotateSecret(secret);
            }

            // Verify deployment still works
            await this.verifyServices();

            // Send notifications
            if (notify) {
                await this.notifyRotation(secretsToRotate);
            }

            console.log("✅ Secret rotation completed successfully");
        } catch (error) {
            console.error("❌ Secret rotation failed:", error);
            await this.rollbackSecrets();
            throw error;
        }
    }

    async getSecretsForRotation(force) {
        const secrets = await this.dotenv.listSecrets(
            "my-app-production",
            "production",
        );
        const secretsToRotate = [];

        for (const secret of secrets) {
            const schedule = this.rotationSchedule[secret.key];
            if (!schedule) continue;

            const lastRotated = new Date(
                secret.metadata?.lastRotated || secret.createdAt,
            );
            const rotationDue = this.isRotationDue(
                lastRotated,
                schedule.frequency,
            );

            if (force || rotationDue) {
                secretsToRotate.push({
                    ...secret,
                    schedule,
                });
            }
        }

        return secretsToRotate;
    }

    isRotationDue(lastRotated, frequency) {
        const ms = require("ms");
        const frequencyMs = ms(frequency);
        const timeSinceRotation = Date.now() - lastRotated.getTime();

        return timeSinceRotation >= frequencyMs;
    }

    async rotateSecret(secret) {
        console.log(`Rotating ${secret.key}...`);

        switch (secret.schedule.type) {
            case "database":
                await this.rotateDatabasePassword(secret);
                break;

            case "token":
                await this.rotateToken(secret);
                break;

            case "api":
                await this.rotateAPIKey(secret);
                break;

            case "encryption":
                await this.rotateEncryptionKey(secret);
                break;

            default:
                throw new Error(
                    `Unknown rotation type: ${secret.schedule.type}`,
                );
        }

        // Update metadata
        await this.dotenv.updateSecretMetadata(
            "my-app-production",
            "production",
            secret.key,
            {
                lastRotated: new Date().toISOString(),
                rotatedBy: "automated-rotation",
                previousValue: secret.value, // Encrypted reference
            },
        );
    }

    async rotateDatabasePassword(secret) {
        // Generate new password
        const newPassword = this.generateSecurePassword();

        // Update database user
        const db = require("pg");
        const client = new db.Client({
            connectionString: process.env.DATABASE_URL,
        });

        await client.connect();

        try {
            await client.query(
                `ALTER USER ${process.env.DB_USER} WITH PASSWORD $1`,
                [newPassword],
            );
        } finally {
            await client.end();
        }

        // Update secret in DotEnv
        const newConnectionString = process.env.DATABASE_URL.replace(
            /:([^@]+)@/,
            `:${newPassword}@`,
        );

        await this.dotenv.updateSecret(
            "my-app-production",
            "production",
            "DATABASE_URL",
            newConnectionString,
        );

        // Update AWS Secrets Manager
        await this.secretsManager
            .updateSecret({
                SecretId: "prod/database/password",
                SecretString: newPassword,
            })
            .promise();
    }

    async rotateToken(secret) {
        const newToken = crypto.randomBytes(32).toString("base64url");

        // Update in DotEnv with grace period
        await this.dotenv.updateSecret(
            "my-app-production",
            "production",
            secret.key,
            newToken,
            {
                gracePeriod: "1h",
                previousValue: secret.value,
            },
        );

        // Deploy with both old and new tokens
        await this.deployWithDualTokens(secret.key, secret.value, newToken);
    }

    async rotateAPIKey(secret) {
        // Implementation depends on the API provider
        console.log(`Rotating API key: ${secret.key}`);

        // Example for Stripe
        if (secret.key === "STRIPE_SECRET_KEY") {
            // Would use Stripe API to create new key
            // and update webhooks
        }
    }

    async rotateEncryptionKey(secret) {
        console.log(`Rotating encryption key: ${secret.key}`);

        // This requires re-encrypting all data
        // Usually done in phases:
        // 1. Add new key alongside old
        // 2. Start encrypting new data with new key
        // 3. Gradually re-encrypt old data
        // 4. Remove old key
    }

    generateSecurePassword() {
        const length = 32;
        const charset =
            "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*";
        let password = "";

        const randomBytes = crypto.randomBytes(length);
        for (let i = 0; i < length; i++) {
            password += charset[randomBytes[i] % charset.length];
        }

        return password;
    }

    async backupCurrentSecrets() {
        console.log("Creating secret backup...");

        const secrets = await this.dotenv.exportSecrets(
            "my-app-production",
            "production",
        );

        const backup = {
            timestamp: new Date().toISOString(),
            secrets: secrets,
            version: process.env.APP_VERSION,
        };

        // Store encrypted backup
        const encrypted = await this.encryptBackup(backup);

        // Save to S3
        const s3 = new AWS.S3();
        await s3
            .putObject({
                Bucket: "prod-secret-backups",
                Key: `backup-${backup.timestamp}.json.enc`,
                Body: encrypted,
                ServerSideEncryption: "AES256",
            })
            .promise();
    }

    async verifyServices() {
        console.log("Verifying services after rotation...");

        const healthChecks = [
            { name: "API", url: "https://api.example.com/health" },
            { name: "Database", url: "https://api.example.com/health/db" },
            { name: "Cache", url: "https://api.example.com/health/cache" },
        ];

        for (const check of healthChecks) {
            const response = await fetch(check.url);
            if (!response.ok) {
                throw new Error(`Health check failed for ${check.name}`);
            }
        }
    }
}

// Schedule rotation
if (require.main === module) {
    const rotation = new ProductionSecretRotation();

    // Run rotation
    rotation
        .rotateSecrets({
            force: process.argv.includes("--force"),
            dryRun: process.argv.includes("--dry-run"),
            notify: !process.argv.includes("--no-notify"),
        })
        .catch(console.error);
}

module.exports = ProductionSecretRotation;
```

### 2. Emergency Access

```javascript
// scripts/emergency-access.js
const { DotEnvClient } = require("@dotenv/sdk");
const speakeasy = require("speakeasy");

class EmergencyAccess {
    constructor() {
        this.dotenv = new DotEnvClient({
            apiKey: process.env.DOTENV_API_KEY,
        });
    }

    async grantEmergencyAccess(options) {
        const { userId, reason, duration = "2h", approvers = [] } = options;

        console.log("🚨 Emergency Access Request");
        console.log(`User: ${userId}`);
        console.log(`Reason: ${reason}`);
        console.log(`Duration: ${duration}`);

        // Require multiple approvals for production
        const requiredApprovals = 2;
        const approvals = await this.collectApprovals(
            approvers,
            requiredApprovals,
        );

        if (approvals.length < requiredApprovals) {
            throw new Error("Insufficient approvals for emergency access");
        }

        // Generate temporary credentials
        const credentials = await this.generateEmergencyCredentials(
            userId,
            duration,
        );

        // Log access grant
        await this.logEmergencyAccess({
            userId,
            reason,
            duration,
            approvals,
            credentials: credentials.id,
            grantedAt: new Date(),
        });

        // Send credentials securely
        await this.sendEmergencyCredentials(userId, credentials);

        // Schedule automatic revocation
        this.scheduleRevocation(credentials.id, duration);

        console.log("✅ Emergency access granted");

        return credentials;
    }

    async collectApprovals(approvers, required) {
        const approvals = [];

        for (const approver of approvers) {
            const otp = speakeasy.totp({
                secret: approver.mfaSecret,
                encoding: "base32",
            });

            console.log(`Awaiting approval from ${approver.email}...`);
            console.log(`OTP required: ${otp}`);

            // In real implementation, this would be async
            // via Slack, email, or approval system
            const approved = await this.requestApproval(approver);

            if (approved) {
                approvals.push({
                    approver: approver.email,
                    timestamp: new Date(),
                    ip: approver.ip,
                });
            }

            if (approvals.length >= required) break;
        }

        return approvals;
    }

    async generateEmergencyCredentials(userId, duration) {
        const credentials = {
            id: `emrg_${Date.now()}_${userId}`,
            apiKey: crypto.randomBytes(32).toString("hex"),
            permissions: ["secrets:read", "logs:read", "metrics:read"],
            restrictions: {
                environments: ["production"],
                readOnly: true,
                auditLog: true,
            },
            expiresAt: new Date(Date.now() + ms(duration)),
        };

        // Create in DotEnv
        await this.dotenv.createTemporaryAccess({
            project: "my-app-production",
            environment: "production",
            credentials,
        });

        return credentials;
    }

    async logEmergencyAccess(details) {
        // Log to multiple systems for redundancy

        // CloudWatch Logs
        const cloudwatch = new AWS.CloudWatchLogs();
        await cloudwatch
            .putLogEvents({
                logGroupName: "/aws/production/emergency-access",
                logStreamName: new Date().toISOString().split("T")[0],
                logEvents: [
                    {
                        timestamp: Date.now(),
                        message: JSON.stringify(details),
                    },
                ],
            })
            .promise();

        // Audit database
        const db = require("./db");
        await db.query(
            `INSERT INTO emergency_access_log 
       (user_id, reason, duration, approvals, credentials_id, granted_at)
       VALUES ($1, $2, $3, $4, $5, $6)`,
            [
                details.userId,
                details.reason,
                details.duration,
                JSON.stringify(details.approvals),
                details.credentials,
                details.grantedAt,
            ],
        );

        // Send to SIEM
        await this.sendToSIEM(details);
    }

    scheduleRevocation(credentialId, duration) {
        setTimeout(async () => {
            try {
                await this.dotenv.revokeAccess(credentialId);
                console.log(`Revoked emergency access: ${credentialId}`);
            } catch (error) {
                console.error(`Failed to revoke ${credentialId}:`, error);
                // Alert security team
            }
        }, ms(duration));
    }
}
```

## Monitoring and Observability

### 1. Production Monitoring Stack

```javascript
// monitoring/production-monitor.js
const prometheus = require("prom-client");
const { StatsD } = require("node-statsd");
const Sentry = require("@sentry/node");

class ProductionMonitor {
    constructor() {
        // Initialize Prometheus
        this.register = new prometheus.Registry();
        prometheus.collectDefaultMetrics({ register: this.register });

        // Custom metrics
        this.httpDuration = new prometheus.Histogram({
            name: "http_request_duration_seconds",
            help: "Duration of HTTP requests in seconds",
            labelNames: ["method", "route", "status_code"],
            buckets: [0.1, 0.5, 1, 2, 5],
        });

        this.activeUsers = new prometheus.Gauge({
            name: "active_users_total",
            help: "Total number of active users",
            labelNames: ["tier"],
        });

        this.paymentProcessed = new prometheus.Counter({
            name: "payments_processed_total",
            help: "Total number of payments processed",
            labelNames: ["status", "payment_method"],
        });

        this.secretAccess = new prometheus.Counter({
            name: "secret_access_total",
            help: "Total number of secret accesses",
            labelNames: ["secret_key", "environment", "result"],
        });

        // Register metrics
        this.register.registerMetric(this.httpDuration);
        this.register.registerMetric(this.activeUsers);
        this.register.registerMetric(this.paymentProcessed);
        this.register.registerMetric(this.secretAccess);

        // Initialize StatsD
        this.statsd = new StatsD({
            host: process.env.STATSD_HOST || "localhost",
            port: 8125,
            prefix: "production.",
        });

        // Initialize Sentry
        Sentry.init({
            dsn: process.env.SENTRY_DSN,
            environment: "production",
            tracesSampleRate: 0.1,
            beforeSend(event) {
                // Scrub sensitive data
                if (event.request) {
                    delete event.request.cookies;
                    delete event.request.headers;
                }
                return event;
            },
        });
    }

    // Express middleware
    requestMetrics() {
        return (req, res, next) => {
            const start = Date.now();

            res.on("finish", () => {
                const duration = (Date.now() - start) / 1000;

                // Prometheus metrics
                this.httpDuration
                    .labels(
                        req.method,
                        req.route?.path || "unknown",
                        res.statusCode,
                    )
                    .observe(duration);

                // StatsD metrics
                this.statsd.timing("request.duration", duration);
                this.statsd.increment(`request.status.${res.statusCode}`);

                // Log slow requests
                if (duration > 1) {
                    console.warn("Slow request:", {
                        method: req.method,
                        path: req.path,
                        duration: `${duration}s`,
                        status: res.statusCode,
                    });
                }
            });

            next();
        };
    }

    // Business metrics
    recordPayment(status, method, amount) {
        this.paymentProcessed.labels(status, method).inc();
        this.statsd.increment(`payment.${status}`);
        this.statsd.gauge("payment.amount", amount);

        if (status === "failed") {
            Sentry.captureMessage("Payment failed", "warning", {
                extra: { method, amount },
            });
        }
    }

    // Security metrics
    recordSecretAccess(secretKey, result) {
        this.secretAccess.labels(secretKey, "production", result).inc();

        if (result === "denied") {
            Sentry.captureMessage("Secret access denied", "warning", {
                extra: { secretKey },
            });
        }
    }

    // Real-time alerts
    checkThresholds() {
        setInterval(async () => {
            // Check error rate
            const errorRate = await this.getErrorRate();
            if (errorRate > 0.05) {
                // 5% threshold
                this.alert("High error rate", {
                    rate: errorRate,
                    severity: "critical",
                });
            }

            // Check response time
            const p95Latency = await this.getP95Latency();
            if (p95Latency > 1000) {
                // 1s threshold
                this.alert("High latency", {
                    p95: p95Latency,
                    severity: "warning",
                });
            }

            // Check memory usage
            const memoryUsage = process.memoryUsage();
            const heapUsed = memoryUsage.heapUsed / memoryUsage.heapTotal;
            if (heapUsed > 0.9) {
                // 90% threshold
                this.alert("High memory usage", {
                    heapUsed: `${(heapUsed * 100).toFixed(2)}%`,
                    severity: "warning",
                });
            }
        }, 60000); // Check every minute
    }

    async alert(message, details) {
        console.error(`🚨 Alert: ${message}`, details);

        // Send to PagerDuty
        if (details.severity === "critical") {
            await this.sendPagerDuty(message, details);
        }

        // Send to Slack
        await this.sendSlack(message, details);

        // Log to CloudWatch
        await this.logToCloudWatch("alerts", {
            message,
            ...details,
            timestamp: new Date(),
        });
    }
}

module.exports = ProductionMonitor;
```

### 2. Log Aggregation

```javascript
// logging/production-logger.js
const winston = require("winston");
const { ElasticsearchTransport } = require("winston-elasticsearch");

class ProductionLogger {
    constructor() {
        this.logger = winston.createLogger({
            level: process.env.LOG_LEVEL || "info",
            format: winston.format.combine(
                winston.format.timestamp(),
                winston.format.errors({ stack: true }),
                winston.format.json(),
            ),
            defaultMeta: {
                service: "api",
                environment: "production",
                version: process.env.APP_VERSION,
            },
            transports: [
                // Console (for container logs)
                new winston.transports.Console({
                    format: winston.format.combine(
                        winston.format.colorize(),
                        winston.format.simple(),
                    ),
                }),

                // Elasticsearch
                new ElasticsearchTransport({
                    level: "info",
                    clientOpts: {
                        node: process.env.ELASTICSEARCH_URL,
                        auth: {
                            username: process.env.ELASTICSEARCH_USER,
                            password: process.env.ELASTICSEARCH_PASS,
                        },
                    },
                    index: "production-logs",
                    dataStream: true,
                }),

                // File (for backup)
                new winston.transports.File({
                    filename: "/var/log/app/production.log",
                    maxsize: 100 * 1024 * 1024, // 100MB
                    maxFiles: 10,
                }),
            ],
        });

        // Add request ID to all logs
        this.logger.add(
            new winston.transports.Console({
                format: winston.format.printf((info) => {
                    const requestId = info.requestId || "system";
                    return `[${info.timestamp}] [${requestId}] ${info.level}: ${info.message}`;
                }),
            }),
        );
    }

    // Structured logging methods
    logRequest(req, res, duration) {
        this.logger.info("HTTP Request", {
            method: req.method,
            path: req.path,
            statusCode: res.statusCode,
            duration,
            userAgent: req.get("user-agent"),
            ip: req.ip,
            requestId: req.id,
        });
    }

    logError(error, context = {}) {
        this.logger.error("Application Error", {
            error: {
                message: error.message,
                stack: error.stack,
                code: error.code,
            },
            ...context,
        });
    }

    logSecurityEvent(event, details) {
        this.logger.warn("Security Event", {
            event,
            ...details,
            timestamp: new Date(),
        });
    }

    logBusinessEvent(event, details) {
        this.logger.info("Business Event", {
            event,
            ...details,
        });
    }
}

module.exports = ProductionLogger;
```

## Disaster Recovery

### 1. Backup Strategy

```javascript
// scripts/production-backup.js
const AWS = require("aws-sdk");
const { execSync } = require("child_process");

class ProductionBackup {
    constructor() {
        this.s3 = new AWS.S3();
        this.rds = new AWS.RDS();
        this.backupBucket = "prod-backups";
    }

    async performFullBackup() {
        console.log("Starting production backup...");

        const backupId = `backup-${Date.now()}`;

        try {
            // 1. Database backup
            await this.backupDatabase(backupId);

            // 2. Application state
            await this.backupApplicationState(backupId);

            // 3. Secrets backup
            await this.backupSecrets(backupId);

            // 4. Configuration backup
            await this.backupConfiguration(backupId);

            // 5. Verify backup
            await this.verifyBackup(backupId);

            // 6. Update recovery point
            await this.updateRecoveryPoint(backupId);

            console.log("✅ Backup completed successfully");

            return backupId;
        } catch (error) {
            console.error("❌ Backup failed:", error);
            await this.cleanupFailedBackup(backupId);
            throw error;
        }
    }

    async backupDatabase(backupId) {
        console.log("Backing up database...");

        // Create RDS snapshot
        const snapshotId = `prod-snapshot-${backupId}`;

        await this.rds
            .createDBSnapshot({
                DBSnapshotIdentifier: snapshotId,
                DBInstanceIdentifier: "prod-database",
                Tags: [
                    { Key: "BackupId", Value: backupId },
                    { Key: "Type", Value: "scheduled" },
                ],
            })
            .promise();

        // Wait for snapshot completion
        await this.rds
            .waitFor("dBSnapshotCompleted", {
                DBSnapshotIdentifier: snapshotId,
            })
            .promise();

        // Export to S3 for long-term storage
        await this.rds
            .startExportTask({
                ExportTaskIdentifier: `export-${backupId}`,
                SourceArn: `arn:aws:rds:us-east-1:123456789012:snapshot:${snapshotId}`,
                S3BucketName: this.backupBucket,
                S3Prefix: `database/${backupId}/`,
                IamRoleArn: "arn:aws:iam::123456789012:role/rds-export-role",
                KmsKeyId: process.env.BACKUP_KMS_KEY,
            })
            .promise();
    }

    async backupApplicationState(backupId) {
        console.log("Backing up application state...");

        // Backup Redis
        const redis = require("redis").createClient({
            url: process.env.REDIS_URL,
        });

        await redis.connect();
        await redis.bgsave();

        // Wait for save to complete
        let saving = true;
        while (saving) {
            const info = await redis.info("persistence");
            saving = info.includes("rdb_bgsave_in_progress:1");
            if (saving) await new Promise((r) => setTimeout(r, 1000));
        }

        // Upload Redis dump
        const rdbPath = "/var/lib/redis/dump.rdb";
        await this.uploadToS3(
            rdbPath,
            `application-state/${backupId}/redis-dump.rdb`,
        );

        // Backup uploaded files
        execSync(`tar -czf /tmp/uploads-${backupId}.tar.gz /var/app/uploads/`);
        await this.uploadToS3(
            `/tmp/uploads-${backupId}.tar.gz`,
            `application-state/${backupId}/uploads.tar.gz`,
        );
    }

    async backupSecrets(backupId) {
        console.log("Backing up secrets...");

        const dotenv = new DotEnvClient({
            apiKey: process.env.DOTENV_API_KEY,
        });

        // Export all production secrets
        const secrets = await dotenv.exportSecrets(
            "my-app-production",
            "production",
        );

        // Encrypt with backup key
        const encrypted = await this.encryptData(
            JSON.stringify(secrets),
            process.env.BACKUP_ENCRYPTION_KEY,
        );

        // Upload to S3
        await this.s3
            .putObject({
                Bucket: this.backupBucket,
                Key: `secrets/${backupId}/production-secrets.enc`,
                Body: encrypted,
                ServerSideEncryption: "aws:kms",
                SSEKMSKeyId: process.env.BACKUP_KMS_KEY,
            })
            .promise();
    }

    async verifyBackup(backupId) {
        console.log("Verifying backup integrity...");

        // List all backup components
        const components = [
            `database/${backupId}/`,
            `application-state/${backupId}/`,
            `secrets/${backupId}/`,
            `configuration/${backupId}/`,
        ];

        for (const component of components) {
            const objects = await this.s3
                .listObjectsV2({
                    Bucket: this.backupBucket,
                    Prefix: component,
                })
                .promise();

            if (!objects.Contents || objects.Contents.length === 0) {
                throw new Error(`Backup component missing: ${component}`);
            }
        }

        // Verify checksums
        // Implementation depends on your checksum strategy
    }

    async uploadToS3(localPath, s3Key) {
        const fileStream = require("fs").createReadStream(localPath);

        await this.s3
            .upload({
                Bucket: this.backupBucket,
                Key: s3Key,
                Body: fileStream,
                ServerSideEncryption: "AES256",
            })
            .promise();
    }
}

// Schedule daily backups
if (require.main === module) {
    const backup = new ProductionBackup();
    backup.performFullBackup().catch(console.error);
}

module.exports = ProductionBackup;
```

### 2. Recovery Procedures

```javascript
// scripts/disaster-recovery.js
class DisasterRecovery {
    async initiateRecovery(backupId, options = {}) {
        const { targetEnvironment = "dr", verifyOnly = false } = options;

        console.log("🚨 Initiating disaster recovery...");
        console.log(`Backup ID: ${backupId}`);
        console.log(`Target: ${targetEnvironment}`);

        if (verifyOnly) {
            console.log("VERIFY ONLY MODE - No changes will be made");
        }

        try {
            // 1. Verify backup exists
            await this.verifyBackupExists(backupId);

            // 2. Restore database
            await this.restoreDatabase(backupId, targetEnvironment);

            // 3. Restore application state
            await this.restoreApplicationState(backupId, targetEnvironment);

            // 4. Restore secrets
            await this.restoreSecrets(backupId, targetEnvironment);

            // 5. Update DNS
            if (!verifyOnly) {
                await this.updateDNS(targetEnvironment);
            }

            // 6. Verify services
            await this.verifyServices(targetEnvironment);

            console.log("✅ Disaster recovery completed successfully");
        } catch (error) {
            console.error("❌ Recovery failed:", error);
            throw error;
        }
    }
}
```

## Best Practices

### 1. Production Checklist

```yaml
# production-checklist.yaml
pre_deployment:
    - All tests passing
    - Security scan completed
    - Performance benchmarks met
    - Documentation updated
    - Rollback plan prepared

deployment:
    - Use blue-green or canary deployment
    - Monitor error rates during deployment
    - Have team members on standby
    - Keep deployment window small
    - Document deployment steps

post_deployment:
    - Run smoke tests
    - Monitor metrics for 30 minutes
    - Check error logs
    - Verify all services healthy
    - Update status page

security:
    - All secrets in DotEnv
    - MFA required for production access
    - Audit logging enabled
    - Regular security scans
    - Incident response plan ready
```

### 2. Production Standards

```javascript
// config/production-standards.js
module.exports = {
    // Deployment requirements
    deployment: {
        strategy: "blue-green",
        canaryPercentage: 10,
        canaryDuration: "10m",
        rollbackThreshold: {
            errorRate: 0.05,
            latencyP95: 1000,
        },
    },

    // Monitoring requirements
    monitoring: {
        metrics: ["latency", "errors", "saturation", "traffic"],
        alerting: {
            channels: ["pagerduty", "slack"],
            escalation: ["on-call-primary", "on-call-secondary", "team-lead"],
        },
        sla: {
            availability: 0.999, // 99.9%
            latencyP95: 500, // 500ms
            errorRate: 0.01, // 1%
        },
    },

    // Security requirements
    security: {
        mfaRequired: true,
        secretRotation: {
            database: "30d",
            apiKeys: "90d",
            certificates: "365d",
        },
        accessLogging: true,
        encryptionAtRest: true,
        encryptionInTransit: true,
    },

    // Backup requirements
    backup: {
        frequency: {
            full: "daily",
            incremental: "hourly",
        },
        retention: {
            daily: "7d",
            weekly: "4w",
            monthly: "12m",
        },
        testing: "weekly",
    },
};
```

## Troubleshooting

### Production Issues

1. **High Latency**

    ```javascript
    // Check slow queries
    SELECT query, mean_time, calls
    FROM pg_stat_statements
    ORDER BY mean_time DESC
    LIMIT 10;
    ```

2. **Memory Leaks**

    ```javascript
    // Profile memory usage
    const heapdump = require("heapdump");
    heapdump.writeSnapshot(`/tmp/heap-${Date.now()}.heapsnapshot`);
    ```

3. **Secret Access Failures**

    ```bash
    # Check DotEnv connectivity
    dotenv status --project my-app-production

    # Verify secret exists
    dotenv get DATABASE_URL --environment production
    ```

## Resources

- [Production Best Practices](https://12factor.net/)
- [Zero Downtime Deployments](https://martinfowler.com/bliki/BlueGreenDeployment.html)
- [Disaster Recovery Planning](https://aws.amazon.com/disaster-recovery/)
- [DotEnv Security Model](/documentation/v1/core-concepts/security-model)

## Next Steps

- [Set up production infrastructure](#production-architecture)
- [Configure deployment pipeline](#deployment-pipeline)
- [Implement secret rotation](#secret-management)
- [Enable monitoring](#monitoring-and-observability)
