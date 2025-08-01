---
title: Integration Overview
slug: overview
order: 1
tags: [integrations, overview, platforms]
---

# Integration Overview

DotEnv seamlessly integrates with your existing development workflow, CI/CD pipelines, and deployment platforms. This guide provides an overview of available integrations and how to get started.

## Integration Categories

### 1. CI/CD Platforms

Automate secret management in your continuous integration and deployment pipelines:

- [GitHub Actions](./github-actions) - Native GitHub integration
- [GitLab CI](./gitlab-ci) - GitLab pipeline integration
- [CircleCI](./circleci) - CircleCI orb and configuration
- [Jenkins](./jenkins) - Jenkins plugin and pipeline support
- [Bitbucket Pipelines](./bitbucket-pipelines) - Bitbucket integration

### 2. Cloud Platforms

Deploy applications with secure secret injection:

- [Vercel](./vercel) - Serverless deployment integration
- [Netlify](./netlify) - Static site and function deployment
- [Heroku](./heroku) - Heroku app configuration
- [AWS](./aws) - AWS services integration
- [Google Cloud](./google-cloud) - GCP integration
- [Azure](./azure) - Microsoft Azure integration

### 3. Container Platforms

Secure containerized applications:

- [Docker](./docker) - Docker and Docker Compose
- [Kubernetes](./kubernetes) - K8s secrets and ConfigMaps

### 4. Development Tools

Enhance your local development experience:

- IDE plugins for IntelliJ, WebStorm, etc.

## Integration Methods

### 1. CLI Integration

The DotEnv CLI is the foundation for most integrations:

```bash
# Install CLI
curl -fsSL https://cli.dotenv.cloud/install.sh | sh

# Authenticate
dotenv auth login

# Pull secrets
dotenv pull production
```

### 2. API Integration

Direct API integration for custom workflows:

```javascript
const response = await fetch("https://api.dotenv.cloud/v1/secrets", {
    headers: {
        Authorization: `Bearer ${process.env.DOTENV_API_KEY}`,
    },
});
```

### 3. SDK Integration

Native SDKs for popular languages:

```javascript
// JavaScript
import { DotEnv } from '@dotenv/sdk';
const dotenv = new DotEnv({ apiKey: process.env.DOTENV_API_KEY });

// Python
from dotenv_sdk import DotEnv
dotenv = DotEnv(api_key=os.environ['DOTENV_API_KEY'])

// Go
import "github.com/dotenv/dotenv-go"
client := dotenv.New(os.Getenv("DOTENV_API_KEY"))
```

### 4. Webhook Integration

Real-time updates via webhooks:

```json
{
    "event": "secret.updated",
    "project": "api-service",
    "environment": "production"
}
```

## Common Integration Patterns

### 1. Build-Time Injection

Inject secrets during the build process:

```yaml
# GitHub Actions example
- name: Load secrets
  run: |
      dotenv pull production --format=github-actions
      source .env
```

### 2. Runtime Injection

Load secrets at application startup:

```javascript
// Node.js example
const { DotEnv } = require("@dotenv/sdk");

async function loadSecrets() {
    const dotenv = new DotEnv({ apiKey: process.env.DOTENV_API_KEY });
    const secrets = await dotenv.secrets.list("production");

    secrets.forEach((secret) => {
        process.env[secret.key] = secret.value;
    });
}

loadSecrets().then(() => {
    // Start application
    require("./app");
});
```

### 3. Proxy Pattern

Use DotEnv as a proxy for secret access:

```javascript
class SecretManager {
    constructor() {
        this.cache = new Map();
        this.dotenv = new DotEnv({ apiKey: process.env.DOTENV_API_KEY });
    }

    async get(key) {
        if (this.cache.has(key)) {
            return this.cache.get(key);
        }

        const secret = await this.dotenv.secrets.get(key);
        this.cache.set(key, secret.value);
        return secret.value;
    }
}
```

### 4. Sidecar Pattern

Run DotEnv CLI as a sidecar container:

```yaml
# Kubernetes example
containers:
    - name: app
      image: myapp:latest
      envFrom:
          - secretRef:
                name: dotenv-secrets
    - name: dotenv-sidecar
      image: dotenv/cli:latest
      command: ["dotenv", "sync", "--watch"]
```

## Security Best Practices

### 1. API Key Management

- Store API keys in CI/CD secret storage
- Use environment-specific keys
- Rotate keys regularly
- Never commit keys to version control

### 2. Least Privilege

- Create project-specific API keys
- Limit permissions to read-only where possible
- Use environment restrictions

### 3. Audit Trail

- Enable webhook notifications
- Monitor secret access logs
- Set up alerts for unauthorized access

### 4. Network Security

- Whitelist CI/CD IP addresses
- Use private network endpoints where available
- Enable request signing

## Integration Checklist

Before integrating DotEnv:

- [ ] **Choose Integration Method**

    - [ ] CLI for simple use cases
    - [ ] API/SDK for programmatic access
    - [ ] Webhooks for real-time updates

- [ ] **Set Up Authentication**

    - [ ] Create API key with appropriate permissions
    - [ ] Store key securely in CI/CD platform
    - [ ] Test authentication

- [ ] **Configure Environment**

    - [ ] Define environment strategy (dev, staging, prod)
    - [ ] Set up secret inheritance
    - [ ] Configure access controls

- [ ] **Implement Integration**

    - [ ] Follow platform-specific guide
    - [ ] Test in non-production first
    - [ ] Set up monitoring

- [ ] **Security Review**
    - [ ] Review API key permissions
    - [ ] Enable audit logging
    - [ ] Set up alerts

## Troubleshooting

### Common Issues

1. **Authentication Failures**

    - Verify API key is valid
    - Check key permissions
    - Ensure correct environment

2. **Network Errors**

    - Check firewall rules
    - Verify API endpoint accessibility
    - Review proxy settings

3. **Permission Errors**

    - Confirm user has required permissions
    - Check organization/project access
    - Review environment restrictions

4. **Integration Conflicts**
    - Check for existing .env files
    - Verify no variable name conflicts
    - Review load order

### Debug Mode

Enable debug logging:

```bash
# CLI
DOTENV_DEBUG=true dotenv pull

# SDK
const dotenv = new DotEnv({
  apiKey: process.env.DOTENV_API_KEY,
  debug: true
});
```

## Getting Help

### Resources

- [API Documentation](/documentation/v1/api/overview)
- [CLI Reference](/documentation/v1/cli/overview)
- [SDK Documentation](#)
- [Security Best Practices](/documentation/v1/security/best-practices)

### Support

- GitHub Issues: Report bugs and request features
- Discord Community: Get help from the community
- Enterprise Support: Priority support for enterprise customers

## Next Steps

Choose your integration path:

1. **CI/CD Integration**

    - [GitHub Actions](./github-actions)
    - [GitLab CI](./gitlab-ci)
    - [CircleCI](./circleci)

2. **Cloud Platform Integration**

    - [Vercel](./vercel)
    - [AWS](./aws)
    - [Google Cloud](./google-cloud)

3. **Container Integration**

    - [Docker](./docker)
    - [Kubernetes](./kubernetes)

4. **Development Tools**
    - [VS Code Extension](/documentation/v1/drafts/integrations/vscode) *(Coming Soon)*
