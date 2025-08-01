---
title: Docker Integration
slug: docker
order: 5
tags: [integrations, docker, containers, docker-compose]
---

# Docker Integration

Integrate DotEnv with Docker to securely manage secrets in containerized applications. This guide covers Docker, Docker Compose, and container orchestration patterns.

## Overview

The DotEnv Docker integration enables:

- Secure secret injection at build and runtime
- Multi-stage build support
- Docker Compose integration
- Development and production workflows
- Container orchestration compatibility

## Installation

### Using DotEnv CLI in Docker

Create a Dockerfile with DotEnv CLI:

```dockerfile
FROM alpine:latest AS dotenv
RUN apk add --no-cache curl bash
RUN curl -fsSL https://cli.dotenv.cloud/install.sh | sh
ENV PATH="/root/.dotenv/bin:$PATH"

FROM node:18-alpine
COPY --from=dotenv /root/.dotenv /root/.dotenv
ENV PATH="/root/.dotenv/bin:$PATH"
```

### Docker Official Image

Use the official DotEnv Docker image:

```dockerfile
FROM dotenv/cli:latest AS secrets
ARG DOTENV_API_KEY
ARG PROJECT_NAME
ARG ENVIRONMENT
RUN dotenv auth login --api-key $DOTENV_API_KEY && \
    dotenv pull $ENVIRONMENT --project $PROJECT_NAME --format docker > /secrets.env

FROM node:18-alpine
COPY --from=secrets /secrets.env /.env
```

## Build-Time Secrets

### 1. BuildKit Secrets (Recommended)

Use Docker BuildKit for secure secret handling:

```dockerfile
# syntax=docker/dockerfile:1
FROM node:18-alpine

# Mount secret file during build
RUN --mount=type=secret,id=dotenv \
    cat /run/secrets/dotenv > .env && \
    npm install && \
    npm run build && \
    rm .env
```

Build command:

```bash
# First, get secrets from DotEnv
dotenv pull production --format docker > .env.production

# Build with BuildKit
DOCKER_BUILDKIT=1 docker build \
  --secret id=dotenv,src=.env.production \
  -t myapp:latest .
```

### 2. Multi-Stage Builds

Separate secret fetching from application build:

```dockerfile
# Stage 1: Fetch secrets
FROM dotenv/cli:latest AS secrets
ARG DOTENV_API_KEY
ARG PROJECT_NAME=my-app
ARG ENVIRONMENT=production
RUN dotenv auth login --api-key $DOTENV_API_KEY && \
    dotenv pull $ENVIRONMENT --project $PROJECT_NAME --format docker > /secrets.env

# Stage 2: Build application
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
COPY --from=secrets /secrets.env .env
RUN npm ci --only=production
COPY . .
RUN npm run build

# Stage 3: Runtime (no secrets)
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
```

### 3. Build Arguments

Pass secrets as build arguments (less secure):

```dockerfile
FROM node:18-alpine

# Accept build arguments
ARG DATABASE_URL
ARG API_KEY
ARG JWT_SECRET

# Use during build
RUN echo "DATABASE_URL=$DATABASE_URL" >> .env && \
    echo "API_KEY=$API_KEY" >> .env && \
    echo "JWT_SECRET=$JWT_SECRET" >> .env && \
    npm run build && \
    rm .env

CMD ["node", "dist/index.js"]
```

Build command:

```bash
# Load secrets into shell
source <(dotenv pull production --format shell)

# Build with arguments
docker build \
  --build-arg DATABASE_URL="$DATABASE_URL" \
  --build-arg API_KEY="$API_KEY" \
  --build-arg JWT_SECRET="$JWT_SECRET" \
  -t myapp:latest .
```

## Runtime Secrets

### 1. Environment Variables

Inject secrets at container runtime:

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN npm ci --only=production

# Don't include secrets in image
CMD ["node", "index.js"]
```

Run with environment file:

```bash
# Generate env file
dotenv pull production --format docker > .env.production

# Run container
docker run --env-file .env.production myapp:latest

# Or with docker-compose
docker-compose --env-file .env.production up
```

### 2. Init Container Pattern

Fetch secrets at startup:

```dockerfile
FROM node:18-alpine
WORKDIR /app

