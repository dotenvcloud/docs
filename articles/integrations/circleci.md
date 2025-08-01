---
title: CircleCI Integration
slug: circleci
order: 4
tags: [integrations, circleci, ci-cd]
---

# CircleCI Integration

Integrate DotEnv with CircleCI to manage secrets in your continuous integration pipelines. This guide covers setup, configuration, and best practices.

## Overview

The DotEnv CircleCI integration provides:

- Secure secret injection into workflows
- Environment-based deployments
- Orb for simplified configuration
- Parallel job support
- Audit logging and compliance

## Setup

### 1. Add API Key to CircleCI

1. Go to Project Settings → Environment Variables
2. Add a new environment variable:
    - Name: `DOTENV_API_KEY`
    - Value: Your DotEnv API key

### 2. Install DotEnv Orb

Add the DotEnv orb to your `.circleci/config.yml`:

```yaml
version: 2.1

orbs:
    dotenv: dotenv/dotenv@1.0.0
```

## Basic Configuration

### Simple Workflow

```yaml
version: 2.1

orbs:
    dotenv: dotenv/dotenv@1.0.0

jobs:
    build:
        docker:
            - image: cimg/node:18.0
        steps:
            - checkout
            - dotenv/load:
                  project: my-app
                  environment: production
            - run:
                  name: Build application
                  command: |
                      npm install
                      npm run build

workflows:
    main:
        jobs:
            - build
```

### Manual CLI Installation

```yaml
version: 2.1

jobs:
    build:
        docker:
            - image: cimg/base:stable
        steps:
            - checkout
            - run:
                  name: Install DotEnv CLI
                  command: |
                      curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                      echo 'export PATH="$HOME/.dotenv/bin:$PATH"' >> $BASH_ENV
                      source $BASH_ENV
            - run:
                  name: Load secrets
                  command: |
                      dotenv auth login --api-key $DOTENV_API_KEY
                      dotenv pull production --project my-app
                      source .env
            - run:
                  name: Build
                  command: npm run build
```

## Environment Management

### Branch-Based Environments

```yaml
version: 2.1

orbs:
    dotenv: dotenv/dotenv@1.0.0

jobs:
    deploy:
        docker:
            - image: cimg/node:18.0
        parameters:
            environment:
                type: string
        steps:
            - checkout
            - dotenv/load:
                  project: my-app
                  environment: << parameters.environment >>
            - run:
                  name: Deploy to << parameters.environment >>
                  command: npm run deploy:<< parameters.environment >>

workflows:
    deploy-all:
        jobs:
            - deploy:
                  name: deploy-dev
                  environment: development
                  filters:
                      branches:
                          only: develop
            - deploy:
                  name: deploy-staging
                  environment: staging
                  filters:
                      branches:
                          only: staging
            - deploy:
                  name: deploy-prod
                  environment: production
                  filters:
                      branches:
                          only: main
```

### Context-Based Configuration

```yaml
version: 2.1

orbs:
    dotenv: dotenv/dotenv@1.0.0

workflows:
    main:
        jobs:
            - build:
                  context: dotenv-dev
                  filters:
                      branches:
                          ignore: main
            - build:
                  context: dotenv-prod
                  filters:
                      branches:
                          only: main

jobs:
    build:
        docker:
            - image: cimg/node:18.0
        steps:
            - checkout
            - dotenv/load:
                  project: my-app
                  environment: ${DOTENV_ENVIRONMENT}
            - run: npm run build
```

## Advanced Features

### 1. Parallel Jobs

```yaml
version: 2.1

orbs:
    dotenv: dotenv/dotenv@1.0.0

jobs:
    test:
        parallelism: 4
        docker:
            - image: cimg/node:18.0
        steps:
            - checkout
            - dotenv/load:
                  project: my-app
                  environment: test
            - run:
                  name: Run tests in parallel
                  command: |
                      TESTFILES=$(circleci tests glob "test/**/*.test.js" | circleci tests split --split-by=timings)
                      npm test $TESTFILES

workflows:
    test:
        jobs:
            - test
```

