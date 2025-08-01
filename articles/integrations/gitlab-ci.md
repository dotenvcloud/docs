---
title: GitLab CI Integration
slug: gitlab-ci
order: 3
tags: [integrations, gitlab, ci-cd, gitlab-ci]
---

# GitLab CI Integration

Integrate DotEnv with GitLab CI/CD to manage secrets across your pipelines. This guide covers configuration, best practices, and advanced patterns.

## Overview

The DotEnv GitLab CI integration enables:

- Secure secret injection into pipelines
- Environment-specific deployments
- Secret validation and rotation
- Compliance and audit logging
- Multi-project secret management

## Setup

### 1. Store API Key

Add your DotEnv API key as a CI/CD variable:

1. Navigate to Settings → CI/CD → Variables
2. Add a new variable:
    - Key: `DOTENV_API_KEY`
    - Value: Your DotEnv API key
    - Type: Variable
    - Protected: Yes (for protected branches)
    - Masked: Yes (to hide in logs)

### 2. Install DotEnv CLI

Create a `.gitlab-ci.yml` with CLI installation:

```yaml
variables:
    DOTENV_VERSION: "latest"

before_script:
    - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
    - export PATH="$HOME/.dotenv/bin:$PATH"
    - dotenv --version
```

## Basic Configuration

### Simple Pipeline

```yaml
stages:
    - build
    - test
    - deploy

variables:
    PROJECT_NAME: "my-app"

.load_secrets: &load_secrets
    - dotenv auth login --api-key $DOTENV_API_KEY
    - dotenv pull $ENVIRONMENT --project $PROJECT_NAME
    - source .env

build:
    stage: build
    variables:
        ENVIRONMENT: "development"
    before_script:
        - *load_secrets
    script:
        - echo "Building with $NODE_ENV environment"
        - npm install
        - npm run build
    artifacts:
        paths:
            - dist/

test:
    stage: test
    variables:
        ENVIRONMENT: "test"
    before_script:
        - *load_secrets
    script:
        - npm test
    coverage: '/Coverage: \d+\.\d+%/'

deploy:
    stage: deploy
    variables:
        ENVIRONMENT: "production"
    before_script:
        - *load_secrets
    script:
        - npm run deploy
    only:
        - main
    environment:
        name: production
        url: https://app.example.com
```

## Environment Management

### Branch-Based Environments

```yaml
.determine_environment: &determine_environment
    - |
        if [[ "$CI_COMMIT_BRANCH" == "main" ]]; then
          export DOTENV_ENV="production"
        elif [[ "$CI_COMMIT_BRANCH" == "staging" ]]; then
          export DOTENV_ENV="staging"
        else
          export DOTENV_ENV="development"
        fi

deploy:
    stage: deploy
    before_script:
        - *determine_environment
        - dotenv auth login --api-key $DOTENV_API_KEY
        - dotenv pull $DOTENV_ENV --project $PROJECT_NAME
        - source .env
    script:
        - echo "Deploying to $DOTENV_ENV"
        - npm run deploy:$DOTENV_ENV
```

### Environment-Specific Jobs

```yaml
deploy:staging:
    stage: deploy
    variables:
        ENVIRONMENT: "staging"
    before_script:
        - dotenv auth login --api-key $DOTENV_API_KEY
        - dotenv pull staging --project $PROJECT_NAME
        - source .env
    script:
        - npm run deploy:staging
    only:
        - staging
    environment:
        name: staging
        url: https://staging.example.com

deploy:production:
    stage: deploy
    variables:
        ENVIRONMENT: "production"
    before_script:
        - dotenv auth login --api-key $DOTENV_API_KEY
        - dotenv pull production --project $PROJECT_NAME
        - source .env
    script:
        - npm run deploy:production
    only:
        - main
    environment:
        name: production
        url: https://app.example.com
    when: manual
```

## Advanced Patterns

### 1. Docker Integration

