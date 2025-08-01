---
title: Microservices Architecture
slug: microservices
order: 1
tags: [use-cases, microservices, architecture, patterns]
---

# Microservices Architecture

Learn how to manage secrets across microservices architectures using DotEnv. This guide covers patterns for secret distribution, service discovery, and maintaining consistency across distributed systems.

## Overview

Managing secrets in microservices presents unique challenges:

- Service-specific secrets
- Shared credentials
- Service discovery
- Secret rotation coordination
- Consistency across services
- Development environment isolation

## Architecture Patterns

### 1. Service-Specific Projects

Organize secrets by service:

```
Organization: my-company
├── Projects:
│   ├── auth-service
│   ├── payment-service
│   ├── notification-service
│   ├── user-service
│   └── shared-resources
```

### 2. Environment Structure

```yaml
# Each service has consistent environments
auth-service:
    - development
    - staging
    - production

payment-service:
    - development
    - staging
    - production
```

### 3. Shared Secrets Pattern

```javascript
// shared-resources project
{
  "DATABASE_HOST": "db.internal.company.com",
  "REDIS_HOST": "redis.internal.company.com",
  "RABBITMQ_URL": "amqp://rabbitmq.internal.company.com",
  "JAEGER_ENDPOINT": "http://jaeger.internal.company.com:14268"
}

// service-specific project inherits shared
{
  "inherit_from": "shared-resources",
  "SERVICE_PORT": "3001",
  "SERVICE_NAME": "auth-service",
  "JWT_SECRET": "service-specific-secret"
}
```

## Implementation Examples

### Docker Compose Setup

```yaml
# docker-compose.yml
version: "3.8"

services:
    auth-service:
        build: ./auth-service
        environment:
            - DOTENV_PROJECT=auth-service
            - DOTENV_ENVIRONMENT=${ENVIRONMENT:-development}
        env_file:
            - .env.auth
        depends_on:
            - postgres
            - redis
        networks:
            - microservices

    payment-service:
        build: ./payment-service
        environment:
            - DOTENV_PROJECT=payment-service
            - DOTENV_ENVIRONMENT=${ENVIRONMENT:-development}
        env_file:
            - .env.payment
        depends_on:
            - postgres
            - rabbitmq
        networks:
            - microservices

    notification-service:
        build: ./notification-service
        environment:
            - DOTENV_PROJECT=notification-service
            - DOTENV_ENVIRONMENT=${ENVIRONMENT:-development}
        env_file:
            - .env.notification
        depends_on:
            - rabbitmq
        networks:
            - microservices

networks:
    microservices:
        driver: bridge
```

### Service Initialization

```javascript
// services/auth/index.js
const { loadSecrets } = require("./config/dotenv");
const express = require("express");
const { connectDatabase } = require("./db");
const { initializeMessageQueue } = require("./queue");

async function startService() {
    // Load service-specific secrets
    await loadSecrets("auth-service", process.env.ENVIRONMENT);

    // Initialize connections
    await connectDatabase();
    await initializeMessageQueue();

    const app = express();

    // Service configuration
    app.use(express.json());
    app.use("/health", require("./routes/health"));
    app.use("/auth", require("./routes/auth"));

    const port = process.env.SERVICE_PORT || 3000;
    app.listen(port, () => {
        console.log(`Auth service running on port ${port}`);
    });
}

startService().catch(console.error);
```

### Shared Configuration Loader

```javascript
// shared/config/dotenv.js
const { execSync } = require("child_process");
const fs = require("fs");

async function loadSecrets(projectName, environment) {
    try {
        // Load shared secrets first
        execSync(
            `dotenv pull ${environment} --project shared-resources --output .env.shared`,
            { stdio: "inherit" },
        );

        // Load service-specific secrets
        execSync(
            `dotenv pull ${environment} --project ${projectName} --output .env.service`,
            { stdio: "inherit" },
        );

        // Merge secrets (service-specific overrides shared)
        const shared = parseEnvFile(".env.shared");
        const service = parseEnvFile(".env.service");

        const merged = { ...shared, ...service };

        // Apply to process.env
        Object.entries(merged).forEach(([key, value]) => {
            process.env[key] = value;
        });

        // Clean up temporary files
        fs.unlinkSync(".env.shared");
        fs.unlinkSync(".env.service");

        console.log(`Loaded secrets for ${projectName} (${environment})`);
    } catch (error) {
        console.error("Failed to load secrets:", error);
        throw error;
    }
}

function parseEnvFile(filename) {
    const content = fs.readFileSync(filename, "utf8");
    const result = {};

    content.split("\n").forEach((line) => {
        const match = line.match(/^([^=]+)=(.*)$/);
        if (match) {
            result[match[1]] = match[2];
        }
    });

    return result;
}

module.exports = { loadSecrets };
```