### 2. Matrix Jobs

```yaml
version: 2.1

orbs:
    dotenv: dotenv/dotenv@1.0.0

jobs:
    test:
        parameters:
            node-version:
                type: string
            environment:
                type: string
        docker:
            - image: cimg/node:<< parameters.node-version >>
        steps:
            - checkout
            - dotenv/load:
                  project: my-app
                  environment: << parameters.environment >>
            - run:
                  name: Test on Node << parameters.node-version >>
                  command: |
                      npm install
                      npm test

workflows:
    test-matrix:
        jobs:
            - test:
                  matrix:
                      parameters:
                          node-version: ["16.0", "18.0", "20.0"]
                          environment: ["test", "staging"]
```

### 3. Docker Integration

```yaml
version: 2.1

orbs:
    dotenv: dotenv/dotenv@1.0.0

jobs:
    build-docker:
        docker:
            - image: cimg/base:stable
        steps:
            - checkout
            - setup_remote_docker:
                  docker_layer_caching: true
            - dotenv/load:
                  project: my-app
                  environment: production
                  format: docker
                  output: .env.docker
            - run:
                  name: Build Docker image
                  command: |
                      docker build \
                        --secret id=dotenv,src=.env.docker \
                        -t myapp:${CIRCLE_SHA1} .
            - run:
                  name: Push to registry
                  command: |
                      echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
                      docker push myapp:${CIRCLE_SHA1}
```

### 4. Approval Workflows

```yaml
version: 2.1

orbs:
    dotenv: dotenv/dotenv@1.0.0

jobs:
    build:
        docker:
            - image: cimg/node:18.0
        steps:
            - checkout
            - dotenv/load:
                  project: my-app
                  environment: staging
            - run: npm run build
            - persist_to_workspace:
                  root: .
                  paths:
                      - dist

    deploy-production:
        docker:
            - image: cimg/node:18.0
        steps:
            - attach_workspace:
                  at: .
            - dotenv/load:
                  project: my-app
                  environment: production
            - run: npm run deploy:production

workflows:
    build-and-deploy:
        jobs:
            - build
            - hold:
                  type: approval
                  requires:
                      - build
            - deploy-production:
                  requires:
                      - hold
```

## Security Best Practices

### 1. Restricted Contexts

Use CircleCI contexts for environment isolation:

```yaml
workflows:
    deploy:
        jobs:
            - deploy-staging:
                  context: staging-secrets
            - deploy-production:
                  context: production-secrets
                  filters:
                      branches:
                          only: main
```

### 2. Temporary Secret Files

Clean up secret files after use:

```yaml
jobs:
    secure-job:
        docker:
            - image: cimg/base:stable
        steps:
            - checkout
            - dotenv/load:
                  project: my-app
                  environment: production
            - run:
                  name: Use secrets
                  command: |
                      # Your commands here
                      echo "Using secrets..."
            - run:
                  name: Cleanup
                  command: |
                      shred -vfz .env 2>/dev/null || rm -f .env
                  when: always
```

### 3. Audit Logging

Enable audit trails:

```yaml
version: 2.1

orbs:
    dotenv: dotenv/dotenv@1.0.0

commands:
    load-with-audit:
        parameters:
            environment:
                type: string
        steps:
            - dotenv/load:
                  project: my-app
                  environment: << parameters.environment >>
                  audit-metadata: |
                      workflow_id=${CIRCLE_WORKFLOW_ID}
                      job_name=${CIRCLE_JOB}
                      branch=${CIRCLE_BRANCH}
                      user=${CIRCLE_USERNAME}
```

### 4. Secret Scanning

Prevent accidental secret exposure:

```yaml
jobs:
    security-scan:
        docker:
            - image: cimg/base:stable
        steps:
            - checkout
            - run:
                  name: Install secret scanner
                  command: |
                      curl -sSfL https://raw.githubusercontent.com/trufflesecurity/trufflehog/main/scripts/install.sh | sh -s -- -b /usr/local/bin
            - run:
                  name: Scan for secrets
                  command: |
                      trufflehog git file://. --only-verified --fail
```

