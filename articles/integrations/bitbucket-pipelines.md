---
title: Bitbucket Pipelines Integration
slug: bitbucket-pipelines
order: 14
tags: [integrations, bitbucket, pipelines, ci-cd]
---

# Bitbucket Pipelines Integration

Integrate DotEnv with Bitbucket Pipelines to manage secrets in your CI/CD workflows. This guide covers setup, configuration, and best practices for Bitbucket Pipelines.

## Overview

The DotEnv Bitbucket Pipelines integration provides:

- Secure secret injection into pipelines
- Repository and workspace variables
- Environment-based deployments
- Parallel step support
- Docker service integration
- Deployment tracking

## Setup

### 1. Repository Variables

Add DotEnv API key to repository settings:

1. Navigate to Repository settings → Repository variables
2. Add a new variable:
    - Name: `DOTENV_API_KEY`
    - Value: Your DotEnv API key
    - Secured: Yes (checkbox)

### 2. Pipeline Configuration

Create `bitbucket-pipelines.yml`:

```yaml
image: node:18

definitions:
    caches:
        dotenv: ~/.dotenv

    scripts:
        - script: &install-dotenv
              name: Install DotEnv CLI
              script:
                  - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                  - export PATH="$HOME/.dotenv/bin:$PATH"
                  - dotenv --version

        - script: &load-secrets
              name: Load secrets from DotEnv
              script:
                  - export PATH="$HOME/.dotenv/bin:$PATH"
                  - dotenv auth login --api-key $DOTENV_API_KEY
                  - dotenv pull $DOTENV_ENVIRONMENT --project $BITBUCKET_REPO_SLUG
                  - source .env

pipelines:
    default:
        - step:
              name: Build and Test
              caches:
                  - node
                  - dotenv
              script:
                  - *install-dotenv
                  - export DOTENV_ENVIRONMENT=development
                  - *load-secrets
                  - npm install
                  - npm test
              after-script:
                  - rm -f .env
```

## Basic Configuration

### Branch-Based Deployments

```yaml
image: node:18

pipelines:
    branches:
        main:
            - step:
                  name: Deploy to Production
                  deployment: production
                  caches:
                      - dotenv
                  script:
                      - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                      - export PATH="$HOME/.dotenv/bin:$PATH"
                      - dotenv auth login --api-key $DOTENV_API_KEY
                      - dotenv pull production --project $BITBUCKET_REPO_SLUG
                      - source .env
                      - npm install
                      - npm run build
                      - npm run deploy:production
                  after-script:
                      - rm -f .env

        staging:
            - step:
                  name: Deploy to Staging
                  deployment: staging
                  script:
                      - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                      - export PATH="$HOME/.dotenv/bin:$PATH"
                      - dotenv auth login --api-key $DOTENV_API_KEY
                      - dotenv pull staging --project $BITBUCKET_REPO_SLUG
                      - source .env
                      - npm install
                      - npm run build
                      - npm run deploy:staging
                  after-script:
                      - rm -f .env

        develop:
            - step:
                  name: Deploy to Development
                  deployment: development
                  script:
                      - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                      - export PATH="$HOME/.dotenv/bin:$PATH"
                      - dotenv auth login --api-key $DOTENV_API_KEY
                      - dotenv pull development --project $BITBUCKET_REPO_SLUG
                      - source .env
                      - npm install
                      - npm run build
                      - npm run deploy:development
                  after-script:
                      - rm -f .env
```

### Pull Request Pipelines

```yaml
pipelines:
    pull-requests:
        "**":
            - step:
                  name: PR Validation
                  script:
                      - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                      - export PATH="$HOME/.dotenv/bin:$PATH"
                      - dotenv auth login --api-key $DOTENV_API_KEY
                      - |
                          # Create PR environment
                          PR_ENV="pr-$BITBUCKET_PR_ID"
                          dotenv env create $PR_ENV --from development --project $BITBUCKET_REPO_SLUG || true
                          dotenv pull $PR_ENV --project $BITBUCKET_REPO_SLUG
                      - source .env
                      - npm install
                      - npm run lint
                      - npm run test
                      - npm run build
                  after-script:
                      - rm -f .env

            - step:
                  name: Deploy Preview
                  deployment: test
                  trigger: manual
                  script:
                      - export PATH="$HOME/.dotenv/bin:$PATH"
                      - dotenv pull pr-$BITBUCKET_PR_ID --project $BITBUCKET_REPO_SLUG
                      - source .env
                      - npm run deploy:preview
                      - echo "Preview deployed to https://pr-$BITBUCKET_PR_ID.preview.example.com"
```