## Kubernetes Deployment

### ConfigMap Generator

```yaml
# k8s/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

configMapGenerator:
    - name: service-configs
      files:
          - configs/services.yaml

secretGenerator:
    - name: dotenv-credentials
      literals:
          - api-key=${DOTENV_API_KEY}
```

### Service Deployment

```yaml
# k8s/services/auth-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: auth-service
spec:
    replicas: 3
    selector:
        matchLabels:
            app: auth-service
    template:
        metadata:
            labels:
                app: auth-service
        spec:
            initContainers:
                - name: secret-loader
                  image: dotenv/cli:latest
                  env:
                      - name: DOTENV_API_KEY
                        valueFrom:
                            secretKeyRef:
                                name: dotenv-credentials
                                key: api-key
                      - name: SERVICE_NAME
                        value: auth-service
                      - name: ENVIRONMENT
                        value: production
                  command:
                      - sh
                      - -c
                      - |
                          dotenv auth login --api-key $DOTENV_API_KEY
                          dotenv pull $ENVIRONMENT --project $SERVICE_NAME --output /shared/secrets/.env
                  volumeMounts:
                      - name: shared-secrets
                        mountPath: /shared/secrets

            containers:
                - name: auth-service
                  image: myregistry/auth-service:latest
                  envFrom:
                      - configMapRef:
                            name: service-configs
                  volumeMounts:
                      - name: shared-secrets
                        mountPath: /secrets
                        readOnly: true
                  command: ["/bin/sh"]
                  args: ["-c", "source /secrets/.env && npm start"]

            volumes:
                - name: shared-secrets
                  emptyDir: {}
```

## Service Communication

### Service Registry Integration

```javascript
// shared/discovery/consul.js
const consul = require("consul")();

class ServiceRegistry {
    constructor(serviceName) {
        this.serviceName = serviceName;
        this.serviceId = `${serviceName}-${process.env.INSTANCE_ID || Date.now()}`;
    }

    async register() {
        const service = {
            name: this.serviceName,
            id: this.serviceId,
            address: process.env.SERVICE_HOST || "localhost",
            port: parseInt(process.env.SERVICE_PORT) || 3000,
            check: {
                http: `http://localhost:${process.env.SERVICE_PORT}/health`,
                interval: "10s",
                timeout: "5s",
            },
            tags: [process.env.ENVIRONMENT, process.env.VERSION],
        };

        await consul.agent.service.register(service);
        console.log(`Registered ${this.serviceName} with Consul`);

        // Graceful shutdown
        process.on("SIGTERM", async () => {
            await this.deregister();
            process.exit(0);
        });
    }

    async deregister() {
        await consul.agent.service.deregister(this.serviceId);
        console.log(`Deregistered ${this.serviceName} from Consul`);
    }

    async getService(serviceName) {
        const services = await consul.health.service(serviceName);
        const healthy = services.filter((s) =>
            s.Checks.every((check) => check.Status === "passing"),
        );

        if (healthy.length === 0) {
            throw new Error(`No healthy instances of ${serviceName} found`);
        }

        // Load balance (round-robin)
        const instance = healthy[Math.floor(Math.random() * healthy.length)];
        return {
            address: instance.Service.Address,
            port: instance.Service.Port,
        };
    }
}

module.exports = ServiceRegistry;
```

### Inter-Service Authentication

```javascript
// shared/auth/service-auth.js
const jwt = require("jsonwebtoken");

class ServiceAuth {
    constructor() {
        this.serviceSecret = process.env.INTER_SERVICE_SECRET;
        this.serviceName = process.env.SERVICE_NAME;
    }

    generateToken(targetService) {
        return jwt.sign(
            {
                service: this.serviceName,
                target: targetService,
                timestamp: Date.now(),
            },
            this.serviceSecret,
            { expiresIn: "30s" },
        );
    }

    verifyToken(token, expectedSource) {
        try {
            const payload = jwt.verify(token, this.serviceSecret);

            if (payload.target !== this.serviceName) {
                throw new Error("Token not intended for this service");
            }

            if (expectedSource && payload.service !== expectedSource) {
                throw new Error("Token from unexpected service");
            }

            return payload;
        } catch (error) {
            throw new Error(`Service authentication failed: ${error.message}`);
        }
    }