## Performance Optimization

### 1. Caching

Cache DotEnv CLI installation:

```yaml
version: 2.1

jobs:
    build:
        docker:
            - image: cimg/node:18.0
        steps:
            - checkout
            - restore_cache:
                  keys:
                      - dotenv-cli-v1-{{ arch }}
            - run:
                  name: Install DotEnv CLI
                  command: |
                      if [ ! -f "$HOME/.dotenv/bin/dotenv" ]; then
                        curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                      fi
                      echo 'export PATH="$HOME/.dotenv/bin:$PATH"' >> $BASH_ENV
            - save_cache:
                  key: dotenv-cli-v1-{{ arch }}
                  paths:
                      - ~/.dotenv
            - run:
                  name: Load secrets
                  command: |
                      source $BASH_ENV
                      dotenv auth login --api-key $DOTENV_API_KEY
                      dotenv pull production --project my-app
```

### 2. Workspace Persistence

Share secrets across jobs:

```yaml
version: 2.1

orbs:
    dotenv: dotenv/dotenv@1.0.0

jobs:
    load-secrets:
        docker:
            - image: cimg/base:stable
        steps:
            - dotenv/load:
                  project: my-app
                  environment: production
                  format: json
                  output: secrets.json
            - persist_to_workspace:
                  root: .
                  paths:
                      - secrets.json

    use-secrets:
        docker:
            - image: cimg/node:18.0
        steps:
            - attach_workspace:
                  at: .
            - run:
                  name: Load secrets from JSON
                  command: |
                      export $(cat secrets.json | jq -r 'to_entries|map("\(.key)=\(.value|tostring)")|.[]')
            - run:
                  name: Use secrets
                  command: echo "API endpoint is $API_URL"
```

## Troubleshooting

### Debug Mode

Enable verbose logging:

```yaml
jobs:
    debug:
        docker:
            - image: cimg/base:stable
        environment:
            DOTENV_DEBUG: true
            DOTENV_LOG_LEVEL: debug
        steps:
            - checkout
            - dotenv/load:
                  project: my-app
                  environment: development
                  debug: true
```

### Common Issues

#### Authentication Failures

```yaml
- run:
      name: Debug authentication
      command: |
          dotenv auth status
          dotenv project list
```

#### Network Issues

```yaml
- run:
      name: Test connectivity
      command: |
          curl -I https://api.dotenv.cloud/v1/health
          dotenv status
```

#### Environment Variables Not Set

```yaml
- run:
      name: Debug environment
      command: |
          # Check if secrets are loaded
          env | grep -E "^(API_|DATABASE_)" || echo "No secrets found"

          # Re-source if needed
          if [ -f .env ]; then
            set -a
            source .env
            set +a
          fi
```

## Examples

### Complete CI/CD Pipeline

```yaml
version: 2.1

orbs:
    dotenv: dotenv/dotenv@1.0.0
    node: circleci/node@5.0.0

executors:
    node-executor:
        docker:
            - image: cimg/node:18.0

jobs:
    test:
        executor: node-executor
        steps:
            - checkout
            - node/install-packages
            - dotenv/load:
                  project: my-app
                  environment: test
            - run:
                  name: Run tests
                  command: npm test
            - run:
                  name: Generate coverage
                  command: npm run coverage
            - store_test_results:
                  path: test-results
            - store_artifacts:
                  path: coverage

    build:
        executor: node-executor
        steps:
            - checkout
            - node/install-packages
            - dotenv/load:
                  project: my-app
                  environment: staging
            - run:
                  name: Build application
                  command: npm run build
            - persist_to_workspace:
                  root: .
                  paths:
                      - dist
                      - package.json
                      - package-lock.json

    deploy-staging:
        executor: node-executor
        steps:
            - attach_workspace:
                  at: .
            - dotenv/load:
                  project: my-app
                  environment: staging
            - run:
                  name: Deploy to staging
                  command: npm run deploy:staging
            - run:
                  name: Run smoke tests
                  command: npm run test:smoke

    deploy-production:
        executor: node-executor
        steps:
            - attach_workspace:
                  at: .
            - dotenv/load:
                  project: my-app
                  environment: production
            - run:
                  name: Deploy to production
                  command: npm run deploy:production
            - run:
                  name: Notify deployment
                  command: |
                      curl -X POST $SLACK_WEBHOOK \
                        -H 'Content-Type: application/json' \
                        -d '{"text":"Deployed to production: '"$CIRCLE_SHA1"'"}'

workflows:
    version: 2
    build-test-deploy:
        jobs:
            - test
            - build:
                  requires:
                      - test
            - deploy-staging:
                  requires:
                      - build
                  filters:
                      branches:
                          only: staging
            - hold:
                  type: approval
                  requires:
                      - build
                  filters:
                      branches:
                          only: main
            - deploy-production:
                  requires:
                      - hold
                  filters:
                      branches:
                          only: main
```

