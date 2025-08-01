---
title: Docker Integration
slug: docker-usage
order: 10
tags: [cli, docker, containers, kubernetes]
---

# Docker Integration

This guide covers how to use the DotEnv CLI with Docker containers, Docker Compose, and container orchestration platforms.

## Docker Basics

### Installing CLI in Docker

```dockerfile
# Alpine-based image (lightweight)
FROM alpine:latest
RUN apk add --no-cache curl bash
RUN curl -sSL https://dotenv.cloud/install.sh | sh
ENV PATH="/root/.dotenv/bin:$PATH"

# Ubuntu-based image
FROM ubuntu:latest
RUN apt-get update && apt-get install -y curl
RUN curl -sSL https://dotenv.cloud/install.sh | sh
ENV PATH="/root/.dotenv/bin:$PATH"

# Debian-based image
FROM debian:slim
RUN apt-get update && apt-get install -y curl ca-certificates
RUN curl -sSL https://dotenv.cloud/install.sh | sh
ENV PATH="/root/.dotenv/bin:$PATH"
```

### Basic Usage in Dockerfile

```dockerfile
FROM node:18

# Install DotEnv CLI
RUN curl -sSL https://dotenv.cloud/install.sh | sh
ENV PATH="/root/.dotenv/bin:$PATH"

# Set build arguments
ARG DOTENV_API_KEY
ARG DOTENV_PROJECT=my-app
ARG DOTENV_ENVIRONMENT=production

# Pull secrets during build
RUN dotenv pull \
    --project "$DOTENV_PROJECT" \
    --environment "$DOTENV_ENVIRONMENT" \
    --output .env

# Build application with secrets
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN source .env && npm run build

# Run with secrets
CMD ["sh", "-c", "source .env && node server.js"]
```

## Build Patterns

### Multi-stage Builds

Separate secret handling from application build:

```dockerfile
# Stage 1: Secret fetcher
FROM alpine:latest AS secrets
ARG DOTENV_API_KEY
ARG PROJECT
ARG ENVIRONMENT

RUN apk add --no-cache curl bash
RUN curl -sSL https://dotenv.cloud/install.sh | sh
ENV PATH="/root/.dotenv/bin:$PATH"

# Pull and validate secrets
RUN dotenv pull \
    --project "${PROJECT}" \
    --environment "${ENVIRONMENT}" \
    --output /secrets/.env \
    && dotenv validate --file /secrets/.env

# Stage 2: Builder
FROM node:18 AS builder
WORKDIR /app

# Copy secrets from previous stage
COPY --from=secrets /secrets/.env .env

# Install dependencies
COPY package*.json ./
RUN npm ci

# Build with secrets
COPY . .
RUN set -a && source .env && set +a && npm run build

# Stage 3: Runtime (no secrets in final image)
FROM node:18-slim
WORKDIR /app

# Copy only built artifacts
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

# Runtime secrets loaded dynamically
COPY --from=secrets /root/.dotenv/bin/dotenv /usr/local/bin/
ENTRYPOINT ["dotenv", "run", "--"]
CMD ["node", "dist/server.js"]
```

### BuildKit Secrets

Use Docker BuildKit for secure secret handling:

```dockerfile
# syntax=docker/dockerfile:1
FROM node:18

# Install DotEnv CLI
RUN curl -sSL https://dotenv.cloud/install.sh | sh
ENV PATH="/root/.dotenv/bin:$PATH"

# Mount secret during build only
RUN --mount=type=secret,id=dotenv_key \
    DOTENV_API_KEY=$(cat /run/secrets/dotenv_key) \
    dotenv pull --project my-app --output .env

# Build with secrets
RUN --mount=type=secret,id=env,source=.env \
    set -a && source /run/secrets/env && set +a && \
    npm run build

# Secrets not persisted in image
CMD ["node", "server.js"]
```

Build command:

```bash
# Enable BuildKit
export DOCKER_BUILDKIT=1

# Build with secrets
docker build \
  --secret id=dotenv_key,src=.secrets/api-key \
  --secret id=env,src=.env \
  -t my-app .
```

