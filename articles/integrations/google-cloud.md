---
title: Google Cloud Integration
slug: google-cloud
order: 11
tags: [integrations, google-cloud, gcp, cloud]
---

# Google Cloud Integration

Integrate DotEnv with Google Cloud Platform to manage secrets across your GCP infrastructure. This guide covers integration with various GCP services and deployment patterns.

## Overview

The DotEnv Google Cloud integration provides:

- Secret Manager synchronization
- Cloud Run integration
- Cloud Functions configuration
- GKE secret management
- Cloud Build integration
- Multi-project secret sharing

## Installation

### 1. Cloud Functions

Install in Cloud Functions:

```javascript
// package.json
{
  "dependencies": {
    "@google-cloud/secret-manager": "^5.0.0"
  },
  "scripts": {
    "gcp-build": "npm run install-dotenv && npm run load-secrets",
    "install-dotenv": "curl -fsSL https://cli.dotenv.cloud/install.sh | sh",
    "load-secrets": "PATH=$HOME/.dotenv/bin:$PATH dotenv pull $ENVIRONMENT --project $PROJECT_NAME"
  }
}
```

### 2. Cloud Run

Add to Dockerfile:

```dockerfile
FROM gcr.io/distroless/nodejs18-debian11

# Install DotEnv CLI in build stage
FROM node:18-alpine AS builder
RUN curl -fsSL https://cli.dotenv.cloud/install.sh | sh
ENV PATH="/root/.dotenv/bin:$PATH"

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

# Load secrets during build
ARG DOTENV_API_KEY
ARG DOTENV_PROJECT
ARG DOTENV_ENVIRONMENT
RUN dotenv auth login --api-key $DOTENV_API_KEY && \
    dotenv pull $DOTENV_ENVIRONMENT --project $DOTENV_PROJECT

# Production stage
FROM gcr.io/distroless/nodejs18-debian11
WORKDIR /app
COPY --from=builder /app .
EXPOSE 8080
CMD ["index.js"]
```

### 3. Compute Engine

Startup script:

```bash
#!/bin/bash
# Install DotEnv
curl -fsSL https://cli.dotenv.cloud/install.sh | sh
export PATH="$HOME/.dotenv/bin:$PATH"

# Get API key from metadata
DOTENV_API_KEY=$(curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/attributes/dotenv-api-key)

# Load secrets
dotenv auth login --api-key $DOTENV_API_KEY
dotenv pull production --project my-app

# Start application
systemctl start my-app
```

## Secret Manager Integration

### Sync to Secret Manager

```python
# scripts/sync_to_gcp.py
from google.cloud import secretmanager
import subprocess
import json
import os

def sync_dotenv_to_gcp():
    """Sync DotEnv secrets to Google Secret Manager."""

    # Initialize client
    client = secretmanager.SecretManagerServiceClient()
    project_id = os.environ.get('GCP_PROJECT')
    parent = f"projects/{project_id}"

    # Get secrets from DotEnv
    dotenv_project = os.environ.get('DOTENV_PROJECT', 'my-app')
    environment = os.environ.get('DOTENV_ENVIRONMENT', 'production')

    result = subprocess.run(
        ['dotenv', 'pull', environment, '--project', dotenv_project, '--format', 'json'],
        capture_output=True,
        text=True
    )

    if result.returncode != 0:
        raise Exception(f"Failed to pull secrets: {result.stderr}")

    secrets = json.loads(result.stdout)

    # Sync each secret
    for key, value in secrets.items():
        secret_id = f"dotenv-{dotenv_project}-{key}".lower().replace('_', '-')

        try:
            # Create secret
            secret = client.create_secret(
                request={
                    "parent": parent,
                    "secret_id": secret_id,
                    "secret": {
                        "replication": {
                            "automatic": {}
                        },
                        "labels": {
                            "managed-by": "dotenv",
                            "project": dotenv_project,
                            "environment": environment
                        }
                    }
                }
            )
            print(f"Created secret: {secret_id}")
        except Exception as e:
            if "already exists" in str(e):
                secret_name = f"{parent}/secrets/{secret_id}"
            else:
                raise

        # Add secret version
        secret_name = f"{parent}/secrets/{secret_id}"
        client.add_secret_version(
            request={
                "parent": secret_name,
                "payload": {
                    "data": value.encode('UTF-8')
                }
            }
        )
        print(f"Updated secret value: {secret_id}")

if __name__ == '__main__':
    sync_dotenv_to_gcp()
```

