---
title: CI/CD Integration
slug: ci-cd-usage
order: 9
tags: [cli, ci-cd, automation, deployment]
---

# CI/CD Integration

This guide covers how to integrate the DotEnv CLI with popular CI/CD platforms for automated secret management in your deployment pipelines.

## General Principles

### Security Best Practices

1. **Use Service Accounts**: Create dedicated service accounts for CI/CD
2. **Minimal Permissions**: Grant only necessary permissions
3. **Rotate Keys Regularly**: Automate key rotation
4. **Audit Access**: Monitor CI/CD secret access
5. **Never Log Secrets**: Ensure secrets aren't exposed in logs

### Common Patterns

```bash
# Basic CI/CD pattern
#!/bin/bash
set -euo pipefail

# Authenticate
export DOTENV_API_KEY="${CI_DOTENV_API_KEY}"

# Pull secrets for environment
dotenv pull \
  --project "${PROJECT_NAME}" \
  --environment "${DEPLOY_ENV}" \
  --output .env

# Use secrets
source .env
./deploy.sh

# Cleanup
rm -f .env
```

## GitHub Actions

### Basic Setup

```yaml
name: Deploy
on:
    push:
        branches: [main, develop]

jobs:
    deploy:
        runs-on: ubuntu-latest

        steps:
            - uses: actions/checkout@v3

            - name: Install DotEnv CLI
              run: |
                  curl -sSL https://dotenv.cloud/install.sh | sh
                  echo "$HOME/.dotenv/bin" >> $GITHUB_PATH

            - name: Pull Secrets
              env:
                  DOTENV_API_KEY: ${{ secrets.DOTENV_API_KEY }}
              run: |
                  dotenv pull \
                    --project my-app \
                    --environment ${{ github.ref_name == 'main' && 'production' || 'staging' }}

            - name: Deploy
              run: |
                  source .env
                  npm run deploy
```

### Using DotEnv Action

```yaml
name: Deploy with DotEnv Action
on: push

jobs:
    deploy:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v3

            - name: Load DotEnv Secrets
              uses: dotenv/github-action@v1
              with:
                  api-key: ${{ secrets.DOTENV_API_KEY }}
                  project: my-app
                  environment: ${{ github.ref_name }}

            - name: Build and Deploy
              run: |
                  # Secrets are now in environment
                  npm run build
                  npm run deploy
```

### Advanced GitHub Actions

```yaml
name: Advanced Deployment

on:
    push:
        branches: [main, staging, develop]
    pull_request:
        types: [opened, synchronize]

env:
    DOTENV_NON_INTERACTIVE: true
    DOTENV_NO_COLOR: true

jobs:
    setup:
        runs-on: ubuntu-latest
        outputs:
            environment: ${{ steps.env.outputs.environment }}
            project: ${{ steps.env.outputs.project }}

        steps:
            - name: Determine Environment
              id: env
              run: |
                  if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
                    echo "environment=production" >> $GITHUB_OUTPUT
                  elif [[ "${{ github.ref }}" == "refs/heads/staging" ]]; then
                    echo "environment=staging" >> $GITHUB_OUTPUT
                  else
                    echo "environment=development" >> $GITHUB_OUTPUT
                  fi
                  echo "project=my-app" >> $GITHUB_OUTPUT

    test:
        needs: setup
        runs-on: ubuntu-latest

        steps:
            - uses: actions/checkout@v3

            - name: Setup DotEnv
              run: |
                  curl -sSL https://dotenv.cloud/install.sh | sh
                  echo "$HOME/.dotenv/bin" >> $GITHUB_PATH

            - name: Load Test Secrets
              env:
                  DOTENV_API_KEY: ${{ secrets.DOTENV_API_KEY }}
              run: |
                  dotenv pull \
                    --project ${{ needs.setup.outputs.project }} \
                    --environment test \
                    --output .env.test

            - name: Run Tests
              run: |
                  source .env.test
                  npm test

            - name: Cleanup
              if: always()
              run: rm -f .env.test

    deploy:
        needs: [setup, test]
        runs-on: ubuntu-latest
        if: github.event_name == 'push'

        environment:
            name: ${{ needs.setup.outputs.environment }}
            url: ${{ steps.deploy.outputs.url }}

        steps:
            - uses: actions/checkout@v3

            - name: Setup DotEnv CLI
              uses: dotenv/setup-cli@v1
              with:
                  version: latest

            - name: Pull Deployment Secrets
              env:
                  DOTENV_API_KEY: ${{ secrets.DOTENV_API_KEY }}
              run: |
                  dotenv pull \
                    --project ${{ needs.setup.outputs.project }} \
                    --environment ${{ needs.setup.outputs.environment }} \
                    --format json > secrets.json

            - name: Deploy
              id: deploy
              run: |
                  # Convert secrets to environment variables
                  export $(jq -r 'to_entries|map("\(.key)=\(.value|@sh)")|.[]' secrets.json)

                  # Deploy application
                  ./scripts/deploy.sh

                  # Set output
                  echo "url=https://${APP_DOMAIN}" >> $GITHUB_OUTPUT

            - name: Verify Deployment
              run: |
                  curl -f https://${APP_DOMAIN}/health || exit 1

            - name: Cleanup
              if: always()
              run: rm -f secrets.json
```