    middleware(expectedSource) {
        return (req, res, next) => {
            const token = req.headers["x-service-token"];

            if (!token) {
                return res
                    .status(401)
                    .json({ error: "Service token required" });
            }

            try {
                req.serviceAuth = this.verifyToken(token, expectedSource);
                next();
            } catch (error) {
                res.status(401).json({ error: error.message });
            }
        };
    }
}

module.exports = ServiceAuth;
```

## Development Workflow

### Local Development Setup

```bash
#!/bin/bash
# scripts/dev-setup.sh

# Load development secrets for all services
services=("auth-service" "payment-service" "notification-service" "user-service")

for service in "${services[@]}"; do
  echo "Loading secrets for $service..."
  dotenv pull development --project $service --output .env.$service
done

# Start services with docker-compose
docker-compose up -d

# Wait for services to be healthy
./scripts/wait-for-healthy.sh

echo "Microservices development environment ready!"
```

### Service Template

```javascript
// templates/service/index.js
const express = require("express");
const { ServiceRegistry } = require("@shared/discovery");
const { ServiceAuth } = require("@shared/auth");
const { loadSecrets } = require("@shared/config");
const { initializeTracing } = require("@shared/tracing");
const { healthCheck } = require("@shared/health");

class MicroserviceBase {
    constructor(serviceName) {
        this.serviceName = serviceName;
        this.app = express();
        this.registry = new ServiceRegistry(serviceName);
        this.auth = new ServiceAuth();
    }

    async initialize() {
        // Load configuration
        await loadSecrets(this.serviceName, process.env.ENVIRONMENT);

        // Initialize tracing
        initializeTracing(this.serviceName);

        // Common middleware
        this.app.use(express.json());
        this.app.use(this.correlationId());
        this.app.use(this.requestLogging());

        // Health check
        this.app.get("/health", healthCheck);

        // Register with service discovery
        await this.registry.register();
    }

    correlationId() {
        return (req, res, next) => {
            req.correlationId =
                req.headers["x-correlation-id"] || require("uuid").v4();
            res.setHeader("x-correlation-id", req.correlationId);
            next();
        };
    }

    requestLogging() {
        return (req, res, next) => {
            const start = Date.now();
            res.on("finish", () => {
                console.log({
                    service: this.serviceName,
                    correlationId: req.correlationId,
                    method: req.method,
                    path: req.path,
                    status: res.statusCode,
                    duration: Date.now() - start,
                });
            });
            next();
        };
    }

    async callService(serviceName, path, options = {}) {
        const service = await this.registry.getService(serviceName);
        const token = this.auth.generateToken(serviceName);

        const response = await fetch(
            `http://${service.address}:${service.port}${path}`,
            {
                ...options,
                headers: {
                    ...options.headers,
                    "x-service-token": token,
                    "x-correlation-id": options.correlationId,
                },
            },
        );

        if (!response.ok) {
            throw new Error(`Service call failed: ${response.statusText}`);
        }

        return response.json();
    }

    start(port) {
        this.app.listen(port, () => {
            console.log(`${this.serviceName} running on port ${port}`);
        });
    }
}

module.exports = MicroserviceBase;
```

## Secret Rotation

### Coordinated Rotation

```javascript
// scripts/rotate-secrets.js
const { DotEnvClient } = require("@dotenv/sdk");

async function rotateServiceSecrets() {
    const client = new DotEnvClient({
        apiKey: process.env.DOTENV_API_KEY,
    });

    const services = [
        "auth-service",
        "payment-service",
        "notification-service",
        "user-service",
    ];

    const environment = process.env.ENVIRONMENT || "production";

    // Phase 1: Generate new secrets
    console.log("Phase 1: Generating new secrets...");
    const newSecrets = {};

    for (const service of services) {
        newSecrets[service] = {
            JWT_SECRET: generateSecret(),
            API_KEY: generateSecret(),
            INTER_SERVICE_SECRET: generateSecret(),
        };
    }

    // Phase 2: Add new secrets alongside old
    console.log("Phase 2: Adding new secrets...");
    for (const service of services) {
        const secrets = await client.getSecrets(service, environment);

        // Add new secrets with _NEW suffix
        for (const [key, value] of Object.entries(newSecrets[service])) {
            secrets[`${key}_NEW`] = value;
        }

        await client.updateSecrets(service, environment, secrets);
    }

    // Phase 3: Deploy services to use new secrets
    console.log("Phase 3: Rolling deployment...");
    await executeRollingDeployment(services);

    // Phase 4: Remove old secrets
    console.log("Phase 4: Cleaning up old secrets...");
    for (const service of services) {
        const secrets = await client.getSecrets(service, environment);

        // Remove _NEW suffix and old secrets
        const updated = {};
        for (const [key, value] of Object.entries(secrets)) {
            if (key.endsWith("_NEW")) {
                updated[key.replace("_NEW", "")] = value;
            } else if (!Object.keys(newSecrets[service]).includes(key)) {
                updated[key] = value;
            }
        }

        await client.updateSecrets(service, environment, updated);
    }

    console.log("Secret rotation completed successfully!");
}