### Load from Secret Manager

```javascript
// utils/secrets.js
const { SecretManagerServiceClient } = require("@google-cloud/secret-manager");

async function loadSecretsFromGCP() {
    const client = new SecretManagerServiceClient();
    const projectId = process.env.GCP_PROJECT;

    // List all secrets with dotenv label
    const [secrets] = await client.listSecrets({
        parent: `projects/${projectId}`,
        filter: "labels.managed-by=dotenv",
    });

    // Load each secret
    for (const secret of secrets) {
        const [version] = await client.accessSecretVersion({
            name: `${secret.name}/versions/latest`,
        });

        const payload = version.payload.data.toString("utf8");
        const key = secret.name
            .split("/")
            .pop()
            .replace(/^dotenv-[^-]+-/, "")
            .replace(/-/g, "_")
            .toUpperCase();

        process.env[key] = payload;
    }

    console.log(`Loaded ${secrets.length} secrets from GCP`);
}

module.exports = { loadSecretsFromGCP };
```

## Cloud Run

### Service Configuration

Deploy with secrets:

```yaml
# service.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
    name: my-app
    annotations:
        run.googleapis.com/launch-stage: GA
spec:
    template:
        metadata:
            annotations:
                run.googleapis.com/execution-environment: gen2
                run.googleapis.com/cpu-throttling: "false"
        spec:
            serviceAccountName: my-app-sa
            containers:
                - image: gcr.io/my-project/my-app
                  ports:
                      - containerPort: 8080
                  env:
                      - name: DOTENV_PROJECT
                        value: my-app
                      - name: DOTENV_ENVIRONMENT
                        value: production
                      - name: DOTENV_API_KEY
                        valueFrom:
                            secretKeyRef:
                                name: dotenv-api-key
                                key: latest
                  resources:
                      limits:
                          cpu: "2"
                          memory: "2Gi"
```

### Deployment Script

```bash
#!/bin/bash
# deploy.sh

# Build and push image
gcloud builds submit --tag gcr.io/$PROJECT_ID/$SERVICE_NAME

# Deploy with secrets
gcloud run deploy $SERVICE_NAME \
  --image gcr.io/$PROJECT_ID/$SERVICE_NAME \
  --platform managed \
  --region $REGION \
  --set-env-vars DOTENV_PROJECT=$SERVICE_NAME \
  --set-env-vars DOTENV_ENVIRONMENT=$ENVIRONMENT \
  --set-secrets DOTENV_API_KEY=dotenv-api-key:latest \
  --service-account $SERVICE_ACCOUNT
```

## Cloud Functions

### HTTP Function

```javascript
// index.js
const functions = require("@google-cloud/functions-framework");
const { execSync } = require("child_process");

// Load secrets at cold start
let secretsLoaded = false;

async function loadSecrets() {
    if (secretsLoaded) return;

    try {
        // Get API key from environment
        const apiKey = process.env.DOTENV_API_KEY;
        if (!apiKey) throw new Error("No API key");

        // Load secrets
        execSync(
            `dotenv pull ${process.env.DOTENV_ENVIRONMENT} --project ${process.env.DOTENV_PROJECT}`,
            { stdio: "inherit" },
        );

        require("dotenv").config();
        secretsLoaded = true;
    } catch (error) {
        console.error("Failed to load secrets:", error);
    }
}

functions.http("helloWorld", async (req, res) => {
    await loadSecrets();

    res.json({
        message: "Hello World!",
        hasDatabase: !!process.env.DATABASE_URL,
    });
});
```

### Deployment