# Install DotEnv CLI
COPY --from=dotenv/cli:latest /usr/local/bin/dotenv /usr/local/bin/dotenv

# Copy application
COPY . .
RUN npm ci --only=production

# Entrypoint script
COPY docker-entrypoint.sh /
RUN chmod +x /docker-entrypoint.sh

ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["node", "index.js"]
```

`docker-entrypoint.sh`:

```bash
#!/bin/sh
set -e

# Fetch secrets if API key is provided
if [ -n "$DOTENV_API_KEY" ]; then
  echo "Fetching secrets from DotEnv..."
  dotenv auth login --api-key $DOTENV_API_KEY
  dotenv pull $ENVIRONMENT --project $PROJECT_NAME

  # Export to environment
  set -a
  source .env
  set +a

  # Clean up
  rm .env
fi

# Execute main command
exec "$@"
```

### 3. Sidecar Pattern

Run DotEnv as a sidecar container:

```yaml
version: "3.8"

services:
    dotenv-sidecar:
        image: dotenv/cli:latest
        environment:
            DOTENV_API_KEY: ${DOTENV_API_KEY}
            PROJECT_NAME: my-app
            ENVIRONMENT: production
        volumes:
            - secrets:/secrets
        command: |
            sh -c "
              dotenv auth login --api-key $$DOTENV_API_KEY &&
              while true; do
                dotenv pull $$ENVIRONMENT --project $$PROJECT_NAME --format docker > /secrets/.env
                sleep 300
              done
            "

    app:
        build: .
        depends_on:
            - dotenv-sidecar
        volumes:
            - secrets:/secrets:ro
        command: |
            sh -c "
              while [ ! -f /secrets/.env ]; do sleep 1; done
              source /secrets/.env && node index.js
            "

volumes:
    secrets:
```

## Docker Compose Integration

### Basic Configuration

```yaml
version: "3.8"

services:
    app:
        build: .
        env_file:
            - .env
        environment:
            NODE_ENV: production
        ports:
            - "3000:3000"
```

Load secrets before running:

```bash
# Pull secrets
dotenv pull production --format docker > .env

# Start services
docker-compose up -d

# Clean up
rm .env
```

### Advanced Docker Compose

```yaml
version: "3.8"

x-common-variables: &common-variables
    NODE_ENV: ${NODE_ENV:-production}
    LOG_LEVEL: ${LOG_LEVEL:-info}

services:
    api:
        build:
            context: ./services/api
            args:
                - DOTENV_API_KEY=${DOTENV_API_KEY}
                - ENVIRONMENT=${ENVIRONMENT:-production}
        environment:
            <<: *common-variables
            SERVICE_NAME: api
        env_file:
            - .env.api
        ports:
            - "3000:3000"
        healthcheck:
            test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
            interval: 30s
            timeout: 10s
            retries: 3

    worker:
        build:
            context: ./services/worker
            args:
                - DOTENV_API_KEY=${DOTENV_API_KEY}
                - ENVIRONMENT=${ENVIRONMENT:-production}
        environment:
            <<: *common-variables
            SERVICE_NAME: worker
        env_file:
            - .env.worker
        depends_on:
            - redis
            - postgres

    redis:
        image: redis:7-alpine
        volumes:
            - redis-data:/data

    postgres:
        image: postgres:15-alpine
        environment:
            POSTGRES_DB: ${POSTGRES_DB}
            POSTGRES_USER: ${POSTGRES_USER}
            POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
        volumes:
            - postgres-data:/var/lib/postgresql/data

volumes:
    redis-data:
    postgres-data:
```

### Development vs Production

```yaml
# docker-compose.yml (base)
version: '3.8'

services:
  app:
    build: .
    ports:
      - "${PORT:-3000}:3000"

# docker-compose.dev.yml
version: '3.8'

services:
  app:
    build:
      context: .
      target: development
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      NODE_ENV: development
    command: npm run dev

# docker-compose.prod.yml
version: '3.8'

services:
  app:
    build:
      context: .
      target: production
    environment:
      NODE_ENV: production
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
```

Usage:

```bash
# Development
dotenv pull development --format docker > .env
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up