function generateSecret() {
    return require("crypto").randomBytes(32).toString("base64");
}

async function executeRollingDeployment(services) {
    // Implement your deployment strategy
    // This could trigger CI/CD pipelines, Kubernetes rollouts, etc.
}

rotateServiceSecrets().catch(console.error);
```

## Monitoring and Observability

### Distributed Tracing

```javascript
// shared/tracing/index.js
const { NodeTracerProvider } = require("@opentelemetry/sdk-trace-node");
const { Resource } = require("@opentelemetry/resources");
const {
    SemanticResourceAttributes,
} = require("@opentelemetry/semantic-conventions");
const { JaegerExporter } = require("@opentelemetry/exporter-jaeger");
const { BatchSpanProcessor } = require("@opentelemetry/sdk-trace-base");
const { registerInstrumentations } = require("@opentelemetry/instrumentation");
const { HttpInstrumentation } = require("@opentelemetry/instrumentation-http");
const {
    ExpressInstrumentation,
} = require("@opentelemetry/instrumentation-express");

function initializeTracing(serviceName) {
    const provider = new NodeTracerProvider({
        resource: new Resource({
            [SemanticResourceAttributes.SERVICE_NAME]: serviceName,
            [SemanticResourceAttributes.SERVICE_VERSION]:
                process.env.VERSION || "1.0.0",
            [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]:
                process.env.ENVIRONMENT,
        }),
    });

    const exporter = new JaegerExporter({
        endpoint:
            process.env.JAEGER_ENDPOINT || "http://localhost:14268/api/traces",
    });

    provider.addSpanProcessor(new BatchSpanProcessor(exporter));
    provider.register();

    registerInstrumentations({
        instrumentations: [
            new HttpInstrumentation({
                requestHook: (span, request) => {
                    span.setAttributes({
                        "http.request.correlation_id":
                            request.headers["x-correlation-id"],
                    });
                },
            }),
            new ExpressInstrumentation(),
        ],
    });

    return provider;
}

module.exports = { initializeTracing };
```

## Best Practices

### 1. Service Isolation

- Each service has its own DotEnv project
- No direct database access between services
- API contracts for inter-service communication

### 2. Configuration Management

```yaml
# config/services.yaml
services:
    auth-service:
        dependencies:
            - postgres
            - redis
        secrets:
            - JWT_SECRET
            - OAUTH_CLIENT_ID
            - OAUTH_CLIENT_SECRET

    payment-service:
        dependencies:
            - postgres
            - stripe-api
        secrets:
            - STRIPE_SECRET_KEY
            - WEBHOOK_SECRET
```

### 3. Development Parity

```bash
# Ensure all developers have same setup
dotenv init microservices --template microservices
dotenv sync --all-projects --environment development
```

## Troubleshooting

### Service Discovery Issues

```bash
# Check service registration
consul catalog services
consul catalog nodes -service=auth-service

# Verify health checks
consul health checks auth-service
```

### Secret Loading Failures

```javascript
// Add retry logic for secret loading
async function loadSecretsWithRetry(project, environment, maxRetries = 3) {
    for (let i = 0; i < maxRetries; i++) {
        try {
            await loadSecrets(project, environment);
            return;
        } catch (error) {
            console.error(`Attempt ${i + 1} failed:`, error);
            if (i === maxRetries - 1) throw error;
            await new Promise((resolve) => setTimeout(resolve, 2000 * (i + 1)));
        }
    }
}
```

## Resources

- [Microservices Patterns](https://microservices.io/patterns/index.html)
- [12 Factor App](https://12factor.net/)
- [Service Mesh Comparison](https://servicemesh.io/)
- [DotEnv Best Practices](/documentation/v1/core-concepts/security-model)

## Next Steps

- [Set up service projects](#architecture-patterns)
- [Implement service discovery](#service-communication)
- [Configure monitoring](#monitoring-and-observability)
- [Plan secret rotation](#secret-rotation)