```bash
# Deploy function
gcloud functions deploy helloWorld \
  --runtime nodejs18 \
  --trigger-http \
  --allow-unauthenticated \
  --set-env-vars DOTENV_PROJECT=my-app \
  --set-env-vars DOTENV_ENVIRONMENT=production \
  --set-secrets DOTENV_API_KEY=projects/$PROJECT_ID/secrets/dotenv-api-key/versions/latest
```

## Google Kubernetes Engine (GKE)

### Workload Identity

Configure service account:

```bash
# Create service account
gcloud iam service-accounts create dotenv-sync \
  --display-name "DotEnv Sync Service Account"

# Grant Secret Manager access
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:dotenv-sync@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Enable Workload Identity
kubectl annotate serviceaccount dotenv-sync \
  iam.gke.io/gcp-service-account=dotenv-sync@$PROJECT_ID.iam.gserviceaccount.com
```

### CronJob for Sync

```yaml
# cronjob.yaml
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
                    serviceAccountName: dotenv-sync
                    containers:
                        - name: sync
                          image: dotenv/cli:latest
                          env:
                              - name: DOTENV_PROJECT
                                value: my-app
                              - name: DOTENV_ENVIRONMENT
                                value: production
                              - name: DOTENV_API_KEY
                                valueFrom:
                                    secretKeyRef:
                                        name: dotenv-credentials
                                        key: api-key
                          command:
                              - /bin/sh
                              - -c
                              - |
                                  dotenv auth login --api-key $DOTENV_API_KEY
                                  dotenv pull $DOTENV_ENVIRONMENT --project $DOTENV_PROJECT --format json > /tmp/secrets.json

                                  # Create Kubernetes secret
                                  kubectl create secret generic app-secrets \
                                    --from-file=secrets.json=/tmp/secrets.json \
                                    --dry-run=client -o yaml | kubectl apply -f -
                    restartPolicy: OnFailure
```

## Cloud Build

### Build Configuration

`cloudbuild.yaml`:

```yaml
steps:
    # Install DotEnv CLI
    - name: "gcr.io/cloud-builders/curl"
      entrypoint: "bash"
      args:
          - "-c"
          - |
              curl -fsSL https://cli.dotenv.cloud/install.sh | sh
              echo "Installed DotEnv CLI"

    # Load secrets
    - name: "gcr.io/cloud-builders/gcloud"
      entrypoint: "bash"
      env:
          - "DOTENV_PROJECT=${_DOTENV_PROJECT}"
          - "DOTENV_ENVIRONMENT=${_DOTENV_ENVIRONMENT}"
      args:
          - "-c"
          - |
              export PATH="$$HOME/.dotenv/bin:$$PATH"

              # Get API key from Secret Manager
              export DOTENV_API_KEY=$(gcloud secrets versions access latest --secret=dotenv-api-key)

              # Load secrets
              dotenv auth login --api-key $$DOTENV_API_KEY
              dotenv pull $$DOTENV_ENVIRONMENT --project $$DOTENV_PROJECT

              # Save for next steps
              cp .env /workspace/.env

    # Build application
    - name: "gcr.io/cloud-builders/npm"
      entrypoint: "bash"
      args:
          - "-c"
          - |
              source /workspace/.env
              npm install
              npm run build

    # Build Docker image
    - name: "gcr.io/cloud-builders/docker"
      args: ["build", "-t", "gcr.io/$PROJECT_ID/$_SERVICE_NAME", "."]

    # Push to Container Registry
    - name: "gcr.io/cloud-builders/docker"
      args: ["push", "gcr.io/$PROJECT_ID/$_SERVICE_NAME"]

    # Deploy to Cloud Run
    - name: "gcr.io/cloud-builders/gcloud"
      args:
          - "run"
          - "deploy"
          - "${_SERVICE_NAME}"
          - "--image=gcr.io/$PROJECT_ID/$_SERVICE_NAME"
          - "--region=${_REGION}"
          - "--platform=managed"

substitutions:
    _SERVICE_NAME: my-app
    _REGION: us-central1
    _DOTENV_PROJECT: my-app
    _DOTENV_ENVIRONMENT: production

options:
    logging: CLOUD_LOGGING_ONLY
```

## Security Best Practices

### 1. Service Account Permissions

Least privilege IAM:

```bash
# Create custom role
gcloud iam roles create dotenvSecretReader \
  --project=$PROJECT_ID \
  --title="DotEnv Secret Reader" \
  --description="Read access to DotEnv managed secrets" \
  --permissions=secretmanager.versions.access \
  --condition="resource.name.matches('projects/.*/secrets/dotenv-.*')"

# Grant to service account
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:my-app@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="projects/$PROJECT_ID/roles/dotenvSecretReader"
```

### 2. VPC Service Controls

Protect secret access:

```yaml
# vpc-service-perimeter.yaml
name: "accessPolicies/POLICY_ID/servicePerimeters/dotenv-perimeter"
title: "DotEnv Secret Perimeter"
perimeter_type: PERIMETER_TYPE_REGULAR
status:
    resources:
        - "projects/PROJECT_NUMBER"
    restricted_services:
        - "secretmanager.googleapis.com"
    vpc_accessible_services:
        enable_restriction: true
        allowed_services:
            - "secretmanager.googleapis.com"
```

### 3. Binary Authorization

Verify container images:

```yaml
# binary-authorization-policy.yaml
name: projects/PROJECT_ID/policy
admissionWhitelistPatterns:
    - namePattern: gcr.io/PROJECT_ID/*
defaultAdmissionRule:
    evaluationMode: REQUIRE_ATTESTATION
    enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
    requireAttestationsBy:
        - projects/PROJECT_ID/attestors/prod-attestor
```

### 4. Audit Logging

Enable Cloud Audit Logs:

```json
{
    "auditConfigs": [
        {
            "service": "secretmanager.googleapis.com",
            "auditLogConfigs": [
                {
                    "logType": "ADMIN_READ"
                },
                {
                    "logType": "DATA_READ",
                    "exemptedMembers": []
                }
            ]
        }
    ]
}
```

## Multi-Project Setup

### Shared Secrets

Share across projects:

```python
# scripts/share_secrets.py
from google.cloud import secretmanager
import google.auth

def share_secrets_across_projects(source_project, target_projects):
    """Share DotEnv secrets across GCP projects."""

    client = secretmanager.SecretManagerServiceClient()

    # List secrets in source project
    parent = f"projects/{source_project}"
    secrets = client.list_secrets(
        request={
            "parent": parent,
            "filter": "labels.managed-by=dotenv"
        }
    )

    for secret in secrets:
        # Get IAM policy
        policy = client.get_iam_policy(
            request={"resource": secret.name}
        )

        # Add target project service accounts
        for target_project in target_projects:
            member = f"serviceAccount:{target_project}@appspot.gserviceaccount.com"

            policy.bindings.append({
                "role": "roles/secretmanager.secretAccessor",
                "members": [member]
            })

        # Update policy
        client.set_iam_policy(
            request={
                "resource": secret.name,
                "policy": policy
            }
        )

        print(f"Shared {secret.name} with {target_projects}")

if __name__ == '__main__':
    share_secrets_across_projects(
        'my-shared-secrets-project',
        ['my-app-dev', 'my-app-staging', 'my-app-prod']
    )
```

## Monitoring

### Cloud Monitoring

Track secret access:

```javascript
// utils/monitoring.js
const monitoring = require("@google-cloud/monitoring");

const client = new monitoring.MetricServiceClient();
const projectId = process.env.GCP_PROJECT;

async function trackSecretAccess(secretName, success) {
    const dataPoint = {
        interval: {
            endTime: {
                seconds: Date.now() / 1000,
            },
        },
        value: {
            int64Value: 1,
        },
    };

    const timeSeries = {
        metric: {
            type: "custom.googleapis.com/dotenv/secret_access",
            labels: {
                secret_name: secretName,
                result: success ? "success" : "failure",
            },
        },
        resource: {
            type: "global",
            labels: {
                project_id: projectId,
            },
        },
        points: [dataPoint],
    };

    const request = {
        name: client.projectPath(projectId),
        timeSeries: [timeSeries],
    };

    await client.createTimeSeries(request);
}

// Create alert policy
async function createSecretAccessAlert() {
    const alertClient = new monitoring.AlertPolicyServiceClient();

    const alertPolicy = {
        displayName: "DotEnv Secret Access Failures",
        conditions: [
            {
                displayName: "Secret access failure rate",
                conditionThreshold: {
                    filter: 'metric.type="custom.googleapis.com/dotenv/secret_access" AND metric.label.result="failure"',
                    comparison: "COMPARISON_GT",
                    thresholdValue: 5,
                    duration: "60s",
                    aggregations: [
                        {
                            alignmentPeriod: "60s",
                            perSeriesAligner: "ALIGN_RATE",
                        },
                    ],
                },
            },
        ],
        notificationChannels: [
            `projects/${projectId}/notificationChannels/${process.env.NOTIFICATION_CHANNEL}`,
        ],
    };

    await alertClient.createAlertPolicy({
        name: client.projectPath(projectId),
        alertPolicy,
    });
}

module.exports = { trackSecretAccess, createSecretAccessAlert };
```