## GitLab CI/CD

### Basic Setup

```yaml
# .gitlab-ci.yml
variables:
    DOTENV_PROJECT: "my-app"
    DOTENV_NON_INTERACTIVE: "true"
    DOTENV_NO_COLOR: "true"

stages:
    - build
    - test
    - deploy

.dotenv_setup:
    before_script:
        - curl -sSL https://dotenv.cloud/install.sh | sh
        - export PATH="$HOME/.dotenv/bin:$PATH"
        - export DOTENV_API_KEY="${CI_DOTENV_API_KEY}"

build:
    extends: .dotenv_setup
    stage: build
    script:
        - dotenv pull --environment development
        - source .env
        - npm run build
    artifacts:
        paths:
            - dist/

test:
    extends: .dotenv_setup
    stage: test
    script:
        - dotenv pull --environment test
        - source .env
        - npm test

deploy:production:
    extends: .dotenv_setup
    stage: deploy
    only:
        - main
    script:
        - dotenv pull --environment production
        - source .env
        - ./deploy.sh production
    environment:
        name: production
        url: https://app.example.com
```

### Advanced GitLab Configuration

```yaml
# .gitlab-ci.yml
include:
    - template: Security/Secret-Detection.gitlab-ci.yml

variables:
    DOTENV_CACHE_DIR: "${CI_PROJECT_DIR}/.dotenv-cache"

.dotenv_template:
    image: alpine:latest
    before_script:
        # Install dependencies
        - apk add --no-cache curl jq bash

        # Install DotEnv CLI
        - |
            if [ ! -f "$DOTENV_CACHE_DIR/bin/dotenv" ]; then
              mkdir -p "$DOTENV_CACHE_DIR"
              curl -sSL https://dotenv.cloud/install.sh | sh -s -- --prefix="$DOTENV_CACHE_DIR"
            fi
        - export PATH="$DOTENV_CACHE_DIR/bin:$PATH"
        # Configure CLI
        - export DOTENV_API_KEY="${CI_DOTENV_API_KEY}"
        - export DOTENV_PROJECT="${CI_PROJECT_NAME}"

    cache:
        key: dotenv-cli
        paths:
            - .dotenv-cache/

# Dynamic environment detection
.determine_environment:
    script:
        - |
            case "$CI_COMMIT_REF_NAME" in
              main|master) export DEPLOY_ENV="production" ;;
              staging) export DEPLOY_ENV="staging" ;;
              develop) export DEPLOY_ENV="development" ;;
              *) export DEPLOY_ENV="review/$CI_COMMIT_REF_SLUG" ;;
            esac
            echo "DEPLOY_ENV=$DEPLOY_ENV" >> deploy.env
    artifacts:
        reports:
            dotenv: deploy.env

# Multi-project deployment
deploy:microservices:
    extends: .dotenv_template
    stage: deploy
    parallel:
        matrix:
            - SERVICE: [api, web, worker]
    script:
        - source deploy.env
        - |
            dotenv pull \
              --project "${CI_PROJECT_NAME}-${SERVICE}" \
              --environment "$DEPLOY_ENV" \
              --output ".env.${SERVICE}"
        - |
            docker build \
              --secret id=env,src=.env.${SERVICE} \
              -t "${CI_REGISTRY_IMAGE}/${SERVICE}:${CI_COMMIT_SHA}" \
              -f docker/${SERVICE}/Dockerfile .
        - docker push "${CI_REGISTRY_IMAGE}/${SERVICE}:${CI_COMMIT_SHA}"
```

