---
title: Advanced CLI Usage
slug: advanced-usage
order: 5
tags: [cli, advanced, power-user]
---

# Advanced CLI Usage

This guide covers advanced features and power-user techniques for the DotEnv CLI.

## Advanced Secret Management

### Bulk Operations

Perform operations on multiple secrets efficiently:

```bash
# Set multiple secrets at once
dotenv set \
  DATABASE_URL="postgresql://..." \
  REDIS_URL="redis://..." \
  API_KEY="sk_live_..."

# Delete multiple secrets
dotenv delete KEY1 KEY2 KEY3

# Export specific keys
dotenv export --keys DATABASE_URL,REDIS_URL,API_KEY

# Import from multiple sources
cat prod.env staging.env | dotenv import -
```

### Secret Templates

Use templates for dynamic secret values:

```bash
# Create template
dotenv templates create database-url \
  --template "postgresql://{{user}}:{{password}}@{{host}}/{{database}}"

# Use template
dotenv set DATABASE_URL --template database-url \
  --vars user=admin,password=$DB_PASS,host=localhost,database=myapp

# List templates
dotenv templates list

# Export template
dotenv templates export database-url > db-template.yml
```

### Conditional Operations

Execute operations based on conditions:

```bash
# Set only if not exists
dotenv set API_KEY="new_value" --if-not-exists

# Update only if older than
dotenv set CERT="new_cert" --if-older-than 30d

# Delete if matches pattern
dotenv delete --pattern "TEMP_*" --if-matches

# Rotate based on age
dotenv rotate --all --older-than 90d
```

### Secret Validation

Validate secrets before use:

```bash
# Validate format
dotenv validate DATABASE_URL --format postgres-uri

# Check requirements
dotenv validate --required-from .dotenv.yml

# Test connections
dotenv validate DATABASE_URL --test-connection

# Validate all
dotenv validate --all --stop-on-error
```

## Environment Management

### Environment Cloning

Clone environments with modifications:

```bash
# Basic clone
dotenv env clone production production-backup

# Clone with modifications
dotenv env clone staging qa \
  --override DATABASE_URL="postgresql://qa-db/app" \
  --exclude PROD_ONLY_KEY

# Partial clone
dotenv env clone production hotfix \
  --only "API_*,DATABASE_*"
```

### Environment Comparison

Compare environments to find differences:

```bash
# Basic comparison
dotenv env diff development production

# Detailed diff
dotenv env diff staging production --detailed

# Export diff
dotenv env diff dev prod --format json > env-diff.json

# Three-way diff
dotenv env diff dev staging prod
```

### Environment Sync

Synchronize environments:

```bash
# Sync from source
dotenv env sync production staging \
  --only-missing \
  --dry-run

# Bidirectional sync
dotenv env sync dev staging \
  --bidirectional \
  --resolve-conflicts newer

# Sync with transform
dotenv env sync prod dev \
  --transform 's/prod/dev/g'
```

## Scripting and Automation

### Batch Processing

Process multiple projects/environments:

```bash
#!/bin/bash
# deploy-all.sh

# Deploy to all environments
for env in dev staging prod; do
  echo "Deploying to $env..."
  dotenv env use $env
  dotenv pull
  ./deploy.sh
done

# Update secret across all projects
for project in $(dotenv projects list --format json | jq -r '.[].name'); do
  dotenv set API_VERSION="v2" --project "$project"
done
```

### Pipeline Integration

Integrate with Unix pipes:

```bash
# Filter and process
dotenv secrets list --format json | \
  jq '.[] | select(.key | startswith("API_"))' | \
  dotenv import -

# Chain operations
dotenv export --format json | \
  jq 'map_values(. | @base64)' | \
  kubectl create secret generic app-secrets --from-file=-

# Parallel processing
dotenv projects list --format tsv | \
  parallel -j4 'dotenv pull --project {}'
```

### Custom Scripts

Create reusable scripts:

```bash
#!/bin/bash
# rotate-all-keys.sh

set -euo pipefail

# Configuration
OLDER_THAN=${1:-90d}
PROJECTS=${2:-$(dotenv projects list --format json | jq -r '.[].name')}

# Rotate function
rotate_project_keys() {
  local project=$1
  echo "Rotating keys for $project older than $OLDER_THAN..."

  dotenv secrets list \
    --project "$project" \
    --format json | \
  jq -r ".[] | select(.updated < \"$(date -d "$OLDER_THAN ago" -Iseconds)\") | .key" | \
  while read -r key; do
    echo "  Rotating $key..."
    dotenv rotate "$key" --project "$project" --generate
  done
}

# Process all projects
for project in $PROJECTS; do
  rotate_project_keys "$project"
done
```

## Advanced Configuration

### Dynamic Configuration