### Scheduled Secret Rotation

```yaml
version: 2.1

orbs:
    dotenv: dotenv/dotenv@1.0.0

jobs:
    rotate-secrets:
        docker:
            - image: cimg/base:stable
        steps:
            - checkout
            - run:
                  name: Install DotEnv CLI
                  command: |
                      curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                      echo 'export PATH="$HOME/.dotenv/bin:$PATH"' >> $BASH_ENV
            - run:
                  name: Rotate secrets
                  command: |
                      source $BASH_ENV
                      dotenv auth login --api-key $DOTENV_API_KEY

                      # Rotate specific secrets
                      dotenv secret rotate \
                        --project my-app \
                        --environment production \
                        --key API_KEY
                        
                      dotenv secret rotate \
                        --project my-app \
                        --environment production \
                        --key DATABASE_PASSWORD
            - run:
                  name: Notify team
                  command: |
                      curl -X POST $SLACK_WEBHOOK \
                        -H 'Content-Type: application/json' \
                        -d '{"text":"Monthly secret rotation completed"}'

workflows:
    version: 2
    monthly-rotation:
        triggers:
            - schedule:
                  cron: "0 0 1 * *"
                  filters:
                      branches:
                          only:
                              - main
        jobs:
            - rotate-secrets
```

## Custom Orb Commands

Create custom commands for your organization:

```yaml
version: 2.1

commands:
    load-project-secrets:
        description: Load secrets for our project
        parameters:
            environment:
                type: enum
                enum: ["development", "staging", "production"]
                default: development
        steps:
            - run:
                  name: Install DotEnv CLI
                  command: |
                      curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                      echo 'export PATH="$HOME/.dotenv/bin:$PATH"' >> $BASH_ENV
            - run:
                  name: Load << parameters.environment >> secrets
                  command: |
                      source $BASH_ENV
                      dotenv auth login --api-key $DOTENV_API_KEY
                      dotenv pull << parameters.environment >> --project ${CIRCLE_PROJECT_REPONAME}
                      source .env
            - run:
                  name: Validate secrets
                  command: |
                      required_vars=("DATABASE_URL" "API_KEY" "JWT_SECRET")
                      for var in "${required_vars[@]}"; do
                        if [ -z "${!var}" ]; then
                          echo "ERROR: Required variable $var is not set"
                          exit 1
                        fi
                      done

jobs:
    deploy:
        docker:
            - image: cimg/node:18.0
        steps:
            - checkout
            - load-project-secrets:
                  environment: production
            - run: npm run deploy
```

## Resources

- [CircleCI Documentation](https://circleci.com/docs/)
- [DotEnv Orb Registry](https://circleci.com/developer/orbs/orb/dotenv/dotenv)
- [DotEnv CLI Reference](/documentation/v1/cli/commands)
- [API Documentation](/documentation/v1/api/overview)

## Next Steps

- [Set up your first workflow](#basic-configuration)
- [Configure approval workflows](#approval-workflows)
- [Implement caching](#caching)
- [Enable security scanning](#secret-scanning)