# Production
dotenv pull production --format docker > .env
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

## Security Best Practices

### 1. Never Commit Secrets

`.dockerignore`:

```
.env
.env.*
*.env
.dotenv
secrets/
*.key
*.pem
```

### 2. Use Secret Management

Docker Swarm secrets:

```bash
# Create secret from DotEnv
dotenv pull production --format docker | docker secret create app_secrets -

# Use in service
docker service create \
  --name myapp \
  --secret app_secrets \
  myapp:latest
```

In container:

```bash
# Secrets available at /run/secrets/app_secrets
source /run/secrets/app_secrets
```

### 3. Minimal Secret Exposure

```dockerfile
# Bad - Secrets in final image
FROM node:18-alpine
COPY .env .
RUN npm install
CMD ["node", "index.js"]

# Good - Secrets only during build
FROM node:18-alpine AS builder
RUN --mount=type=secret,id=dotenv,target=/tmp/.env \
    cp /tmp/.env . && \
    npm install && \
    npm run build && \
    rm .env

FROM node:18-alpine
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
```

### 4. Runtime Validation

Validate secrets at startup:

```javascript
// validate-env.js
const required = ["DATABASE_URL", "REDIS_URL", "JWT_SECRET", "API_KEY"];

const missing = required.filter((key) => !process.env[key]);

if (missing.length > 0) {
    console.error(
        `Missing required environment variables: ${missing.join(", ")}`,
    );
    process.exit(1);
}

console.log("Environment validation passed");
```

Dockerfile:

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN npm ci --only=production

# Validate before starting
CMD ["sh", "-c", "node validate-env.js && node index.js"]
```

## Development Workflow

### 1. Local Development

`docker-compose.dev.yml`:

```yaml
version: "3.8"

services:
    app:
        build:
            context: .
            dockerfile: Dockerfile.dev
        volumes:
            - .:/app
            - /app/node_modules
        environment:
            - NODE_ENV=development
        env_file:
            - .env.development
        ports:
            - "3000:3000"
            - "9229:9229" # Debugger
        command: npm run dev
```

`Dockerfile.dev`:

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Install DotEnv CLI
RUN apk add --no-cache curl bash
RUN curl -fsSL https://cli.dotenv.cloud/install.sh | sh
ENV PATH="/root/.dotenv/bin:$PATH"

# Development dependencies
COPY package*.json ./
RUN npm install

# Watch for changes
CMD ["npm", "run", "dev"]
```

### 2. Hot Reload Setup

```json
// package.json
{
    "scripts": {
        "dev": "nodemon --inspect=0.0.0.0:9229 index.js",
        "dev:docker": "docker-compose -f docker-compose.dev.yml up",
        "dev:rebuild": "docker-compose -f docker-compose.dev.yml up --build"
    }
}
```

### 3. Multiple Environments

```bash
#!/bin/bash
# scripts/docker-dev.sh

# Determine environment
ENV=${1:-development}

# Pull secrets
echo "Loading $ENV secrets..."
dotenv pull $ENV --format docker > .env.$ENV

# Start containers
docker-compose \
  -f docker-compose.yml \
  -f docker-compose.$ENV.yml \
  --env-file .env.$ENV \
  up

# Cleanup
rm .env.$ENV
```

## Production Deployment

### 1. CI/CD Pipeline

`.github/workflows/docker.yml`:

```yaml
name: Build and Deploy

on:
    push:
        branches: [main]

jobs:
    build:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v3

            - name: Load DotEnv secrets
              run: |
                  curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                  export PATH="$HOME/.dotenv/bin:$PATH"
                  dotenv auth login --api-key ${{ secrets.DOTENV_API_KEY }}
                  dotenv pull production --format docker > .env.production

            - name: Build Docker image
              run: |
                  DOCKER_BUILDKIT=1 docker build \
                    --secret id=dotenv,src=.env.production \
                    --tag ${{ github.repository }}:${{ github.sha }} \
                    --tag ${{ github.repository }}:latest \
                    .

            - name: Push to registry
              run: |
                  echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
                  docker push ${{ github.repository }}:${{ github.sha }}
                  docker push ${{ github.repository }}:latest
```

### 2. Container Registry

Push to different registries:

```bash
# Docker Hub
docker tag myapp:latest username/myapp:latest
docker push username/myapp:latest

# GitHub Container Registry
docker tag myapp:latest ghcr.io/username/myapp:latest
docker push ghcr.io/username/myapp:latest

# AWS ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $ECR_REGISTRY
docker tag myapp:latest $ECR_REGISTRY/myapp:latest
docker push $ECR_REGISTRY/myapp:latest
```

## Troubleshooting

### Common Issues

#### Secrets Not Loading

```dockerfile
# Debug secrets loading
FROM node:18-alpine
WORKDIR /app

# Add debugging
RUN --mount=type=secret,id=dotenv \
    echo "Secret file exists: $(test -f /run/secrets/dotenv && echo 'yes' || echo 'no')" && \
    echo "Secret file size: $(stat -c%s /run/secrets/dotenv 2>/dev/null || echo '0')" && \
    echo "First line: $(head -n1 /run/secrets/dotenv 2>/dev/null || echo 'empty')"
```

#### Permission Issues

```dockerfile
# Fix permissions
FROM node:18-alpine
WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Install as root
COPY package*.json ./
RUN npm ci --only=production

# Switch to non-root
USER nodejs
COPY --chown=nodejs:nodejs . .

CMD ["node", "index.js"]
```

#### Build Cache Issues

```bash
# Clear Docker build cache
docker builder prune -a

# Build without cache
docker build --no-cache -t myapp:latest .

# Use specific cache invalidation
docker build --build-arg CACHEBUST=$(date +%s) -t myapp:latest .
```

## Examples

### Complete Microservices Setup

```yaml
# docker-compose.yml
version: "3.8"

x-dotenv-common: &dotenv-common
    DOTENV_API_KEY: ${DOTENV_API_KEY}
    PROJECT_PREFIX: myapp

services:
    # API Gateway
    gateway:
        build:
            context: ./services/gateway
            args:
                <<: *dotenv-common
                SERVICE_NAME: gateway
        ports:
            - "80:80"
        depends_on:
            - auth
            - api
        healthcheck:
            test: ["CMD", "curl", "-f", "http://localhost/health"]

    # Auth Service
    auth:
        build:
            context: ./services/auth
            args:
                <<: *dotenv-common
                SERVICE_NAME: auth
        environment:
            SERVICE_NAME: auth
            PORT: 3001
        depends_on:
            - postgres
            - redis

    # API Service
    api:
        build:
            context: ./services/api
            args:
                <<: *dotenv-common
                SERVICE_NAME: api
        environment:
            SERVICE_NAME: api
            PORT: 3002
        depends_on:
            - postgres
            - redis

    # Background Worker
    worker:
        build:
            context: ./services/worker
            args:
                <<: *dotenv-common
                SERVICE_NAME: worker
        environment:
            SERVICE_NAME: worker
        depends_on:
            - redis
            - postgres

    # Databases
    postgres:
        image: postgres:15-alpine
        environment:
            POSTGRES_PASSWORD_FILE: /run/secrets/db_password
        secrets:
            - db_password
        volumes:
            - postgres-data:/var/lib/postgresql/data

    redis:
        image: redis:7-alpine
        command: redis-server --requirepass ${REDIS_PASSWORD}
        volumes:
            - redis-data:/data

secrets:
    db_password:
        external: true

volumes:
    postgres-data:
    redis-data:
```

### Kubernetes Migration

For Kubernetes deployments, secrets can be synchronized:

```bash
# Export secrets for Kubernetes
dotenv pull production --format json | \
  jq -r 'to_entries|map("  \(.key): \(.value|@base64)")|.[]' | \
  kubectl create secret generic app-secrets --from-file=-
```

## Resources

- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Reference](https://docs.docker.com/compose/)
- [BuildKit Documentation](https://docs.docker.com/develop/develop-images/build_enhancements/)
- [DotEnv CLI Reference](/documentation/v1/cli/commands)

## Next Steps

- [Set up BuildKit secrets](#buildkit-secrets-recommended)
- [Configure Docker Compose](#docker-compose-integration)
- [Implement security best practices](#security-best-practices)
- [Explore Kubernetes integration](./kubernetes)