### Build Arguments vs Secrets

```dockerfile
# Build arguments (visible in image history)
ARG NODE_ENV=production
ARG APP_VERSION=1.0.0

# Secrets (not visible in image history)
RUN --mount=type=secret,id=api_key \
    API_KEY=$(cat /run/secrets/api_key) && \
    dotenv login --token "$API_KEY"

# Runtime environment (loaded at container start)
ENTRYPOINT ["sh", "-c", "dotenv pull && exec $@", "--"]
CMD ["node", "server.js"]
```

## Runtime Patterns

### Dynamic Secret Loading

Load secrets at container startup:

```dockerfile
FROM node:18

# Install CLI
RUN curl -sSL https://dotenv.cloud/install.sh | sh
ENV PATH="/root/.dotenv/bin:$PATH"

# Copy application
WORKDIR /app
COPY . .
RUN npm ci --production

# Entrypoint script
COPY docker-entrypoint.sh /
RUN chmod +x /docker-entrypoint.sh

ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["node", "server.js"]
```

docker-entrypoint.sh:

```bash
#!/bin/bash
set -e

# Pull latest secrets
echo "Loading secrets from DotEnv..."
dotenv pull \
  --project "${DOTENV_PROJECT}" \
  --environment "${DOTENV_ENVIRONMENT:-production}" \
  --output .env

# Validate required secrets
REQUIRED_SECRETS="DATABASE_URL API_KEY JWT_SECRET"
for secret in $REQUIRED_SECRETS; do
  if ! grep -q "^${secret}=" .env; then
    echo "Error: Missing required secret: $secret"
    exit 1
  fi
done

# Source secrets and run command
set -a
source .env
set +a

# Execute main command
exec "$@"
```

### Sidecar Container Pattern

Use a sidecar to manage secrets:

```yaml
# docker-compose.yml
version: "3.8"

services:
    # Secret manager sidecar
    secrets-manager:
        image: dotenv/cli:latest
        environment:
            - DOTENV_API_KEY
            - DOTENV_PROJECT=my-app
            - DOTENV_ENVIRONMENT=production
        volumes:
            - secrets:/secrets
        command: |
            sh -c '
              while true; do
                dotenv pull --output /secrets/.env
                sleep 300  # Refresh every 5 minutes
              done
            '

    # Main application
    app:
        image: my-app:latest
        depends_on:
            - secrets-manager
        volumes:
            - secrets:/secrets:ro
        command: |
            sh -c '
              while [ ! -f /secrets/.env ]; do
                echo "Waiting for secrets..."
                sleep 2
              done
              source /secrets/.env
              exec node server.js
            '

volumes:
    secrets:
        driver: local
        driver_opts:
            type: tmpfs
            device: tmpfs
```

### Init Container Pattern

For better startup control:

```yaml
# docker-compose.yml
version: "3.8"

services:
    app:
        image: my-app:latest
        environment:
            - DOTENV_API_KEY
        entrypoint: |
            sh -c '
              # Init phase: Pull secrets
              dotenv pull --output /tmp/.env
              
              # Validate secrets
              if ! dotenv validate --file /tmp/.env; then
                echo "Secret validation failed"
                exit 1
              fi
              
              # Run application
              source /tmp/.env
              exec node server.js
            '
```

## Docker Compose Integration

### Development Setup

```yaml
# docker-compose.yml
version: "3.8"

services:
    app:
        build:
            context: .
            args:
                - DOTENV_API_KEY=${DOTENV_API_KEY}
                - DOTENV_ENVIRONMENT=development
        environment:
            - NODE_ENV=development
        volumes:
            - .:/app
            - /app/node_modules
        command: |
            sh -c '
              dotenv pull --environment development
              source .env
              npm run dev
            '

    database:
        image: postgres:15
        env_file:
            - .env
        environment:
            - POSTGRES_DB=${DB_NAME}
            - POSTGRES_USER=${DB_USER}
            - POSTGRES_PASSWORD=${DB_PASSWORD}

    redis:
        image: redis:7
        command: redis-server --requirepass ${REDIS_PASSWORD}
```