## Jenkins

### Jenkinsfile Pipeline

```groovy
pipeline {
    agent any

    environment {
        DOTENV_API_KEY = credentials('dotenv-api-key')
        DOTENV_PROJECT = 'my-app'
        DOTENV_NON_INTERACTIVE = 'true'
    }

    stages {
        stage('Setup') {
            steps {
                sh '''
                    # Install DotEnv CLI
                    if ! command -v dotenv &> /dev/null; then
                        curl -sSL https://dotenv.cloud/install.sh | sh
                        echo 'export PATH="$HOME/.dotenv/bin:$PATH"' >> ~/.bashrc
                    fi
                '''
            }
        }

        stage('Pull Secrets') {
            steps {
                script {
                    def environment = env.BRANCH_NAME == 'main' ? 'production' : 'staging'

                    sh """
                        export PATH="\$HOME/.dotenv/bin:\$PATH"
                        dotenv pull \\
                            --environment ${environment} \\
                            --output .env
                    """

                    // Load secrets into Jenkins environment
                    def secrets = readProperties file: '.env'
                    secrets.each { key, value ->
                        env["${key}"] = value
                    }
                }
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test'
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh './scripts/deploy.sh'
            }
        }
    }

    post {
        always {
            sh 'rm -f .env'
        }
    }
}
```

### Jenkins Shared Library

```groovy
// vars/dotenvSecrets.groovy
def call(Map config = [:]) {
    def project = config.project ?: env.JOB_NAME
    def environment = config.environment ?: 'development'
    def format = config.format ?: 'properties'

    sh """
        dotenv pull \\
            --project '${project}' \\
            --environment '${environment}' \\
            --format '${format}' \\
            --output secrets.${format}
    """

    if (format == 'properties') {
        def props = readProperties file: "secrets.${format}"
        props.each { k, v -> env[k] = v }
    } else if (format == 'json') {
        def json = readJSON file: "secrets.${format}"
        json.each { k, v -> env[k] = v }
    }

    sh "rm -f secrets.${format}"
}

// Usage in Jenkinsfile
@Library('shared-library') _

pipeline {
    agent any
    stages {
        stage('Load Secrets') {
            steps {
                dotenvSecrets(
                    project: 'my-app',
                    environment: env.BRANCH_NAME == 'main' ? 'production' : 'staging'
                )
            }
        }
    }
}
```

## CircleCI

### Basic Configuration