Use dynamic configuration values:

```yaml
# .dotenv.yml
project: ${DOTENV_PROJECT:-my-app}
environment: ${DOTENV_ENV:-development}

required:
    - DATABASE_URL
    - API_KEY

environments:
    development:
        optional:
            DEBUG: "true"

    production:
        required:
            - SSL_CERT
            - SSL_KEY

scripts:
    deploy: |
        if [ "$DOTENV_ENV" = "production" ]; then
          dotenv validate --all
        fi
        dotenv pull
        ./deploy.sh
```

### Plugin System

Extend CLI with plugins:

```bash
# Install plugin
dotenv plugins install dotenv-vault

# List plugins
dotenv plugins list

# Configure plugin
dotenv config set plugins.vault.endpoint "https://vault.company.com"

# Use plugin command
dotenv vault seal my-secret
```

### Hook System

Advanced hook configuration:

```yaml
# .dotenv.yml
hooks:
    pre_pull: |
        # Backup current env
        cp .env .env.backup

    post_pull: |
        # Validate changes
        dotenv validate --all

        # Run tests
        npm test

    on_error: |
        # Restore backup
        mv .env.backup .env

        # Notify team
        slack-notify "Pull failed: $DOTENV_ERROR"

    on_change: |
        # Restart services
        if systemctl is-active myapp; then
          systemctl restart myapp
        fi
```

## Security Hardening

### Key Management

Advanced encryption key handling:

```bash
# Generate strong key
dotenv keys generate \
  --algorithm aes-256-gcm \
  --key-size 32

# Rotate with grace period
dotenv keys rotate \
  --grace-period 24h \
  --notify-consumers

# Export for backup
dotenv keys export \
  --format pem \
  --encrypt-with-passphrase > keys-backup.pem

# Hardware security module
dotenv config set \
  encryption.hsm.enabled true \
  encryption.hsm.slot 1
```

### Access Control

Fine-grained access control:

```bash
# Create restricted token
dotenv tokens create \
  --name "ci-readonly" \
  --scopes "secrets:read" \
  --projects "api,web" \
  --expires 90d

# IP restrictions
dotenv tokens update ci-token \
  --allowed-ips "10.0.0.0/8,192.168.1.0/24"

# Time-based access
dotenv tokens create \
  --name "temp-access" \
  --valid-hours "09:00-17:00" \
  --valid-days "Mon-Fri"
```

### Audit and Compliance

Track and audit all operations:

```bash
# Enable detailed audit
dotenv config set audit.level detailed

# Query audit logs
dotenv audit query \
  --user alice@example.com \
  --action "secret:write" \
  --days 30

# Export for compliance
dotenv audit export \
  --format csv \
  --start 2024-01-01 \
  --end 2024-12-31 > audit-2024.csv

# Real-time monitoring
dotenv audit stream --follow
```

## Performance Optimization

### Caching Strategies

Optimize performance with caching:

```bash
# Enable aggressive caching
dotenv config set \
  cache.ttl 3600 \
  cache.size "1GB" \
  cache.strategy "lru"

# Warm cache
dotenv cache warm --all-projects

# Cache statistics
dotenv cache stats

# Selective cache clear
dotenv cache clear --project my-app
```

### Parallel Operations

Execute operations in parallel:

```bash
# Parallel pull
dotenv pull \
  --parallel \
  --max-workers 10 \
  --projects "api,web,mobile"

# Concurrent validation
dotenv validate \
  --all \
  --parallel \
  --fail-fast

# Batch API calls
dotenv config set \
  api.batch_size 100 \
  api.concurrent_requests 5
```

### Network Optimization

Optimize network usage:

```bash
# Compression
dotenv config set \
  network.compression gzip \
  network.compression_level 9

# Connection pooling
dotenv config set \
  network.pool_size 20 \
  network.keepalive true

# Regional endpoints
dotenv config set \
  api.endpoint "https://eu.api.dotenv.cloud" \
  api.fallback "https://api.dotenv.cloud"
```

## Advanced Workflows

### GitOps Integration

Integrate with GitOps workflows:

```yaml
# .gitops/dotenv-sync.yml
apiVersion: batch/v1
kind: CronJob
metadata:
    name: dotenv-sync
spec:
    schedule: "*/15 * * * *"
    jobTemplate:
        spec:
            template:
                spec:
                    containers:
                        - name: sync
                          image: dotenv/cli:latest
                          command:
                              - sh
                              - -c
                              - |
                                  dotenv pull --project $PROJECT --environment $ENV
                                  kubectl create secret generic app-secrets \
                                    --from-env-file=.env \
                                    --dry-run=client -o yaml | \
                                    kubectl apply -f -
```

### Multi-Cloud Sync