### Production Setup

```yaml
# docker-compose.prod.yml
version: "3.8"

x-common-env: &common-env
    DOTENV_API_KEY: ${DOTENV_API_KEY}
    DOTENV_PROJECT: my-app
    DOTENV_NON_INTERACTIVE: "true"

services:
    # Secret initialization service
    init-secrets:
        image: alpine:latest
        environment:
            <<: *common-env
            DOTENV_ENVIRONMENT: production
        volumes:
            - secrets:/secrets
        command: |
            sh -c '
              apk add --no-cache curl bash
              curl -sSL https://dotenv.cloud/install.sh | sh
              export PATH="/root/.dotenv/bin:$PATH"
              
              # Pull secrets for all services
              dotenv pull --output /secrets/app.env
              dotenv pull --project my-app-db --output /secrets/db.env
              dotenv pull --project my-app-cache --output /secrets/cache.env
              
              # Set permissions
              chmod 400 /secrets/*.env
            '

    app:
        image: my-app:${VERSION:-latest}
        depends_on:
            init-secrets:
                condition: service_completed_successfully
        volumes:
            - secrets:/secrets:ro
        deploy:
            replicas: 3
            restart_policy:
                condition: on-failure
        entrypoint: ["/bin/sh", "-c"]
        command:
            - |
                source /secrets/app.env
                exec node server.js

    nginx:
        image: nginx:alpine
        depends_on:
            - app
        volumes:
            - secrets:/secrets:ro
            - ./nginx.conf:/etc/nginx/nginx.conf:ro
        entrypoint: ["/bin/sh", "-c"]
        command:
            - |
                source /secrets/app.env
                envsubst < /etc/nginx/nginx.conf > /tmp/nginx.conf
                exec nginx -c /tmp/nginx.conf -g 'daemon off;'

volumes:
    secrets:
        driver: local
        driver_opts:
            type: tmpfs
            device: tmpfs
            o: size=10m,uid=1000
```

### Override Files

```yaml
# docker-compose.override.yml
version: "3.8"

services:
    app:
        environment:
            - DOTENV_ENVIRONMENT=development
            - DOTENV_DEBUG=true
        volumes:
            - .:/app
        command: npm run dev

    # Local services
    mailhog:
        image: mailhog/mailhog
        ports:
            - "8025:8025"
            - "1025:1025"
```

## Kubernetes Integration

### ConfigMap Generation

```bash
#!/bin/bash
# generate-configmap.sh

# Pull secrets and create ConfigMap
dotenv pull \
  --project my-app \
  --environment production \
  --format yaml | \
kubectl create configmap app-config \
  --from-file=config.yaml=/dev/stdin \
  --dry-run=client -o yaml
```

### Secret Generation

```bash
#!/bin/bash
# generate-secret.sh

# Pull secrets and create Kubernetes Secret
dotenv pull \
  --project my-app \
  --environment production \
  --format json | \
jq -r 'to_entries | map("\(.key)=\(.value|@base64)") | .[]' | \
kubectl create secret generic app-secrets \
  --from-env-file=/dev/stdin \
  --dry-run=client -o yaml
```

### Helm Integration

```yaml
# helm/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  template:
    spec:
      initContainers:
      - name: secret-fetcher
        image: alpine:latest
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
          apk add --no-cache curl jq
          curl -sSL https://dotenv.cloud/install.sh | sh
          export PATH="/root/.dotenv/bin:$PATH"

          # Pull secrets as JSON
          dotenv pull \
            --project {{ .Values.dotenv.project }} \
            --environment {{ .Values.dotenv.environment }} \
            --format json > /shared/secrets.json

          # Convert to Kubernetes-friendly format
          jq -r 'to_entries|map("export \(.key)=\(.value|@sh)")|.[]' \
            /shared/secrets.json > /shared/secrets.env
        volumeMounts:
        - name: shared-secrets
          mountPath: /shared

      containers:
      - name: app
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        command: ["/bin/sh", "-c"]
        args:
          - |
            source /shared/secrets.env
            exec node server.js
        volumeMounts:
        - name: shared-secrets
          mountPath: /shared
          readOnly: true

      volumes:
      - name: shared-secrets
        emptyDir:
          medium: Memory
```