## Advanced Features

### 1. Parallel Steps

```yaml
pipelines:
    default:
        - parallel:
              - step:
                    name: Unit Tests
                    caches:
                        - node
                        - dotenv
                    script:
                        - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                        - export PATH="$HOME/.dotenv/bin:$PATH"
                        - dotenv auth login --api-key $DOTENV_API_KEY
                        - dotenv pull test --project $BITBUCKET_REPO_SLUG
                        - source .env
                        - npm install
                        - npm run test:unit

              - step:
                    name: Integration Tests
                    caches:
                        - node
                        - dotenv
                    services:
                        - postgres
                        - redis
                    script:
                        - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                        - export PATH="$HOME/.dotenv/bin:$PATH"
                        - dotenv auth login --api-key $DOTENV_API_KEY
                        - dotenv pull test --project $BITBUCKET_REPO_SLUG
                        - source .env
                        - npm install
                        - npm run test:integration

              - step:
                    name: Lint
                    caches:
                        - node
                    script:
                        - npm install
                        - npm run lint

definitions:
    services:
        postgres:
            image: postgres:14
            variables:
                POSTGRES_DB: test
                POSTGRES_USER: test
                POSTGRES_PASSWORD: test
        redis:
            image: redis:7
```

### 2. Docker Integration

```yaml
image: atlassian/default-image:3

pipelines:
    branches:
        main:
            - step:
                  name: Build Docker Image
                  services:
                      - docker
                  caches:
                      - docker
                      - dotenv
                  script:
                      # Install DotEnv
                      - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                      - export PATH="$HOME/.dotenv/bin:$PATH"

                      # Load secrets
                      - dotenv auth login --api-key $DOTENV_API_KEY
                      - dotenv pull production --project $BITBUCKET_REPO_SLUG --format docker > .env.docker

                      # Build image with secrets
                      - |
                          export DOCKER_BUILDKIT=1
                          docker build \
                            --secret id=dotenv,src=.env.docker \
                            -t $DOCKER_REGISTRY/$BITBUCKET_REPO_SLUG:$BITBUCKET_COMMIT \
                            -t $DOCKER_REGISTRY/$BITBUCKET_REPO_SLUG:latest \
                            .

                      # Push to registry
                      - echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin $DOCKER_REGISTRY
                      - docker push $DOCKER_REGISTRY/$BITBUCKET_REPO_SLUG:$BITBUCKET_COMMIT
                      - docker push $DOCKER_REGISTRY/$BITBUCKET_REPO_SLUG:latest
                  after-script:
                      - rm -f .env.docker
```

### 3. Custom Pipelines

```yaml
pipelines:
    custom:
        deploy-production:
            - variables:
                  - name: DEPLOYMENT_REGION
                    default: us-east-1
                    allowed-values:
                        - us-east-1
                        - us-west-2
                        - eu-west-1
                        - ap-southeast-1
            - step:
                  name: Deploy to $DEPLOYMENT_REGION
                  deployment: production
                  script:
                      - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                      - export PATH="$HOME/.dotenv/bin:$PATH"
                      - dotenv auth login --api-key $DOTENV_API_KEY
                      - dotenv pull production --project $BITBUCKET_REPO_SLUG
                      - source .env
                      - |
                          # Region-specific deployment
                          export AWS_REGION=$DEPLOYMENT_REGION
                          npm run deploy:production

        rotate-secrets:
            - step:
                  name: Rotate Secrets
                  script:
                      - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                      - export PATH="$HOME/.dotenv/bin:$PATH"
                      - dotenv auth login --api-key $DOTENV_API_KEY
                      - |
                          # Rotate specific secrets
                          dotenv secret rotate \
                            --project $BITBUCKET_REPO_SLUG \
                            --environment production \
                            --key DATABASE_PASSWORD

                          dotenv secret rotate \
                            --project $BITBUCKET_REPO_SLUG \
                            --environment production \
                            --key API_KEY
```

### 4. Multi-Repository Setup

