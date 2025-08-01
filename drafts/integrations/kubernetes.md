---
title: Kubernetes Integration
slug: kubernetes
order: 6
tags: [integrations, kubernetes, k8s, containers, orchestration]
---

# Kubernetes Integration

Integrate DotEnv with Kubernetes to manage secrets across your containerized applications. This guide covers Kubernetes secrets, ConfigMaps, operators, and best practices.

## Overview

The DotEnv Kubernetes integration provides:

- Automatic secret synchronization
- Native Kubernetes secret management
- Multi-namespace support
- Secret rotation capabilities
- Compliance and audit logging

## Installation Methods

### 1. DotEnv Operator (Recommended)

Install the DotEnv Kubernetes Operator:

```bash
# Using Helm
helm repo add dotenv https://charts.dotenv.cloud
helm repo update
helm install dotenv-operator dotenv/dotenv-operator \
  --namespace dotenv-system \
  --create-namespace \
  --set apiKey=$DOTENV_API_KEY

# Using kubectl
kubectl apply -f https://raw.githubusercontent.com/dotenv/k8s-operator/main/deploy/operator.yaml
```

### 2. CLI-Based Sync

Use DotEnv CLI in init containers or CronJobs:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
    name: dotenv-sync
spec:
    schedule: "*/5 * * * *"
    jobTemplate:
        spec:
            template:
                spec:
                    containers:
                        - name: sync
                          image: dotenv/cli:latest
                          env:
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
                                  dotenv pull production --project my-app --format json > /tmp/secrets.json
                                  kubectl create secret generic app-secrets \
                                    --from-file=secrets.json=/tmp/secrets.json \
                                    --dry-run=client -o yaml | kubectl apply -f -
                    restartPolicy: OnFailure
```

### 3. External Secrets Operator

Integrate with External Secrets Operator:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
    name: dotenv-backend
spec:
    provider:
        webhook:
            url: "https://api.dotenv.cloud/v1/secrets"
            headers:
                Authorization:
                    value: "Bearer <your-api-key>"
            result:
                jsonPath: "$.data"
```

## Basic Configuration

### Using DotEnv Operator

Create a DotEnvSecret resource:

```yaml
apiVersion: dotenv.cloud/v1
kind: DotEnvSecret
metadata:
    name: app-secrets
    namespace: default
spec:
    project: my-app
    environment: production
    target:
        name: app-secrets
        type: Secret
    refreshInterval: 5m
    secretType: Opaque
```

The operator will create and maintain a Kubernetes Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
    name: app-secrets
    namespace: default
type: Opaque
data:
    DATABASE_URL: <base64-encoded-value>
    API_KEY: <base64-encoded-value>
    JWT_SECRET: <base64-encoded-value>
```

### Using in Deployments

Reference the synchronized secrets:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: my-app
spec:
    replicas: 3
    selector:
        matchLabels:
            app: my-app
    template:
        metadata:
            labels:
                app: my-app
        spec:
            containers:
                - name: app
                  image: myapp:latest
                  envFrom:
                      - secretRef:
                            name: app-secrets
                  env:
                      - name: NODE_ENV
                        value: production
                  ports:
                      - containerPort: 3000
```

## Advanced Patterns

### 1. Multi-Environment Setup

Manage multiple environments in different namespaces:

```yaml
# Development namespace
---
apiVersion: v1
kind: Namespace
metadata:
    name: development
---
apiVersion: dotenv.cloud/v1
kind: DotEnvSecret
metadata:
    name: app-secrets
    namespace: development
spec:
    project: my-app
    environment: development
    target:
        name: app-secrets

# Staging namespace
---
apiVersion: v1
kind: Namespace
metadata:
    name: staging
---
apiVersion: dotenv.cloud/v1
kind: DotEnvSecret
metadata:
    name: app-secrets
    namespace: staging
spec:
    project: my-app
    environment: staging
    target:
        name: app-secrets

# Production namespace
---
apiVersion: v1
kind: Namespace
metadata:
    name: production
---
apiVersion: dotenv.cloud/v1
kind: DotEnvSecret
metadata:
    name: app-secrets
    namespace: production
spec:
    project: my-app
    environment: production
    target:
        name: app-secrets
    # Production-specific settings
    refreshInterval: 1m
    secretRotation:
        enabled: true
        schedule: "0 0 * * 0" # Weekly
```