## Security Best Practices

### Minimal Images

```dockerfile
# Bad: Secrets in final image
FROM node:18
COPY .env .
CMD ["node", "server.js"]

# Good: No secrets in final image
FROM node:18-slim
RUN curl -sSL https://dotenv.cloud/install.sh | sh
ENV PATH="/root/.dotenv/bin:$PATH"
ENTRYPOINT ["dotenv", "run", "--"]
CMD ["node", "server.js"]
```

### Non-root User

```dockerfile
FROM node:18-slim

# Create non-root user
RUN useradd -m -u 1000 appuser

# Install CLI as root
RUN curl -sSL https://dotenv.cloud/install.sh | sh
RUN mv /root/.dotenv /opt/dotenv
RUN chown -R appuser:appuser /opt/dotenv

# Switch to non-root user
USER appuser
ENV PATH="/opt/dotenv/bin:$PATH"

WORKDIR /app
COPY --chown=appuser:appuser . .

ENTRYPOINT ["dotenv", "run", "--"]
CMD ["node", "server.js"]
```

### Read-only Filesystem

```dockerfile
FROM node:18-slim

# Install dependencies
RUN curl -sSL https://dotenv.cloud/install.sh | sh
WORKDIR /app
COPY . .
RUN npm ci --production

# Runtime configuration
USER node
ENV PATH="/root/.dotenv/bin:$PATH"

# Read-only root filesystem
# Secrets written to tmpfs mount
ENTRYPOINT ["sh", "-c", "dotenv pull --output /tmp/.env && source /tmp/.env && exec node server.js"]
```

Docker run:

```bash
docker run \
  --read-only \
  --tmpfs /tmp \
  -e DOTENV_API_KEY="$API_KEY" \
  my-app
```

### Secret Scanning

```dockerfile
# .dockerignore
.env
.env.*
*.key
*.pem
*.crt
secrets/
credentials/

# Dockerfile
FROM node:18

# Scan for secrets during build
RUN apk add --no-cache git
RUN git init && \
    git add . && \
    git secrets --install && \
    git secrets --scan
```

## Container Orchestration

### Docker Swarm

```yaml
# docker-stack.yml
version: "3.8"

services:
    app:
        image: my-app:latest
        deploy:
            replicas: 3
            secrets:
                - dotenv_api_key
        environment:
            - DOTENV_PROJECT=my-app
            - DOTENV_ENVIRONMENT=production
        entrypoint: |
            sh -c '
              export DOTENV_API_KEY=$(cat /run/secrets/dotenv_api_key)
              dotenv pull
              source .env
              exec node server.js
            '

secrets:
    dotenv_api_key:
        external: true
```

### Amazon ECS

```json
{
    "family": "my-app",
    "taskRoleArn": "arn:aws:iam::123456789012:role/ecsTaskRole",
    "containerDefinitions": [
        {
            "name": "secret-fetcher",
            "image": "alpine:latest",
            "essential": false,
            "command": [
                "sh",
                "-c",
                "apk add --no-cache curl && curl -sSL https://dotenv.cloud/install.sh | sh && /root/.dotenv/bin/dotenv pull --output /shared/.env"
            ],
            "environment": [
                {
                    "name": "DOTENV_API_KEY",
                    "valueFrom": "arn:aws:secretsmanager:region:123456789012:secret:dotenv-api-key"
                }
            ],
            "mountPoints": [
                {
                    "sourceVolume": "shared-secrets",
                    "containerPath": "/shared"
                }
            ]
        },
        {
            "name": "app",
            "image": "my-app:latest",
            "essential": true,
            "dependsOn": [
                {
                    "containerName": "secret-fetcher",
                    "condition": "SUCCESS"
                }
            ],
            "entryPoint": ["sh", "-c"],
            "command": ["source /shared/.env && exec node server.js"],
            "mountPoints": [
                {
                    "sourceVolume": "shared-secrets",
                    "containerPath": "/shared",
                    "readOnly": true
                }
            ]
        }
    ],
    "volumes": [
        {
            "name": "shared-secrets",
            "dockerVolumeConfiguration": {
                "scope": "task",
                "driver": "local"
            }
        }
    ]
}
```