```yaml
build:docker:
    stage: build
    image: docker:latest
    services:
        - docker:dind
    variables:
        DOCKER_DRIVER: overlay2
        DOCKER_TLS_CERTDIR: "/certs"
    before_script:
        - apk add --no-cache curl bash
        - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
        - export PATH="$HOME/.dotenv/bin:$PATH"
        - dotenv auth login --api-key $DOTENV_API_KEY
        - dotenv pull production --project $PROJECT_NAME --format docker > .env.docker
    script:
        - docker build --secret id=dotenv,src=.env.docker -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
        - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    after_script:
        - rm -f .env.docker
```

### 2. Multi-Project Monorepo

```yaml
.base_job:
    before_script:
        - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
        - export PATH="$HOME/.dotenv/bin:$PATH"
        - dotenv auth login --api-key $DOTENV_API_KEY

api:build:
    extends: .base_job
    stage: build
    variables:
        SERVICE: "api"
    script:
        - dotenv pull production --project "${PROJECT_NAME}-api" --prefix API_
        - source .env
        - cd services/api
        - npm install
        - npm run build

web:build:
    extends: .base_job
    stage: build
    variables:
        SERVICE: "web"
    script:
        - dotenv pull production --project "${PROJECT_NAME}-web" --prefix WEB_
        - source .env
        - cd services/web
        - npm install
        - npm run build
```

### 3. Secret Validation

```yaml
validate:secrets:
    stage: .pre
    before_script:
        - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
        - export PATH="$HOME/.dotenv/bin:$PATH"
        - dotenv auth login --api-key $DOTENV_API_KEY
    script:
        - |
            echo "Validating secrets for all environments..."
            for env in development staging production; do
              echo "Checking $env environment..."
              dotenv validate --project $PROJECT_NAME --environment $env
              
              # Check required secrets
              dotenv pull $env --project $PROJECT_NAME --format json > secrets.json
              
              required_secrets=("DATABASE_URL" "API_KEY" "JWT_SECRET")
              for secret in "${required_secrets[@]}"; do
                if ! jq -e "has(\"$secret\")" secrets.json > /dev/null; then
                  echo "ERROR: Missing required secret: $secret in $env"
                  exit 1
                fi
              done
              
              rm secrets.json
            done
    rules:
        - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

### 4. Parallel Deployments

```yaml
deploy:regions:
    stage: deploy
    parallel:
        matrix:
            - REGION: us-east-1
              DOTENV_PROJECT: my-app-us-east
            - REGION: eu-west-1
              DOTENV_PROJECT: my-app-eu-west
            - REGION: ap-southeast-1
              DOTENV_PROJECT: my-app-ap-southeast
    before_script:
        - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
        - export PATH="$HOME/.dotenv/bin:$PATH"
        - dotenv auth login --api-key $DOTENV_API_KEY
        - dotenv pull production --project $DOTENV_PROJECT
        - source .env
    script:
        - echo "Deploying to $REGION"
        - npm run deploy -- --region $REGION
    environment:
        name: production-$REGION
```

## Security Best Practices

### 1. Protected Variables

Use GitLab's protected variables:

```yaml
# Only available on protected branches
deploy:production:
    variables:
        DOTENV_API_KEY: $DOTENV_API_KEY_PROD # Protected variable
    script:
        - dotenv pull production --project $PROJECT_NAME
    only:
        - main
```

### 2. Temporary Secrets

Clean up secrets after use:

```yaml
.secure_job:
    after_script:
        - shred -vfz .env 2>/dev/null || rm -f .env
        - unset $(grep -v '^#' .env | sed -E 's/(.*)=.*/\1/' | xargs)
```

### 3. Audit Logging

Enable audit trails:

```yaml
deploy:
    before_script:
        - dotenv auth login --api-key $DOTENV_API_KEY
        - |
            dotenv pull production \
              --project $PROJECT_NAME \
              --audit-metadata "pipeline_id=$CI_PIPELINE_ID,job_id=$CI_JOB_ID,user=$GITLAB_USER_LOGIN"