```yaml
# .circleci/config.yml
version: 2.1

executors:
    default:
        docker:
            - image: cimg/node:18.0

commands:
    setup-dotenv:
        steps:
            - run:
                  name: Install DotEnv CLI
                  command: |
                      curl -sSL https://dotenv.cloud/install.sh | sh
                      echo 'export PATH="$HOME/.dotenv/bin:$PATH"' >> $BASH_ENV
                      source $BASH_ENV

            - run:
                  name: Configure DotEnv
                  command: |
                      echo "export DOTENV_API_KEY='${DOTENV_API_KEY}'" >> $BASH_ENV
                      echo "export DOTENV_PROJECT='my-app'" >> $BASH_ENV
                      echo "export DOTENV_NON_INTERACTIVE='true'" >> $BASH_ENV
                      source $BASH_ENV

jobs:
    test:
        executor: default
        steps:
            - checkout
            - setup-dotenv

            - run:
                  name: Load Test Secrets
                  command: |
                      dotenv pull --environment test
                      source .env

            - run:
                  name: Run Tests
                  command: npm test

            - run:
                  name: Cleanup
                  when: always
                  command: rm -f .env

    deploy:
        executor: default
        parameters:
            environment:
                type: string
                default: staging

        steps:
            - checkout
            - setup-dotenv

            - run:
                  name: Pull Deployment Secrets
                  command: |
                      dotenv pull --environment << parameters.environment >>

            - run:
                  name: Deploy
                  command: |
                      source .env
                      ./scripts/deploy.sh << parameters.environment >>

workflows:
    version: 2
    build-deploy:
        jobs:
            - test:
                  context: dotenv-credentials

            - deploy:
                  name: deploy-staging
                  environment: staging
                  requires:
                      - test
                  filters:
                      branches:
                          only: staging

            - deploy:
                  name: deploy-production
                  environment: production
                  requires:
                      - test
                  filters:
                      branches:
                          only: main
```

### CircleCI Orbs

```yaml
# .circleci/config.yml
version: 2.1

orbs:
    dotenv: dotenv/cli@1.0.0

workflows:
    deploy:
        jobs:
            - dotenv/load-secrets:
                  project: my-app
                  environment: production
                  context: dotenv-credentials

            - build:
                  requires:
                      - dotenv/load-secrets

            - deploy:
                  requires:
                      - build
```

## Travis CI

### .travis.yml Configuration

```yaml
language: node_js
node_js:
    - "18"

env:
    global:
        - DOTENV_PROJECT=my-app
        - DOTENV_NON_INTERACTIVE=true

before_install:
    # Install DotEnv CLI
    - curl -sSL https://dotenv.cloud/install.sh | sh
    - export PATH="$HOME/.dotenv/bin:$PATH"

    # Determine environment
    - |
        if [ "$TRAVIS_BRANCH" = "main" ] && [ "$TRAVIS_PULL_REQUEST" = "false" ]; then
          export DEPLOY_ENV="production"
        elif [ "$TRAVIS_BRANCH" = "staging" ] && [ "$TRAVIS_PULL_REQUEST" = "false" ]; then
          export DEPLOY_ENV="staging"
        else
          export DEPLOY_ENV="development"
        fi

install:
    - dotenv pull --environment "$DEPLOY_ENV"
    - source .env
    - npm install

script:
    - npm test
    - npm run build

deploy:
    - provider: script
      script: ./scripts/deploy.sh production
      on:
          branch: main

    - provider: script
      script: ./scripts/deploy.sh staging
      on:
          branch: staging

after_script:
    - rm -f .env
```

## Azure DevOps

### azure-pipelines.yml