```yaml
# Monorepo configuration
pipelines:
    default:
        - step:
              name: Detect Changes
              script:
                  - |
                      # Detect which services changed
                      CHANGED_SERVICES=""
                      for service in api web worker; do
                        if git diff HEAD~1 --name-only | grep -q "^services/$service/"; then
                          CHANGED_SERVICES="$CHANGED_SERVICES $service"
                        fi
                      done
                      echo "Changed services: $CHANGED_SERVICES"
                      echo $CHANGED_SERVICES > changed-services.txt
              artifacts:
                  - changed-services.txt

        - step:
              name: Build Changed Services
              script:
                  - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                  - export PATH="$HOME/.dotenv/bin:$PATH"
                  - dotenv auth login --api-key $DOTENV_API_KEY
                  - |
                      CHANGED_SERVICES=$(cat changed-services.txt)
                      for service in $CHANGED_SERVICES; do
                        echo "Building $service..."
                        dotenv pull production --project "${BITBUCKET_REPO_SLUG}-${service}"
                        source .env
                        cd services/$service
                        npm install
                        npm run build
                        cd ../..
                        rm .env
                      done
```

## Security Best Practices

### 1. Workspace Variables

Use workspace-level variables for shared secrets:

```yaml
# Reference workspace variables
pipelines:
    default:
        - step:
              script:
                  # Workspace variable (inherited by all repos)
                  - echo "Using workspace API key: $WORKSPACE_DOTENV_API_KEY"

                  # Repository variable (overrides workspace)
                  - echo "Using repo API key: $DOTENV_API_KEY"
```

### 2. Secure Variable Handling

```yaml
definitions:
    scripts:
        - script: &secure-load-secrets
              name: Securely load secrets
              script:
                  - |
                      # Create temporary directory for secrets
                      SECRETS_DIR=$(mktemp -d)
                      cd $SECRETS_DIR

                      # Load secrets to temporary location
                      export PATH="$HOME/.dotenv/bin:$PATH"
                      dotenv auth login --api-key $DOTENV_API_KEY
                      dotenv pull $DOTENV_ENVIRONMENT --project $BITBUCKET_REPO_SLUG

                      # Export variables without logging
                      set +x
                      source .env
                      set -x

                      # Return to workspace
                      cd $BITBUCKET_CLONE_DIR

                      # Clean up
                      rm -rf $SECRETS_DIR
```

### 3. Deployment Permissions

```yaml
# Restrict production deployments
pipelines:
    branches:
        main:
            - step:
                  name: Production Deployment
                  deployment: production
                  # Only specific users can trigger
                  trigger: manual
                  script:
                      - |
                          # Verify deployment permission
                          if [[ "$BITBUCKET_STEP_TRIGGERER_UUID" != "{allowed-user-uuid}" ]]; then
                            echo "Unauthorized deployment attempt"
                            exit 1
                          fi
                      -  # ... deployment steps
```

### 4. Audit Logging

```yaml
definitions:
    scripts:
        - script: &audit-log
              name: Log secret access
              script:
                  - |
                      # Send audit log
                      curl -X POST https://audit.example.com/api/logs \
                        -H "Content-Type: application/json" \
                        -d '{
                          "event": "secret_access",
                          "repository": "'$BITBUCKET_REPO_FULL_NAME'",
                          "branch": "'$BITBUCKET_BRANCH'",
                          "commit": "'$BITBUCKET_COMMIT'",
                          "user": "'$BITBUCKET_STEP_TRIGGERER_UUID'",
                          "environment": "'$DOTENV_ENVIRONMENT'",
                          "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"
                        }'
```

## Docker Service Configuration

### Using Docker Compose

```yaml
pipelines:
    default:
        - step:
              name: Test with Services
              services:
                  - docker
              script:
                  # Install DotEnv
                  - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                  - export PATH="$HOME/.dotenv/bin:$PATH"

                  # Load secrets
                  - dotenv auth login --api-key $DOTENV_API_KEY
                  - dotenv pull test --project $BITBUCKET_REPO_SLUG

                  # Create docker-compose override with secrets
                  - |
                      cat > docker-compose.override.yml << EOF
                      version: '3.8'
                      services:
                        app:
                          env_file: .env
                      EOF

                  # Run tests with services
                  - docker-compose up -d
                  - docker-compose run app npm test
                  - docker-compose down
```

## Caching Strategies

### Optimize Build Times