```

### 4. Role-Based Access

Use environment-specific API keys:

```yaml
deploy:dev:
    variables:
        DOTENV_API_KEY: $DOTENV_API_KEY_DEV # Read-only for dev

deploy:prod:
    variables:
        DOTENV_API_KEY: $DOTENV_API_KEY_PROD # Production access
    only:
        - main
```

## GitLab Features Integration

### 1. Review Apps

```yaml
review:
    stage: deploy
    variables:
        ENVIRONMENT: "review-$CI_MERGE_REQUEST_IID"
    before_script:
        - dotenv auth login --api-key $DOTENV_API_KEY
        - dotenv env create $ENVIRONMENT --project $PROJECT_NAME --from development || true
        - dotenv pull $ENVIRONMENT --project $PROJECT_NAME
        - source .env
    script:
        - npm run deploy -- --environment $ENVIRONMENT
    environment:
        name: review/$CI_MERGE_REQUEST_IID
        url: https://$CI_ENVIRONMENT_SLUG.example.com
        on_stop: stop_review
    only:
        - merge_requests

stop_review:
    stage: deploy
    variables:
        ENVIRONMENT: "review-$CI_MERGE_REQUEST_IID"
    before_script:
        - dotenv auth login --api-key $DOTENV_API_KEY
    script:
        - dotenv env delete $ENVIRONMENT --project $PROJECT_NAME --force
    environment:
        name: review/$CI_MERGE_REQUEST_IID
        action: stop
    when: manual
    only:
        - merge_requests
```

### 2. Scheduled Pipelines

```yaml
rotate:secrets:
    stage: maintenance
    before_script:
        - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
        - export PATH="$HOME/.dotenv/bin:$PATH"
        - dotenv auth login --api-key $DOTENV_API_KEY
    script:
        - |
            echo "Rotating secrets..."
            dotenv secret rotate \
              --project $PROJECT_NAME \
              --environment production \
              --key EXTERNAL_API_KEY \
              --notify ops@example.com
    only:
        - schedules
    variables:
        SCHEDULE_TYPE: "secret_rotation"
```

### 3. Child Pipelines

```yaml
# .gitlab-ci.yml
trigger:services:
    stage: deploy
    trigger:
        include:
            - local: .gitlab/ci/api.yml
            - local: .gitlab/ci/web.yml
        strategy: depend
    variables:
        PARENT_PIPELINE_ID: $CI_PIPELINE_ID

# .gitlab/ci/api.yml
api:deploy:
    before_script:
        - dotenv auth login --api-key $DOTENV_API_KEY
        - dotenv pull production --project "${PROJECT_NAME}-api"
        - source .env
    script:
        - cd services/api
        - npm run deploy
```

## Troubleshooting

### Debug Mode

Enable verbose logging:

```yaml
debug:secrets:
    variables:
        DOTENV_DEBUG: "true"
        DOTENV_LOG_LEVEL: "debug"
    script:
        - dotenv auth status
        - dotenv project list
        - dotenv env list --project $PROJECT_NAME
        - dotenv pull development --project $PROJECT_NAME --dry-run
```

### Common Issues

#### Permission Denied

```yaml
fix:permissions:
    before_script:
        - chmod +x $HOME/.dotenv/bin/dotenv
        - export PATH="$HOME/.dotenv/bin:$PATH"
```

#### Network Timeouts

```yaml
.network_retry: &network_retry
    retry:
        max: 3
        when:
            - api_failure
            - runner_system_failure
            - stuck_or_timeout_failure
```

#### Variable Expansion

```yaml
# Use double quotes for variable expansion
script:
  - dotenv pull "$ENVIRONMENT" --project "$PROJECT_NAME"

# Escape special characters
script:
  - dotenv set KEY="value with spaces" --environment production
```

## Performance Optimization

### 1. Cache DotEnv CLI

```yaml
variables:
    DOTENV_CACHE_KEY: "dotenv-cli-$DOTENV_VERSION"

