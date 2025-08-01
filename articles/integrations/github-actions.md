---
title: GitHub Actions Integration
slug: github-actions
order: 2
tags: [integrations, github, ci-cd, github-actions]
---

# GitHub Actions Integration

Integrate DotEnv with GitHub Actions to securely manage secrets in your CI/CD workflows. This guide covers setup, usage patterns, and best practices.

## Overview

The DotEnv GitHub Actions integration allows you to:

- Pull secrets from DotEnv into your workflows
- Sync secrets between environments
- Validate secret configurations
- Automate secret rotation
- Audit secret usage in CI/CD

## Installation

### 1. GitHub Action

Use the official DotEnv GitHub Action:

```yaml
- name: Load DotEnv secrets
  uses: dotenv/actions/load@v1
  with:
      api-key: ${{ secrets.DOTENV_API_KEY }}
      project: my-project
      environment: production
```

### 2. CLI Method

Install and use the DotEnv CLI directly:

```yaml
- name: Install DotEnv CLI
  run: |
      curl -fsSL https://cli.dotenv.cloud/install.sh | sh
      echo "$HOME/.dotenv/bin" >> $GITHUB_PATH

- name: Load secrets
  env:
      DOTENV_API_KEY: ${{ secrets.DOTENV_API_KEY }}
  run: |
      dotenv auth login --api-key $DOTENV_API_KEY
      dotenv pull production
      source .env
```

## Configuration

### Store API Key

1. Go to your GitHub repository settings
2. Navigate to Secrets and variables → Actions
3. Add a new repository secret named `DOTENV_API_KEY`
4. Paste your DotEnv API key

### Basic Workflow

```yaml
name: Deploy Application
on:
    push:
        branches: [main]

jobs:
    deploy:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v3

            - name: Load DotEnv secrets
              uses: dotenv/actions/load@v1
              with:
                  api-key: ${{ secrets.DOTENV_API_KEY }}
                  project: my-app
                  environment: production

            - name: Build application
              run: |
                  npm install
                  npm run build
              env:
                  API_URL: ${{ env.API_URL }}
                  DATABASE_URL: ${{ env.DATABASE_URL }}

            - name: Deploy
              run: npm run deploy
```

## Usage Patterns

### 1. Environment-Based Deployment

Deploy to different environments based on branch:

```yaml
name: Deploy by Environment
on:
    push:
        branches: [main, staging, develop]

jobs:
    deploy:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v3

            - name: Determine environment
              id: env
              run: |
                  if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
                    echo "environment=production" >> $GITHUB_OUTPUT
                  elif [[ "${{ github.ref }}" == "refs/heads/staging" ]]; then
                    echo "environment=staging" >> $GITHUB_OUTPUT
                  else
                    echo "environment=development" >> $GITHUB_OUTPUT
                  fi

            - name: Load DotEnv secrets
              uses: dotenv/actions/load@v1
              with:
                  api-key: ${{ secrets.DOTENV_API_KEY }}
                  project: my-app
                  environment: ${{ steps.env.outputs.environment }}

            - name: Deploy to ${{ steps.env.outputs.environment }}
              run: |
                  echo "Deploying to ${{ steps.env.outputs.environment }}"
                  npm run deploy:${{ steps.env.outputs.environment }}
```

### 2. Pull Request Validation

Validate secrets before merging:

```yaml
name: PR Validation
on:
    pull_request:
        types: [opened, synchronize]

jobs:
    validate:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v3

            - name: Validate DotEnv configuration
              uses: dotenv/actions/validate@v1
              with:
                  api-key: ${{ secrets.DOTENV_API_KEY }}
                  project: my-app
                  environment: staging

            - name: Test with secrets
              uses: dotenv/actions/load@v1
              with:
                  api-key: ${{ secrets.DOTENV_API_KEY }}
                  project: my-app
                  environment: staging

            - name: Run tests
              run: |
                  npm install
                  npm test
              env:
                  NODE_ENV: test
```

### 3. Multi-Project Workflow

Handle multiple projects in a monorepo:

```yaml
name: Deploy Monorepo
on:
    push:
        branches: [main]

jobs:
    deploy-api:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v3

            - name: Load API secrets
              uses: dotenv/actions/load@v1
              with:
                  api-key: ${{ secrets.DOTENV_API_KEY }}
                  project: my-app-api
                  environment: production
                  prefix: API_

            - name: Deploy API
              run: |
                  cd packages/api
                  npm run deploy

    deploy-web:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v3

            - name: Load Web secrets
              uses: dotenv/actions/load@v1
              with:
                  api-key: ${{ secrets.DOTENV_API_KEY }}
                  project: my-app-web
                  environment: production
                  prefix: WEB_

            - name: Deploy Web
              run: |
                  cd packages/web
                  npm run deploy
```

### 4. Secret Rotation Workflow

Automate secret rotation:

```yaml
name: Rotate Secrets
on:
    schedule:
        - cron: "0 0 1 * *" # Monthly
    workflow_dispatch:

jobs:
    rotate:
        runs-on: ubuntu-latest
        steps:
            - name: Rotate API keys
              uses: dotenv/actions/rotate@v1
              with:
                  api-key: ${{ secrets.DOTENV_API_KEY }}
                  project: my-app
                  environment: production
                  secrets:
                      - STRIPE_API_KEY
                      - SENDGRID_API_KEY
                      - EXTERNAL_API_KEY

            - name: Notify team
              uses: actions/github-script@v6
              with:
                  script: |
                      github.rest.issues.create({
                        owner: context.repo.owner,
                        repo: context.repo.repo,
                        title: 'Secrets Rotated',
                        body: 'Monthly secret rotation completed successfully.'
                      })
```

## Advanced Features

### 1. Conditional Secret Loading

Load secrets based on conditions:

```yaml
- name: Load secrets conditionally
  uses: dotenv/actions/load@v1
  with:
      api-key: ${{ secrets.DOTENV_API_KEY }}
      project: my-app
      environment: ${{ github.event_name == 'release' && 'production' || 'staging' }}
      only-if-exists: true
```

### 2. Export Formats

Export secrets in different formats:

```yaml
- name: Export as JSON
  uses: dotenv/actions/export@v1
  with:
      api-key: ${{ secrets.DOTENV_API_KEY }}
      project: my-app
      environment: production
      format: json
      output-file: secrets.json

- name: Export for Docker
  uses: dotenv/actions/export@v1
  with:
      api-key: ${{ secrets.DOTENV_API_KEY }}
      project: my-app
      environment: production
      format: docker
      output-file: .env.docker
```

### 3. Matrix Builds

Test across multiple environments:

```yaml
name: Matrix Testing
on: [push]

jobs:
    test:
        runs-on: ubuntu-latest
        strategy:
            matrix:
                environment: [development, staging, production]
                node: [16, 18, 20]
        steps:
            - uses: actions/checkout@v3

            - name: Use Node.js ${{ matrix.node }}
              uses: actions/setup-node@v3
              with:
                  node-version: ${{ matrix.node }}

            - name: Load secrets for ${{ matrix.environment }}
              uses: dotenv/actions/load@v1
              with:
                  api-key: ${{ secrets.DOTENV_API_KEY }}
                  project: my-app
                  environment: ${{ matrix.environment }}

            - name: Run tests
              run: |
                  npm install
                  npm test
```

### 4. Masked Output

Automatically mask sensitive values:

```yaml
- name: Load and mask secrets
  uses: dotenv/actions/load@v1
  with:
      api-key: ${{ secrets.DOTENV_API_KEY }}
      project: my-app
      environment: production
      mask-values: true

- name: Debug (secrets are masked)
  run: |
      echo "API URL: $API_URL"
      echo "Database: $DATABASE_URL"
```

## Security Best Practices

### 1. Minimal Permissions

Create API keys with minimal required permissions:

```yaml
# .github/dotenv-permissions.yml
permissions:
    - resource: secrets
      actions: [read]
      environments: [production, staging]
    - resource: projects
      actions: [read]
```

### 2. Environment Protection

Use GitHub environment protection rules:

```yaml
jobs:
    deploy:
        runs-on: ubuntu-latest
        environment:
            name: production
            url: https://app.example.com
        steps:
            - name: Load production secrets
              uses: dotenv/actions/load@v1
              with:
                  api-key: ${{ secrets.DOTENV_API_KEY_PROD }}
                  project: my-app
                  environment: production
```

### 3. Audit Logging

Enable audit logging for compliance:

```yaml
- name: Load secrets with audit
  uses: dotenv/actions/load@v1
  with:
      api-key: ${{ secrets.DOTENV_API_KEY }}
      project: my-app
      environment: production
      audit-log: true
      audit-metadata:
          workflow: ${{ github.workflow }}
          run-id: ${{ github.run_id }}
          actor: ${{ github.actor }}
```

### 4. Secret Scanning

Prevent accidental secret exposure:

```yaml
- name: Scan for secrets
  uses: dotenv/actions/scan@v1
  with:
      scan-pull-request: true
      fail-on-secrets: true
      exclude-patterns:
          - "*.test.js"
          - "docs/**"
```

## Troubleshooting

### Common Issues

#### Authentication Failed

```yaml
- name: Debug authentication
  env:
      DOTENV_DEBUG: true
  run: |
      dotenv auth status
      dotenv project list
```

#### Environment Not Found

```yaml
- name: List available environments
  run: |
      dotenv env list --project my-app
```

#### Network Issues

```yaml
- name: Test connectivity
  run: |
      curl -I https://api.dotenv.cloud/v1/health
      dotenv status
```

### Debug Mode

Enable detailed logging:

```yaml
- name: Load secrets (debug mode)
  uses: dotenv/actions/load@v1
  with:
      api-key: ${{ secrets.DOTENV_API_KEY }}
      project: my-app
      environment: production
      debug: true
```

## Examples

### Complete CI/CD Pipeline

```yaml
name: Complete CI/CD
on:
    push:
        branches: [main]
    pull_request:
        branches: [main]

env:
    NODE_VERSION: 18

jobs:
    test:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v3

            - name: Setup Node.js
              uses: actions/setup-node@v3
              with:
                  node-version: ${{ env.NODE_VERSION }}
                  cache: npm

            - name: Load test secrets
              uses: dotenv/actions/load@v1
              with:
                  api-key: ${{ secrets.DOTENV_API_KEY }}
                  project: my-app
                  environment: test

            - name: Install dependencies
              run: npm ci

            - name: Run tests
              run: npm test

            - name: Upload coverage
              uses: codecov/codecov-action@v3

    build:
        needs: test
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v3

            - name: Load build secrets
              uses: dotenv/actions/load@v1
              with:
                  api-key: ${{ secrets.DOTENV_API_KEY }}
                  project: my-app
                  environment: staging

            - name: Build application
              run: |
                  npm ci
                  npm run build

            - name: Upload artifacts
              uses: actions/upload-artifact@v3
              with:
                  name: build-artifacts
                  path: dist/

    deploy:
        needs: build
        runs-on: ubuntu-latest
        if: github.ref == 'refs/heads/main'
        environment:
            name: production
            url: https://app.example.com
        steps:
            - uses: actions/checkout@v3

            - name: Download artifacts
              uses: actions/download-artifact@v3
              with:
                  name: build-artifacts
                  path: dist/

            - name: Load production secrets
              uses: dotenv/actions/load@v1
              with:
                  api-key: ${{ secrets.DOTENV_API_KEY_PROD }}
                  project: my-app
                  environment: production

            - name: Deploy to production
              run: |
                  npm run deploy:prod

            - name: Notify deployment
              uses: actions/github-script@v6
              with:
                  script: |
                      github.rest.repos.createDeploymentStatus({
                        owner: context.repo.owner,
                        repo: context.repo.repo,
                        deployment_id: context.payload.deployment.id,
                        state: 'success',
                        environment_url: 'https://app.example.com'
                      })
```

## Migration Guide

### From GitHub Secrets

Migrate from GitHub Secrets to DotEnv:

```yaml
# Before
- name: Deploy
  env:
      API_KEY: ${{ secrets.API_KEY }}
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
  run: npm run deploy

# After
- name: Load DotEnv secrets
  uses: dotenv/actions/load@v1
  with:
      api-key: ${{ secrets.DOTENV_API_KEY }}
      project: my-app
      environment: production

- name: Deploy
  run: npm run deploy
```

## Resources

- [DotEnv CLI Documentation](/documentation/v1/cli/overview)
- [API Reference](/documentation/v1/api/overview)
- [GitHub Actions Documentation](https://docs.github.com/actions)
- [Security Best Practices](/documentation/v1/security/best-practices)

## Next Steps

- [Set up your first workflow](#basic-workflow)
- [Configure environment protection](#environment-protection)
- [Implement secret rotation](#secret-rotation-workflow)
- [Enable audit logging](#audit-logging)