### 2. Secret Mapping

Map specific secrets to different keys:

```yaml
apiVersion: dotenv.cloud/v1
kind: DotEnvSecret
metadata:
    name: database-config
spec:
    project: my-app
    environment: production
    target:
        name: database-config
    secretMappings:
        - source: DATABASE_URL
          target: POSTGRES_CONNECTION_STRING
        - source: DB_HOST
          target: PGHOST
        - source: DB_PORT
          target: PGPORT
        - source: DB_NAME
          target: PGDATABASE
        - source: DB_USER
          target: PGUSER
        - source: DB_PASSWORD
          target: PGPASSWORD
```

### 3. Init Container Pattern

Fetch secrets at pod startup:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: my-app
spec:
    template:
        spec:
            initContainers:
                - name: secret-fetcher
                  image: dotenv/cli:latest
                  env:
                      - name: DOTENV_API_KEY
                        valueFrom:
                            secretKeyRef:
                                name: dotenv-credentials
                                key: api-key
                  command:
                      - sh
                      - -c
                      - |
                          dotenv auth login --api-key $DOTENV_API_KEY
                          dotenv pull production --project my-app --format env > /shared/secrets.env
                  volumeMounts:
                      - name: shared-data
                        mountPath: /shared
            containers:
                - name: app
                  image: myapp:latest
                  command:
                      - sh
                      - -c
                      - |
                          source /shared/secrets.env
                          exec node index.js
                  volumeMounts:
                      - name: shared-data
                        mountPath: /shared
            volumes:
                - name: shared-data
                  emptyDir: {}
```

### 4. Sidecar Pattern

Run DotEnv as a sidecar for continuous updates:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: my-app
spec:
    template:
        spec:
            containers:
                # Main application
                - name: app
                  image: myapp:latest
                  volumeMounts:
                      - name: secrets
                        mountPath: /etc/secrets
                        readOnly: true
                  env:
                      - name: SECRETS_PATH
                        value: /etc/secrets/.env

                # DotEnv sidecar
                - name: dotenv-sidecar
                  image: dotenv/cli:latest
                  env:
                      - name: DOTENV_API_KEY
                        valueFrom:
                            secretKeyRef:
                                name: dotenv-credentials
                                key: api-key
                      - name: PROJECT_NAME
                        value: my-app
                      - name: ENVIRONMENT
                        value: production
                  command:
                      - sh
                      - -c
                      - |
                          dotenv auth login --api-key $DOTENV_API_KEY
                          while true; do
                            dotenv pull $ENVIRONMENT --project $PROJECT_NAME --format env > /secrets/.env.tmp
                            mv /secrets/.env.tmp /secrets/.env
                            sleep 60
                          done
                  volumeMounts:
                      - name: secrets
                        mountPath: /secrets

            volumes:
                - name: secrets
                  emptyDir: {}
```

## Security Best Practices

### 1. RBAC Configuration

Create appropriate roles and bindings:

```yaml
# ServiceAccount for DotEnv operator
apiVersion: v1
kind: ServiceAccount
metadata:
    name: dotenv-operator
    namespace: dotenv-system

---
# Role for secret management
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
    name: dotenv-operator
rules:
    - apiGroups: [""]
      resources: ["secrets", "configmaps"]
      verbs: ["get", "list", "watch", "create", "update", "patch"]
    - apiGroups: ["dotenv.cloud"]
      resources: ["dotenvsecrets"]
      verbs: ["get", "list", "watch"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
    name: dotenv-operator
roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: dotenv-operator
subjects:
    - kind: ServiceAccount
      name: dotenv-operator
      namespace: dotenv-system
```