Sync across cloud providers:

```bash
#!/bin/bash
# sync-to-clouds.sh

# Pull from DotEnv
dotenv pull --format json > secrets.json

# Sync to AWS
cat secrets.json | jq -r 'to_entries | .[] |
  "aws secretsmanager put-secret-value \
   --secret-id /app/\(.key) \
   --secret-string \(.value | @sh)"' | bash

# Sync to Azure
cat secrets.json | jq -r 'to_entries | .[] |
  "az keyvault secret set \
   --vault-name myvault \
   --name \(.key) \
   --value \(.value | @sh)"' | bash

# Sync to GCP
cat secrets.json | jq -r 'to_entries | .[] |
  "echo -n \(.value | @sh) | \
   gcloud secrets create \(.key) \
   --data-file=-"' | bash
```

### Disaster Recovery

Implement disaster recovery:

```bash
# Backup script
#!/bin/bash
# backup-secrets.sh

DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="backups/$DATE"
mkdir -p "$BACKUP_DIR"

# Export all projects
for project in $(dotenv projects list --format json | jq -r '.[].name'); do
  echo "Backing up $project..."

  # Export all environments
  for env in $(dotenv env list --project "$project" --format json | jq -r '.[].name'); do
    dotenv export \
      --project "$project" \
      --environment "$env" \
      --format json \
      --include-metadata > "$BACKUP_DIR/${project}-${env}.json"
  done
done

# Encrypt backup
tar czf - "$BACKUP_DIR" | \
  openssl enc -aes-256-cbc -salt -out "backup-$DATE.tar.gz.enc"

# Upload to S3
aws s3 cp "backup-$DATE.tar.gz.enc" "s3://backups/dotenv/"
```

## Integration Patterns

### Kubernetes Integration

Native Kubernetes integration:

```bash
# Install operator
kubectl apply -f https://dotenv.cloud/k8s/operator.yaml

# Create SecretSync
cat <<EOF | kubectl apply -f -
apiVersion: dotenv.cloud/v1
kind: SecretSync
metadata:
  name: app-secrets
spec:
  project: my-app
  environment: production
  target:
    name: app-secrets
    namespace: default
  refresh: 300
EOF

# Watch sync status
kubectl get secretsync -w
```

### Service Mesh

Integrate with service meshes:

```bash
# Istio integration
dotenv secrets pull --format json | \
  jq '{apiVersion:"v1",kind:"Secret",metadata:{name:"app-secrets"},data:
    (. | to_entries | map({key:.key,value:(.value|@base64)}) | from_entries)}' | \
  kubectl apply -f -

# Consul integration
dotenv secrets list --format json | \
  jq -r 'to_entries | .[] | "consul kv put app/\(.key) \(.value | @sh)"' | bash

# Linkerd integration
dotenv export --format yaml | \
  linkerd inject - | \
  kubectl apply -f -
```

### Terraform Provider

Use with Terraform:

```hcl
terraform {
  required_providers {
    dotenv = {
      source  = "dotenv/dotenv"
      version = "~> 1.0"
    }
  }
}

provider "dotenv" {
  api_key = var.dotenv_api_key
}

data "dotenv_secrets" "app" {
  project     = "my-app"
  environment = "production"
}

resource "aws_ecs_task_definition" "app" {
  family = "my-app"

  container_definitions = jsonencode([
    {
      name = "app"
      environment = [
        for k, v in data.dotenv_secrets.app.secrets : {
          name  = k
          value = v
        }
      ]
    }
  ])
}
```

## Monitoring and Observability

### Metrics Export

Export metrics for monitoring:

```bash
# Prometheus format
dotenv metrics \
  --format prometheus \
  --port 9090

# CloudWatch
dotenv metrics push \
  --backend cloudwatch \
  --namespace DotEnv/CLI

# Custom metrics
dotenv metrics custom \
  --name "secrets_accessed" \
  --value 42 \
  --tags "project=my-app,env=prod"
```

### Tracing

Enable distributed tracing:

```bash
# OpenTelemetry
export OTEL_EXPORTER_OTLP_ENDPOINT="http://collector:4317"
export OTEL_SERVICE_NAME="dotenv-cli"
dotenv config set telemetry.enabled true

# Jaeger
export JAEGER_AGENT_HOST="localhost"
export JAEGER_AGENT_PORT="6831"
dotenv config set telemetry.backend jaeger

# Custom spans
dotenv --trace-context "trace-id=123,parent-id=456" pull
```

## Next Steps

- [Scripting Guide](./scripting) - Automation examples
- [Troubleshooting](./troubleshooting) - Common issues
- [API Integration](/documentation/v1/api/overview) - Direct API usage
- [Security Best Practices](/documentation/v1/security/best-practices) - Security guide