.install_dotenv: &install_dotenv
    - |
        if [ ! -f "$HOME/.dotenv/bin/dotenv" ]; then
          curl -fsSL https://cli.dotenv.cloud/install.sh | sh
        fi
    - export PATH="$HOME/.dotenv/bin:$PATH"

cache:
    key: $DOTENV_CACHE_KEY
    paths:
        - $HOME/.dotenv/
```

### 2. Parallel Secret Loading

```yaml
load:all:secrets:
    stage: .pre
    parallel:
        matrix:
            - ENV: [development, staging, production]
    script:
        - dotenv pull $ENV --project $PROJECT_NAME --format json > $ENV.json
    artifacts:
        paths:
            - "*.json"
        expire_in: 1 hour
```

### 3. Conditional Secret Loading

```yaml
.load_if_changed: &load_if_changed
    - |
        if git diff HEAD~1 --name-only | grep -E "(package\.json|\.env\.example)"; then
          echo "Dependencies changed, reloading secrets..."
          dotenv pull $ENVIRONMENT --project $PROJECT_NAME
        else
          echo "Using cached secrets"
        fi
```

## Examples

### Complete Pipeline

```yaml
image: node:18

stages:
    - validate
    - build
    - test
    - deploy

variables:
    PROJECT_NAME: "my-app"
    DOTENV_VERSION: "latest"

# Templates
.install_dotenv: &install_dotenv
    - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
    - export PATH="$HOME/.dotenv/bin:$PATH"
    - dotenv auth login --api-key $DOTENV_API_KEY

.cleanup_secrets: &cleanup_secrets
    - shred -vfz .env 2>/dev/null || rm -f .env

# Jobs
validate:secrets:
    stage: validate
    before_script: *install_dotenv
    script:
        - dotenv validate --project $PROJECT_NAME --environment production
        - dotenv validate --project $PROJECT_NAME --environment staging
    rules:
        - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

build:app:
    stage: build
    before_script:
        - *install_dotenv
        - dotenv pull development --project $PROJECT_NAME
        - source .env
    script:
        - npm ci
        - npm run build
    after_script: *cleanup_secrets
    artifacts:
        paths:
            - dist/
        expire_in: 1 day

test:unit:
    stage: test
    before_script:
        - *install_dotenv
        - dotenv pull test --project $PROJECT_NAME
        - source .env
    script:
        - npm ci
        - npm test
    after_script: *cleanup_secrets
    coverage: '/Coverage: \d+\.\d+%/'

test:integration:
    stage: test
    services:
        - postgres:14
        - redis:7
    variables:
        POSTGRES_DB: test
        POSTGRES_USER: test
        POSTGRES_PASSWORD: test
    before_script:
        - *install_dotenv
        - dotenv pull test --project $PROJECT_NAME
        - source .env
    script:
        - npm ci
        - npm run test:integration
    after_script: *cleanup_secrets

deploy:staging:
    stage: deploy
    before_script:
        - *install_dotenv
        - dotenv pull staging --project $PROJECT_NAME
        - source .env
    script:
        - npm run deploy:staging
    after_script: *cleanup_secrets
    environment:
        name: staging
        url: https://staging.example.com
    only:
        - staging

deploy:production:
    stage: deploy
    before_script:
        - *install_dotenv
        - dotenv pull production --project $PROJECT_NAME
        - source .env
    script:
        - npm run deploy:production
    after_script: *cleanup_secrets
    environment:
        name: production
        url: https://app.example.com
    only:
        - main
    when: manual
```

## Resources

- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [DotEnv CLI Reference](/documentation/v1/cli/commands)
- [API Documentation](/documentation/v1/api/overview)
- [Security Best Practices](/documentation/v1/security/best-practices)

## Next Steps

- [Configure protected variables](#protected-variables)
- [Set up review apps](#review-apps)
- [Implement secret rotation](#scheduled-pipelines)
- [Enable audit logging](#audit-logging)