### 2. Network Policies

Restrict secret access:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
    name: dotenv-operator-policy
    namespace: dotenv-system
spec:
    podSelector:
        matchLabels:
            app: dotenv-operator
    policyTypes:
        - Egress
    egress:
        - to:
              - namespaceSelector: {}
          ports:
              - protocol: TCP
                port: 443 # HTTPS to DotEnv API
        - to:
              - namespaceSelector:
                    matchLabels:
                        name: kube-system
          ports:
              - protocol: TCP
                port: 443 # Kubernetes API
```

### 3. Pod Security Standards

Enforce security policies:

```yaml
apiVersion: v1
kind: Namespace
metadata:
    name: production
    labels:
        pod-security.kubernetes.io/enforce: restricted
        pod-security.kubernetes.io/audit: restricted
        pod-security.kubernetes.io/warn: restricted
```

### 4. Secret Encryption

Enable encryption at rest:

```yaml
# encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
    - resources:
          - secrets
      providers:
          - aescbc:
                keys:
                    - name: key1
                      secret: <base64-encoded-key>
          - identity: {}
```

## Monitoring and Observability

### 1. Prometheus Metrics

Export metrics from DotEnv operator:

```yaml
apiVersion: v1
kind: Service
metadata:
    name: dotenv-operator-metrics
    namespace: dotenv-system
    labels:
        app: dotenv-operator
spec:
    ports:
        - name: metrics
          port: 8080
          targetPort: 8080
    selector:
        app: dotenv-operator

---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
    name: dotenv-operator
    namespace: dotenv-system
spec:
    selector:
        matchLabels:
            app: dotenv-operator
    endpoints:
        - port: metrics
          interval: 30s
```

### 2. Event Logging

Configure audit logging:

```yaml
apiVersion: dotenv.cloud/v1
kind: DotEnvSecret
metadata:
    name: app-secrets
spec:
    project: my-app
    environment: production
    auditLog:
        enabled: true
        includeData: false
        webhookUrl: https://audit.example.com/webhook
    events:
        - secret.created
        - secret.updated
        - secret.accessed
        - secret.rotated
```

### 3. Alerts

Configure alerting rules:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
    name: dotenv-alerts
    namespace: dotenv-system
spec:
    groups:
        - name: dotenv
          interval: 30s
          rules:
              - alert: DotEnvSyncFailure
                expr: dotenv_sync_failures_total > 0
                for: 5m
                annotations:
                    summary: "DotEnv sync failures detected"
                    description: "{{ $labels.namespace }}/{{ $labels.name }} has failed to sync"

              - alert: DotEnvAPIRateLimit
                expr: dotenv_api_rate_limit_remaining < 100
                for: 1m
                annotations:
                    summary: "DotEnv API rate limit low"
                    description: "Only {{ $value }} requests remaining"
```

## Deployment Strategies

### 1. Blue-Green Deployment

```yaml
# Blue deployment (current)
apiVersion: apps/v1
kind: Deployment
metadata:
    name: my-app-blue
    labels:
        version: blue
spec:
    replicas: 3
    selector:
        matchLabels:
            app: my-app
            version: blue
    template:
        metadata:
            labels:
                app: my-app
                version: blue
        spec:
            containers:
                - name: app
                  image: myapp:v1.0
                  envFrom:
                      - secretRef:
                            name: app-secrets-blue

---
# Green deployment (new)
apiVersion: apps/v1
kind: Deployment
metadata:
    name: my-app-green
    labels:
        version: green
spec:
    replicas: 3
    selector:
        matchLabels:
            app: my-app
            version: green
    template:
        metadata:
            labels:
                app: my-app
                version: green
        spec:
            containers:
                - name: app
                  image: myapp:v2.0
                  envFrom:
                      - secretRef:
                            name: app-secrets-green

---
# Service with selector switch
apiVersion: v1
kind: Service
metadata:
    name: my-app
spec:
    selector:
        app: my-app
        version: blue # Switch to green when ready
    ports:
        - port: 80
          targetPort: 3000
```