```yaml
definitions:
    caches:
        dotenv-cli: ~/.dotenv
        npm: node_modules
        build: dist

pipelines:
    default:
        - step:
              name: Install Dependencies
              caches:
                  - dotenv-cli
                  - npm
              script:
                  - |
                      # Check if DotEnv CLI is cached
                      if [ ! -f "$HOME/.dotenv/bin/dotenv" ]; then
                        curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                      fi
                  - export PATH="$HOME/.dotenv/bin:$PATH"
                  - dotenv --version
                  - npm ci
              artifacts:
                  - node_modules/**

        - step:
              name: Build
              caches:
                  - build
              script:
                  - npm run build
              artifacts:
                  - dist/**

        - step:
              name: Deploy
              script:
                  - export PATH="$HOME/.dotenv/bin:$PATH"
                  - dotenv auth login --api-key $DOTENV_API_KEY
                  - dotenv pull production --project $BITBUCKET_REPO_SLUG
                  - source .env
                  - npm run deploy
```

## Monitoring and Notifications

### Slack Integration

```yaml
pipelines:
    branches:
        main:
            - step:
                  name: Deploy and Notify
                  deployment: production
                  script:
                      # ... deployment steps ...
                  after-script:
                      - |
                          # Send deployment notification
                          if [ "$BITBUCKET_EXIT_CODE" -eq 0 ]; then
                            STATUS="success"
                            COLOR="good"
                          else
                            STATUS="failure"
                            COLOR="danger"
                          fi

                          curl -X POST $SLACK_WEBHOOK_URL \
                            -H 'Content-Type: application/json' \
                            -d '{
                              "attachments": [{
                                "color": "'$COLOR'",
                                "title": "Deployment '$STATUS'",
                                "fields": [
                                  {"title": "Repository", "value": "'$BITBUCKET_REPO_FULL_NAME'", "short": true},
                                  {"title": "Branch", "value": "'$BITBUCKET_BRANCH'", "short": true},
                                  {"title": "Commit", "value": "'$BITBUCKET_COMMIT'", "short": true},
                                  {"title": "Author", "value": "'$BITBUCKET_COMMIT_AUTHOR'", "short": true}
                                ]
                              }]
                            }'
```

## Troubleshooting

### Common Issues

#### Pipeline Timeouts

```yaml
pipelines:
    default:
        - step:
              name: Long Running Task
              # Increase timeout (max 120 minutes)
              max-time: 60
              script:
                  -  # ... long running tasks
```

#### Memory Issues

```yaml
definitions:
    steps:
        - step: &large-build
              size: 2x # 8GB memory
              script:
                  - npm run build:production
```

#### Debugging Failed Steps

```yaml
pipelines:
    default:
        - step:
              name: Debug Build
              script:
                  - |
                      # Enable debug mode
                      set -x
                      export DOTENV_DEBUG=true
                      export DOTENV_LOG_LEVEL=debug

                      # Install with verbose output
                      curl -fsSL https://cli.dotenv.cloud/install.sh | sh -x

                      # Check environment
                      echo "Current directory: $(pwd)"
                      echo "User: $(whoami)"
                      echo "Home: $HOME"
                      echo "Path: $PATH"

                      # Test DotEnv CLI
                      $HOME/.dotenv/bin/dotenv --version || echo "DotEnv CLI not found"
```

## Examples

### Complete CI/CD Pipeline