```yaml
trigger:
    branches:
        include:
            - main
            - staging
            - develop

pool:
    vmImage: "ubuntu-latest"

variables:
    DOTENV_PROJECT: "my-app"
    DOTENV_NON_INTERACTIVE: "true"

stages:
    - stage: Setup
      jobs:
          - job: DetermineEnvironment
            steps:
                - bash: |
                      case "$(Build.SourceBranchName)" in
                        main) ENV="production" ;;
                        staging) ENV="staging" ;;
                        *) ENV="development" ;;
                      esac
                      echo "##vso[task.setvariable variable=DeployEnvironment;isOutput=true]$ENV"
                  name: SetEnv

    - stage: Build
      dependsOn: Setup
      variables:
          DeployEnv: $[ stageDependencies.Setup.DetermineEnvironment.outputs['SetEnv.DeployEnvironment'] ]

      jobs:
          - job: BuildApp
            steps:
                - task: Bash@3
                  displayName: "Install DotEnv CLI"
                  inputs:
                      targetType: "inline"
                      script: |
                          curl -sSL https://dotenv.cloud/install.sh | sh
                          echo "##vso[task.prependpath]$HOME/.dotenv/bin"

                - task: Bash@3
                  displayName: "Pull Secrets"
                  env:
                      DOTENV_API_KEY: $(DotEnvApiKey)
                  inputs:
                      targetType: "inline"
                      script: |
                          dotenv pull \
                            --environment "$(DeployEnv)" \
                            --output pipeline.env

                          # Load secrets into pipeline
                          while IFS='=' read -r key value; do
                            echo "##vso[task.setvariable variable=$key]$value"
                          done < pipeline.env

                          rm -f pipeline.env

                - task: Npm@1
                  displayName: "Build Application"
                  inputs:
                      command: "custom"
                      customCommand: "run build"

    - stage: Deploy
      dependsOn: Build
      condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))

      jobs:
          - deployment: DeployProduction
            environment: "production"
            strategy:
                runOnce:
                    deploy:
                        steps:
                            - bash: |
                                  dotenv pull --environment production
                                  source .env
                                  ./scripts/deploy.sh
```

## Bitbucket Pipelines

### bitbucket-pipelines.yml

```yaml
image: node:18

definitions:
    caches:
        dotenv: ~/.dotenv

    steps:
        - step: &install-dotenv
              name: Install DotEnv CLI
              caches:
                  - dotenv
              script:
                  - |
                      if [ ! -f "$HOME/.dotenv/bin/dotenv" ]; then
                        curl -sSL https://dotenv.cloud/install.sh | sh
                      fi
                  - export PATH="$HOME/.dotenv/bin:$PATH"

        - step: &load-secrets
              name: Load Secrets
              script:
                  - export PATH="$HOME/.dotenv/bin:$PATH"
                  - export DOTENV_API_KEY="$DOTENV_API_KEY"
                  - |
                      case "$BITBUCKET_BRANCH" in
                        main) ENV="production" ;;
                        staging) ENV="staging" ;;
                        *) ENV="development" ;;
                      esac
                  - dotenv pull --project my-app --environment "$ENV"
                  - source .env

pipelines:
    default:
        - step: *install-dotenv
        - step: *load-secrets
        - step:
              name: Test
              script:
                  - source .env
                  - npm test

    branches:
        main:
            - step: *install-dotenv
            - step: *load-secrets
            - step:
                  name: Deploy Production
                  deployment: production
                  script:
                      - source .env
                      - ./scripts/deploy.sh production

        staging:
            - step: *install-dotenv
            - step: *load-secrets
            - step:
                  name: Deploy Staging
                  deployment: staging
                  script:
                      - source .env
                      - ./scripts/deploy.sh staging
```

## Docker Integration

### Multi-stage Build

```dockerfile
# Dockerfile
FROM alpine:latest AS dotenv
ARG DOTENV_API_KEY
ARG DOTENV_PROJECT
ARG DOTENV_ENVIRONMENT=production

# Install CLI
RUN apk add --no-cache curl bash
RUN curl -sSL https://dotenv.cloud/install.sh | sh

# Pull secrets
ENV PATH="/root/.dotenv/bin:$PATH"
RUN dotenv pull \
    --project "$DOTENV_PROJECT" \
    --environment "$DOTENV_ENVIRONMENT" \
    --output /tmp/.env

# Build stage
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
COPY --from=dotenv /tmp/.env .env
RUN source .env && npm run build

# Runtime stage
FROM node:18-slim
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=dotenv /tmp/.env .env
CMD ["sh", "-c", "source .env && node dist/server.js"]
```