### 2. Canary Deployment

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
    name: my-app
spec:
    targetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: my-app
    service:
        port: 80
    analysis:
        interval: 1m
        threshold: 10
        maxWeight: 50
        stepWeight: 5
    # Update secrets during canary
    preRolloutWebhooks:
        - name: update-secrets
          url: http://dotenv-operator.dotenv-system/webhook/update
          metadata:
              project: my-app
              environment: production
```

### 3. Rolling Update

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: my-app
spec:
    replicas: 10
    strategy:
        type: RollingUpdate
        rollingUpdate:
            maxSurge: 2
            maxUnavailable: 1
    template:
        spec:
            containers:
                - name: app
                  image: myapp:latest
                  envFrom:
                      - secretRef:
                            name: app-secrets
                  livenessProbe:
                      httpGet:
                          path: /health
                          port: 3000
                      initialDelaySeconds: 30
                      periodSeconds: 10
                  readinessProbe:
                      httpGet:
                          path: /ready
                          port: 3000
                      initialDelaySeconds: 5
                      periodSeconds: 5
```

## Troubleshooting

### Debug DotEnv Operator

```bash
# Check operator logs
kubectl logs -n dotenv-system deployment/dotenv-operator

# Check operator status
kubectl get dotenvsecrets -A

# Describe specific secret sync
kubectl describe dotenvsecret app-secrets

# Check events
kubectl get events -n default --field-selector involvedObject.name=app-secrets
```

### Common Issues

#### Secret Not Syncing

```bash
# Check API connectivity
kubectl run debug --rm -it --image=curlimages/curl -- \
  curl -H "Authorization: Bearer $DOTENV_API_KEY" \
  https://api.dotenv.cloud/v1/projects

# Check RBAC permissions
kubectl auth can-i create secrets --as=system:serviceaccount:dotenv-system:dotenv-operator

# Verify secret exists
kubectl get secret app-secrets -o yaml
```

#### Permission Denied

```yaml
# Fix: Grant proper permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
    name: secret-reader
    namespace: default
rules:
    - apiGroups: [""]
      resources: ["secrets"]
      resourceNames: ["app-secrets"]
      verbs: ["get", "list"]
```

## Examples

### Complete Application Deployment

```yaml
# Namespace
apiVersion: v1
kind: Namespace
metadata:
    name: myapp-prod

---
# DotEnv API credentials
apiVersion: v1
kind: Secret
metadata:
    name: dotenv-credentials
    namespace: myapp-prod
type: Opaque
data:
    api-key: <base64-encoded-api-key>

---
# DotEnv secret sync
apiVersion: dotenv.cloud/v1
kind: DotEnvSecret
metadata:
    name: app-config
    namespace: myapp-prod
spec:
    project: my-app
    environment: production
    refreshInterval: 1m
    target:
        name: app-secrets
    secretRotation:
        enabled: true
        schedule: "0 2 * * SUN"

---
# ConfigMap for non-sensitive config
apiVersion: v1
kind: ConfigMap
metadata:
    name: app-config
    namespace: myapp-prod
data:
    NODE_ENV: production
    LOG_LEVEL: info
    PORT: "3000"

---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
    name: my-app
    namespace: myapp-prod
spec:
    replicas: 3
    selector:
        matchLabels:
            app: my-app
    template:
        metadata:
            labels:
                app: my-app
            annotations:
                dotenv.cloud/inject: "true"
        spec:
            serviceAccountName: my-app
            securityContext:
                runAsNonRoot: true
                runAsUser: 1000
                fsGroup: 1000
            containers:
                - name: app
                  image: myapp:latest
                  imagePullPolicy: Always
                  ports:
                      - containerPort: 3000
                        name: http
                  envFrom:
                      - configMapRef:
                            name: app-config
                      - secretRef:
                            name: app-secrets
                  resources:
                      requests:
                          memory: "256Mi"
                          cpu: "250m"
                      limits:
                          memory: "512Mi"
                          cpu: "500m"
                  livenessProbe:
                      httpGet:
                          path: /health
                          port: http
                      initialDelaySeconds: 30
                      periodSeconds: 10
                  readinessProbe:
                      httpGet:
                          path: /ready
                          port: http
                      initialDelaySeconds: 5
                      periodSeconds: 5
                  volumeMounts:
                      - name: cache
                        mountPath: /app/cache
            volumes:
                - name: cache
                  emptyDir: {}

---
# Service
apiVersion: v1
kind: Service
metadata:
    name: my-app
    namespace: myapp-prod
spec:
    selector:
        app: my-app
    ports:
        - port: 80
          targetPort: http
    type: ClusterIP

---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
    name: my-app
    namespace: myapp-prod
    annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
        nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
    tls:
        - hosts:
              - app.example.com
          secretName: app-tls
    rules:
        - host: app.example.com
          http:
              paths:
                  - path: /
                    pathType: Prefix
                    backend:
                        service:
                            name: my-app
                            port:
                                number: 80

---
# HorizontalPodAutoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
    name: my-app
    namespace: myapp-prod
spec:
    scaleTargetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: my-app
    minReplicas: 3
    maxReplicas: 10
    metrics:
        - type: Resource
          resource:
              name: cpu
              target:
                  type: Utilization
                  averageUtilization: 70
        - type: Resource
          resource:
              name: memory
              target:
                  type: Utilization
                  averageUtilization: 80

---
# PodDisruptionBudget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
    name: my-app
    namespace: myapp-prod
spec:
    minAvailable: 2
    selector:
        matchLabels:
            app: my-app
```