## Development Workflows

### Hot Reload with Secrets

```yaml
# docker-compose.dev.yml
version: "3.8"

services:
    app:
        build:
            context: .
            target: development
        volumes:
            - .:/app
            - /app/node_modules
            - secrets:/secrets
        environment:
            - DOTENV_API_KEY
            - CHOKIDAR_USEPOLLING=true
        command: |
            sh -c '
              # Initial secret load
              dotenv pull --output /secrets/.env
              
              # Watch for changes
              nodemon \
                --watch /secrets/.env \
                --exec "source /secrets/.env && node" \
                server.js
            '

    # Secret reloader
    secret-watcher:
        image: alpine:latest
        environment:
            - DOTENV_API_KEY
        volumes:
            - secrets:/secrets
        command: |
            sh -c '
              apk add --no-cache curl inotify-tools
              curl -sSL https://dotenv.cloud/install.sh | sh
              export PATH="/root/.dotenv/bin:$PATH"
              
              while true; do
                dotenv pull --output /secrets/.env.new
                if ! cmp -s /secrets/.env /secrets/.env.new; then
                  mv /secrets/.env.new /secrets/.env
                  echo "Secrets updated at $(date)"
                fi
                sleep 60
              done
            '

volumes:
    secrets:
```

### Testing with Secrets

```dockerfile
# Dockerfile.test
FROM node:18

# Install CLI
RUN curl -sSL https://dotenv.cloud/install.sh | sh
ENV PATH="/root/.dotenv/bin:$PATH"

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .

# Test entrypoint
ENTRYPOINT ["sh", "-c"]
CMD ["dotenv pull --environment test && source .env && npm test"]
```

Run tests:

```bash
docker build -f Dockerfile.test -t my-app-test .
docker run --rm \
  -e DOTENV_API_KEY="$DOTENV_API_KEY" \
  my-app-test
```

## Troubleshooting

### Common Issues

1. **Secrets not loading**

    ```dockerfile
    # Debug entrypoint
    ENTRYPOINT ["sh", "-c", "set -x && dotenv pull -v && source .env && exec $@", "--"]
    ```

2. **Permission denied**

    ```dockerfile
    # Fix permissions
    RUN chmod +x /usr/local/bin/dotenv
    ```

3. **Network issues in container**

    ```dockerfile
    # Add CA certificates
    RUN apk add --no-cache ca-certificates
    ```

4. **Memory issues with secrets**
    ```yaml
    # Use tmpfs with size limit
    volumes:
        secrets:
            driver: local
            driver_opts:
                type: tmpfs
                device: tmpfs
                o: size=10m
    ```

### Debug Container

```dockerfile
# Dockerfile.debug
FROM node:18

# Install debugging tools
RUN apt-get update && apt-get install -y \
    curl \
    vim \
    netcat \
    procps \
    strace

# Install DotEnv CLI
RUN curl -sSL https://dotenv.cloud/install.sh | sh
ENV PATH="/root/.dotenv/bin:$PATH"

WORKDIR /app
COPY . .

# Interactive debug shell
CMD ["/bin/bash"]
```

## Next Steps

- [CI/CD Integration](./ci-cd-usage) - Pipeline integration
- [Kubernetes Guide](/documentation/v1/integrations/kubernetes) - K8s patterns
- [Security Best Practices](/documentation/v1/security/containers) - Container security
- [Troubleshooting](./troubleshooting) - Common Docker issues