### Docker Compose

```yaml
# docker-compose.yml
version: "3.8"

services:
    dotenv-loader:
        image: alpine
        volumes:
            - secrets:/secrets
        environment:
            - DOTENV_API_KEY
            - DOTENV_PROJECT
            - DOTENV_ENVIRONMENT
        command: |
            sh -c '
              apk add --no-cache curl bash
              curl -sSL https://dotenv.cloud/install.sh | sh
              export PATH="/root/.dotenv/bin:$PATH"
              dotenv pull --output /secrets/.env
            '

    app:
        build: .
        depends_on:
            dotenv-loader:
                condition: service_completed_successfully
        volumes:
            - secrets:/secrets:ro
        command: |
            sh -c '
              source /secrets/.env
              node server.js
            '

volumes:
    secrets:
        driver: local
        driver_opts:
            type: tmpfs
            device: tmpfs
```

## Kubernetes Integration

### Init Container Pattern

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: my-app
spec:
    template:
        spec:
            initContainers:
                - name: dotenv-secrets
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
                          apk add --no-cache curl
                          curl -sSL https://dotenv.cloud/install.sh | sh
                          export PATH="/root/.dotenv/bin:$PATH"
                          dotenv pull \
                            --project my-app \
                            --environment production \
                            --format json > /shared/secrets.json
                  volumeMounts:
                      - name: shared-data
                        mountPath: /shared

            containers:
                - name: app
                  image: my-app:latest
                  command:
                      - sh
                      - -c
                      - |
                          export $(cat /shared/secrets.json | jq -r 'to_entries|map("\(.key)=\(.value)")|.[]')
                          node server.js
                  volumeMounts:
                      - name: shared-data
                        mountPath: /shared
                        readOnly: true

            volumes:
                - name: shared-data
                  emptyDir:
                      medium: Memory
```

## Security Considerations

### Service Account Setup

```bash
#!/bin/bash
# create-ci-service-account.sh

# Create service account
dotenv service-accounts create \
  --name "ci-deployer" \
  --description "CI/CD deployment service" \
  --expires 365d \
  --permissions "secrets:read,secrets:list" \
  --projects "my-app,my-api" \
  --environments "staging,production"

# Output for CI configuration
echo "Save this API key in your CI/CD secrets:"
echo "DOTENV_API_KEY=<generated-key>"
```

### Audit CI/CD Access

```bash
# Monitor CI/CD usage
dotenv audit query \
  --service-account "ci-deployer" \
  --days 30 \
  --format json | jq '.[] | {
    timestamp: .timestamp,
    action: .action,
    ip: .ip_address,
    user_agent: .user_agent
  }'
```

### Rotate CI/CD Keys

```yaml
# .github/workflows/rotate-keys.yml
name: Rotate CI/CD Keys
on:
    schedule:
        - cron: "0 0 1 * *" # Monthly

jobs:
    rotate:
        runs-on: ubuntu-latest
        steps:
            - name: Rotate DotEnv API Key
              run: |
                  # This would typically be done manually
                  # or through your security automation
                  echo "Time to rotate CI/CD keys"
```

## Best Practices

1. **Use Least Privilege**: Grant minimal permissions needed
2. **Separate Environments**: Use different keys per environment
3. **Clean Up Secrets**: Always remove secrets after use
4. **Enable Audit Logs**: Track all CI/CD secret access
5. **Use Ephemeral Storage**: Store secrets in memory when possible
6. **Implement Rotation**: Regularly rotate CI/CD credentials
7. **Monitor Usage**: Set up alerts for unusual access patterns

## Next Steps

- [Security Guide](/documentation/v1/security/ci-cd-security) - CI/CD security best practices
- [API Reference](/documentation/v1/api/authentication) - Direct API usage
- [Troubleshooting](./troubleshooting) - Common CI/CD issues
- [GitHub Action](/documentation/v1/integrations/github) - Official GitHub Action