## Helm Chart Integration

Create a Helm chart with DotEnv integration:

```yaml
# values.yaml
dotenv:
  enabled: true
  project: my-app
  environment: production
  refreshInterval: 5m

app:
  name: my-app
  image:
    repository: myapp
    tag: latest
    pullPolicy: Always

  replicas: 3

  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"

# templates/dotenvsecret.yaml
{{- if .Values.dotenv.enabled }}
apiVersion: dotenv.cloud/v1
kind: DotEnvSecret
metadata:
  name: {{ .Values.app.name }}-secrets
spec:
  project: {{ .Values.dotenv.project }}
  environment: {{ .Values.dotenv.environment }}
  refreshInterval: {{ .Values.dotenv.refreshInterval }}
  target:
    name: {{ .Values.app.name }}-secrets
{{- end }}
```

## Migration from Kubernetes Secrets

Migrate existing secrets to DotEnv:

```bash
#!/bin/bash
# migrate-secrets.sh

NAMESPACE="default"
SECRET_NAME="my-app-secrets"
PROJECT_NAME="my-app"
ENVIRONMENT="production"

# Export existing secrets
kubectl get secret $SECRET_NAME -n $NAMESPACE -o json | \
  jq -r '.data | to_entries[] | "\(.key)=\(.value | @base64d)"' > secrets.env

# Import to DotEnv
dotenv import --project $PROJECT_NAME --environment $ENVIRONMENT --file secrets.env

# Create DotEnvSecret resource
cat <<EOF | kubectl apply -f -
apiVersion: dotenv.cloud/v1
kind: DotEnvSecret
metadata:
  name: $SECRET_NAME
  namespace: $NAMESPACE
spec:
  project: $PROJECT_NAME
  environment: $ENVIRONMENT
  target:
    name: $SECRET_NAME
EOF

# Clean up
rm secrets.env
```

## Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [DotEnv Operator Documentation](https://github.com/dotenv/k8s-operator)
- [Helm Charts](https://charts.dotenv.cloud)
- [Security Best Practices](/documentation/v1/security/best-practices)

## Next Steps

- [Install DotEnv Operator](#dotenv-operator-recommended)
- [Configure secret synchronization](#basic-configuration)
- [Set up monitoring](#monitoring-and-observability)
- [Implement security policies](#security-best-practices)
