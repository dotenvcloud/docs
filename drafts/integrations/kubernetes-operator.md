---
title: Kubernetes Operator
slug: kubernetes-operator
order: 40
tags: [integrations, kubernetes, k8s, operator, cloud-native]
---

# Kubernetes Operator

Advanced Kubernetes integration using a custom operator for automated secret management and synchronization.

## Overview

The DotEnv Kubernetes Operator provides:

- Custom Resource Definitions (CRDs) for secret management
- Automatic secret synchronization
- Multi-tenancy support
- Secret rotation automation
- RBAC integration
- Admission webhooks for security

## Installation

### Using Helm

```bash
helm repo add dotenv https://charts.dotenv.cloud
helm repo update
helm install dotenv-operator dotenv/operator \
  --namespace dotenv-system \
  --create-namespace \
  --set apiKey=your-api-key
```

### Using kubectl

```bash
kubectl apply -f https://dotenv.cloud/operator/install.yaml
```

## Custom Resources

### DotEnvProject

```yaml
apiVersion: secrets.dotenv.cloud/v1
kind: DotEnvProject
metadata:
  name: my-app-project
  namespace: default
spec:
  projectId: "proj_123456"
  syncInterval: 300 # seconds
  targetSecret:
    name: my-app-secrets
    type: Opaque
```

### DotEnvSecret

```yaml
apiVersion: secrets.dotenv.cloud/v1
kind: DotEnvSecret
metadata:
  name: database-credentials
  namespace: default
spec:
  projectRef:
    name: my-app-project
  secrets:
    - key: DATABASE_URL
      remoteRef:
        key: PROD_DATABASE_URL
    - key: API_KEY
      remoteRef:
        key: PROD_API_KEY
  refreshInterval: 60
```

### DotEnvSecretRotation

```yaml
apiVersion: secrets.dotenv.cloud/v1
kind: DotEnvSecretRotation
metadata:
  name: api-key-rotation
spec:
  secretRef:
    name: database-credentials
    key: API_KEY
  schedule: "0 0 * * 0" # Weekly
  notificationChannels:
    - slack
    - email
```

## Advanced Features

### Multi-Environment Sync

```yaml
apiVersion: secrets.dotenv.cloud/v1
kind: DotEnvEnvironmentSync
metadata:
  name: multi-env-sync
spec:
  projectId: "proj_123456"
  environments:
    - name: development
      targetNamespace: dev
      targetSecret: app-secrets
    - name: staging
      targetNamespace: staging
      targetSecret: app-secrets
    - name: production
      targetNamespace: prod
      targetSecret: app-secrets
      requireApproval: true
```

### Secret Policy Enforcement

```yaml
apiVersion: secrets.dotenv.cloud/v1
kind: DotEnvSecretPolicy
metadata:
  name: security-policy
spec:
  selector:
    matchLabels:
      tier: production
  rules:
    - requiredTags:
        - compliance
        - owner
    - encryption:
        required: true
        keyRotationDays: 90
    - access:
        requireMFA: true
        allowedNamespaces:
          - prod
          - prod-backup
```

### Webhook Configuration

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: dotenv-secret-validator
webhooks:
  - name: validate.secrets.dotenv.cloud
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: ["secrets.dotenv.cloud"]
        apiVersions: ["v1"]
        resources: ["dotenvsecrets"]
    clientConfig:
      service:
        name: dotenv-webhook
        namespace: dotenv-system
        path: "/validate"
```

## Monitoring and Observability

### Prometheus Metrics

```yaml
apiVersion: v1
kind: ServiceMonitor
metadata:
  name: dotenv-operator-metrics
spec:
  selector:
    matchLabels:
      app: dotenv-operator
  endpoints:
    - port: metrics
      interval: 30s
```

Available metrics:
- `dotenv_sync_duration_seconds`
- `dotenv_sync_errors_total`
- `dotenv_secret_rotation_total`
- `dotenv_api_requests_total`

## Security Considerations

1. **RBAC Configuration**
   - Minimal permissions principle
   - Namespace isolation
   - Audit logging

2. **Network Policies**
   - Restrict operator egress
   - API endpoint allowlisting

3. **Secret Encryption**
   - Encryption at rest
   - Encryption in transit
   - Key management integration

## Implementation Status

This advanced Kubernetes operator is currently in planning. The basic Kubernetes integration using init containers and ConfigMaps is available now.