```yaml
image: node:18

definitions:
    caches:
        dotenv: ~/.dotenv
        sonar: ~/.sonar/cache

    steps:
        - step: &build-test
              name: Build and Test
              caches:
                  - node
                  - dotenv
              script:
                  - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                  - export PATH="$HOME/.dotenv/bin:$PATH"
                  - dotenv auth login --api-key $DOTENV_API_KEY
                  - dotenv pull $DOTENV_ENVIRONMENT --project $BITBUCKET_REPO_SLUG
                  - source .env
                  - npm ci
                  - npm run build
                  - npm run test:ci
              artifacts:
                  - dist/**
                  - coverage/**
              after-script:
                  - rm -f .env

pipelines:
    pull-requests:
        "**":
            - step:
                  <<: *build-test
                  name: PR Validation
                  script:
                      - export DOTENV_ENVIRONMENT=development
                      - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                      - export PATH="$HOME/.dotenv/bin:$PATH"
                      - dotenv auth login --api-key $DOTENV_API_KEY
                      - dotenv pull $DOTENV_ENVIRONMENT --project $BITBUCKET_REPO_SLUG
                      - source .env
                      - npm ci
                      - npm run lint
                      - npm run test:ci
                      - npm run build

            - step:
                  name: SonarQube Analysis
                  caches:
                      - sonar
                  script:
                      - pipe: sonarsource/sonarqube-scan:1.0.0
                        variables:
                            SONAR_HOST_URL: ${SONAR_HOST_URL}
                            SONAR_TOKEN: ${SONAR_TOKEN}

    branches:
        develop:
            - step:
                  <<: *build-test
                  name: Development Build
                  deployment: development
                  script:
                      - export DOTENV_ENVIRONMENT=development
                      - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                      - export PATH="$HOME/.dotenv/bin:$PATH"
                      - dotenv auth login --api-key $DOTENV_API_KEY
                      - dotenv pull $DOTENV_ENVIRONMENT --project $BITBUCKET_REPO_SLUG
                      - source .env
                      - npm ci
                      - npm run build:dev
                      - npm run test:ci

            - step:
                  name: Deploy to Development
                  deployment: development
                  script:
                      - pipe: atlassian/aws-s3-deploy:1.1.0
                        variables:
                            AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
                            AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
                            AWS_DEFAULT_REGION: ${AWS_DEFAULT_REGION}
                            S3_BUCKET: "dev-bucket"
                            LOCAL_PATH: "dist"

        staging:
            - step:
                  <<: *build-test
                  name: Staging Build
                  deployment: staging
                  script:
                      - export DOTENV_ENVIRONMENT=staging
                      - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                      - export PATH="$HOME/.dotenv/bin:$PATH"
                      - dotenv auth login --api-key $DOTENV_API_KEY
                      - dotenv pull $DOTENV_ENVIRONMENT --project $BITBUCKET_REPO_SLUG
                      - source .env
                      - npm ci
                      - npm run build:staging
                      - npm run test:ci

            - step:
                  name: Deploy to Staging
                  deployment: staging
                  trigger: manual
                  script:
                      - pipe: atlassian/aws-cloudformation-deploy:0.16.1
                        variables:
                            AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
                            AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
                            AWS_DEFAULT_REGION: ${AWS_DEFAULT_REGION}
                            STACK_NAME: "staging-stack"
                            TEMPLATE: "cloudformation/staging.yml"
                            WAIT: "true"

        main:
            - parallel:
                  - step:
                        <<: *build-test
                        name: Production Build
                        deployment: production
                        script:
                            - export DOTENV_ENVIRONMENT=production
                            - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                            - export PATH="$HOME/.dotenv/bin:$PATH"
                            - dotenv auth login --api-key $DOTENV_API_KEY
                            - dotenv pull $DOTENV_ENVIRONMENT --project $BITBUCKET_REPO_SLUG
                            - source .env
                            - npm ci
                            - npm run build:prod
                            - npm run test:ci

                  - step:
                        name: Security Scan
                        script:
                            - pipe: atlassian/git-secrets-scan:0.5.1

            - step:
                  name: Deploy to Production
                  deployment: production
                  trigger: manual
                  script:
                      - export PATH="$HOME/.dotenv/bin:$PATH"
                      - dotenv auth login --api-key $DOTENV_API_KEY
                      - dotenv pull production --project $BITBUCKET_REPO_SLUG
                      - source .env
                      - |
                          # Blue-green deployment
                          ./scripts/deploy-production.sh
                      - |
                          # Notify deployment
                          curl -X POST https://api.example.com/deployments \
                            -H "Authorization: Bearer $DEPLOYMENT_TOKEN" \
                            -H "Content-Type: application/json" \
                            -d '{
                              "service": "'$BITBUCKET_REPO_SLUG'",
                              "version": "'$BITBUCKET_COMMIT'",
                              "environment": "production",
                              "deployed_by": "'$BITBUCKET_STEP_TRIGGERER_UUID'",
                              "deployed_at": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"
                            }'
```

## Resources

- [Bitbucket Pipelines Documentation](https://support.atlassian.com/bitbucket-cloud/docs/get-started-with-bitbucket-pipelines/)
- [Pipeline Configuration Reference](https://support.atlassian.com/bitbucket-cloud/docs/bitbucket-pipelines-configuration-reference/)
- [Repository Variables](https://support.atlassian.com/bitbucket-cloud/docs/variables-and-secrets/)
- [DotEnv CLI Reference](/documentation/v1/cli/commands)

## Next Steps

- [Set up repository variables](#repository-variables)
- [Configure your first pipeline](#basic-configuration)
- [Implement branch deployments](#branch-based-deployments)
- [Enable security features](#security-best-practices)