## Troubleshooting

### Common Issues

#### Permission Denied

```bash
# Check service account permissions
gcloud projects get-iam-policy $PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:serviceAccount:*" \
  --format="table(bindings.role)"

# Test secret access
gcloud secrets versions access latest \
  --secret=dotenv-api-key \
  --impersonate-service-account=my-app@$PROJECT_ID.iam.gserviceaccount.com
```

#### Cloud Build Failures

```yaml
# Add debugging to cloudbuild.yaml
options:
    logging: CLOUD_LOGGING_ONLY
    env:
        - "DEBUG=true"
        - "DOTENV_LOG_LEVEL=debug"
```

## Examples

### Complete Application

```javascript
// app.js
const express = require("express");
const { Storage } = require("@google-cloud/storage");
const { loadSecretsFromGCP } = require("./utils/secrets");

async function startApp() {
    // Load secrets
    if (process.env.K_SERVICE) {
        // Running on Cloud Run
        await loadSecretsFromGCP();
    } else {
        // Local development
        require("dotenv").config();
    }

    const app = express();
    const storage = new Storage();

    app.get("/", (req, res) => {
        res.json({
            service: process.env.K_SERVICE || "local",
            revision: process.env.K_REVISION || "dev",
            hasDatabase: !!process.env.DATABASE_URL,
        });
    });

    app.get("/health", (req, res) => {
        res.status(200).send("OK");
    });

    const port = process.env.PORT || 8080;
    app.listen(port, () => {
        console.log(`Server running on port ${port}`);
    });
}

startApp().catch(console.error);
```

### Terraform Configuration

```hcl
# main.tf
resource "google_secret_manager_secret" "dotenv_api_key" {
  secret_id = "dotenv-api-key"

  labels = {
    managed-by = "terraform"
  }

  replication {
    automatic = true
  }
}

resource "google_secret_manager_secret_version" "dotenv_api_key" {
  secret = google_secret_manager_secret.dotenv_api_key.id
  secret_data = var.dotenv_api_key
}

resource "google_cloud_run_service" "app" {
  name     = "my-app"
  location = var.region

  template {
    spec {
      service_account_name = google_service_account.app.email

      containers {
        image = "gcr.io/${var.project_id}/my-app"

        env {
          name  = "DOTENV_PROJECT"
          value = "my-app"
        }

        env {
          name  = "DOTENV_ENVIRONMENT"
          value = var.environment
        }

        env {
          name = "DOTENV_API_KEY"
          value_from {
            secret_key_ref {
              name = google_secret_manager_secret.dotenv_api_key.secret_id
              key  = "latest"
            }
          }
        }
      }
    }
  }
}
```

## Resources

- [Google Cloud Documentation](https://cloud.google.com/docs)
- [Secret Manager](https://cloud.google.com/secret-manager/docs)
- [Cloud Run](https://cloud.google.com/run/docs)
- [DotEnv CLI Reference](/documentation/v1/cli/commands)

## Next Steps

- [Configure Secret Manager sync](#secret-manager-integration)
- [Deploy to Cloud Run](#cloud-run)
- [Set up Cloud Functions](#cloud-functions)
- [Enable monitoring](#monitoring)
