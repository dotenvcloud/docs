---
title: CI/CD Pipeline Integration
slug: ci-cd-pipelines
order: 5
tags: [use-cases, ci-cd, automation, deployment]
---

# CI/CD Pipeline Integration

Learn how to integrate DotEnv into your CI/CD pipelines for automated secret management, secure deployments, and consistent environment configurations across your build and deployment processes.

## Overview

CI/CD integration challenges:

- Secure secret storage
- Build-time vs runtime secrets
- Multi-stage pipelines
- Deployment environments
- Secret rotation automation
- Audit compliance

## Pipeline Architecture

### 1. Pipeline Stages

```yaml
# Pipeline flow with DotEnv integration
stages:
    - name: Setup
      tasks:
          - Install DotEnv CLI
          - Authenticate with API key

    - name: Build
      tasks:
          - Load build secrets
          - Compile application
          - Run tests

    - name: Test
      tasks:
          - Load test secrets
          - Run unit tests
          - Run integration tests

    - name: Deploy
      tasks:
          - Load deployment secrets
          - Deploy to environment
          - Verify deployment

    - name: Cleanup
      tasks:
          - Remove temporary files
          - Clear secret cache
```

### 2. Secret Hierarchy

```javascript
// config/ci-secrets.js
const secretHierarchy = {
    global: {
        // Shared across all pipelines
        NPM_TOKEN: "Organization npm registry token",
        DOCKER_REGISTRY: "Docker registry URL",
        SONAR_TOKEN: "SonarQube analysis token",
    },

    build: {
        // Build-specific secrets
        BUILD_CERTIFICATE: "Code signing certificate",
        MAVEN_CREDENTIALS: "Maven repository credentials",
        NUGET_API_KEY: "NuGet package API key",
    },

    test: {
        // Test environment secrets
        TEST_DATABASE_URL: "Test database connection",
        MOCK_API_KEY: "Mock service API key",
        BROWSER_STACK_KEY: "Browser testing service key",
    },

    deploy: {
        // Deployment secrets
        AWS_ACCESS_KEY_ID: "AWS deployment credentials",
        KUBERNETES_TOKEN: "K8s cluster access token",
        SLACK_WEBHOOK: "Deployment notification webhook",
    },
};
```

## GitHub Actions Integration

### 1. Reusable Workflow

```yaml
# .github/workflows/dotenv-setup.yml
name: DotEnv Setup

on:
    workflow_call:
        inputs:
            project:
                required: true
                type: string
            environment:
                required: true
                type: string
        secrets:
            DOTENV_API_KEY:
                required: true

jobs:
    setup:
        runs-on: ubuntu-latest
        outputs:
            cache-key: ${{ steps.cache.outputs.cache-key }}
        steps:
            - name: Install DotEnv CLI
              run: |
                  curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                  echo "$HOME/.dotenv/bin" >> $GITHUB_PATH

            - name: Cache DotEnv CLI
              id: cache
              uses: actions/cache@v3
              with:
                  path: ~/.dotenv
                  key: dotenv-cli-${{ runner.os }}-${{ hashFiles('.dotenv-version') }}

            - name: Authenticate DotEnv
              env:
                  DOTENV_API_KEY: ${{ secrets.DOTENV_API_KEY }}
              run: |
                  dotenv auth login --api-key $DOTENV_API_KEY

            - name: Pull secrets
              run: |
                  dotenv pull ${{ inputs.environment }} \
                    --project ${{ inputs.project }} \
                    --format github-actions >> $GITHUB_ENV

            - name: Mask sensitive values
              run: |
                  # Mask all secret values in logs
                  while IFS='=' read -r key value; do
                    if [[ -n "$value" ]]; then
                      echo "::add-mask::$value"
                    fi
                  done < .env
```

### 2. Main Pipeline

```yaml
# .github/workflows/main.yml
name: CI/CD Pipeline

on:
    push:
        branches: [main, develop]
    pull_request:
        branches: [main]

jobs:
    setup-secrets:
        uses: ./.github/workflows/dotenv-setup.yml
        with:
            project: my-app
            environment: ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}
        secrets:
            DOTENV_API_KEY: ${{ secrets.DOTENV_API_KEY }}

    build:
        needs: setup-secrets
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v3

            - name: Setup Node.js
              uses: actions/setup-node@v3
              with:
                  node-version: "18"
                  cache: "npm"

            - name: Install dependencies
              run: npm ci
              env:
                  NPM_TOKEN: ${{ env.NPM_TOKEN }}

            - name: Build application
              run: npm run build
              env:
                  NODE_ENV: production
                  API_ENDPOINT: ${{ env.API_ENDPOINT }}

            - name: Upload artifacts
              uses: actions/upload-artifact@v3
              with:
                  name: build-artifacts
                  path: dist/

    test:
        needs: build
        runs-on: ubuntu-latest
        strategy:
            matrix:
                test-suite: [unit, integration, e2e]
        steps:
            - uses: actions/checkout@v3

            - name: Download artifacts
              uses: actions/download-artifact@v3
              with:
                  name: build-artifacts
                  path: dist/

            - name: Run ${{ matrix.test-suite }} tests
              run: npm run test:${{ matrix.test-suite }}
              env:
                  TEST_DATABASE_URL: ${{ env.TEST_DATABASE_URL }}
                  API_KEY: ${{ env.TEST_API_KEY }}

    deploy:
        needs: [build, test]
        if: github.ref == 'refs/heads/main'
        runs-on: ubuntu-latest
        environment: production
        steps:
            - uses: actions/checkout@v3

            - name: Configure AWS credentials
              uses: aws-actions/configure-aws-credentials@v2
              with:
                  aws-access-key-id: ${{ env.AWS_ACCESS_KEY_ID }}
                  aws-secret-access-key: ${{ env.AWS_SECRET_ACCESS_KEY }}
                  aws-region: us-east-1

            - name: Deploy to ECS
              run: |
                  aws ecs update-service \
                    --cluster production \
                    --service my-app \
                    --force-new-deployment

            - name: Notify deployment
              uses: 8398a7/action-slack@v3
              with:
                  status: ${{ job.status }}
                  webhook_url: ${{ env.SLACK_WEBHOOK }}
```

## GitLab CI Integration

### 1. Pipeline Template

```yaml
# .gitlab-ci.yml
include:
    - local: ".gitlab/dotenv-template.yml"

variables:
    DOTENV_PROJECT: "my-app"
    DOCKER_DRIVER: overlay2

stages:
    - setup
    - build
    - test
    - deploy

.dotenv_setup:
    image: alpine:latest
    before_script:
        - apk add --no-cache curl bash
        - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
        - export PATH="$HOME/.dotenv/bin:$PATH"
        - dotenv auth login --api-key $DOTENV_API_KEY
    cache:
        key: dotenv-${CI_COMMIT_REF_SLUG}
        paths:
            - .dotenv/

setup:secrets:
    extends: .dotenv_setup
    stage: setup
    script:
        - |
            if [[ "$CI_COMMIT_BRANCH" == "main" ]]; then
              ENV="production"
            elif [[ "$CI_COMMIT_BRANCH" == "staging" ]]; then
              ENV="staging"
            else
              ENV="development"
            fi
        - dotenv pull $ENV --project $DOTENV_PROJECT --format gitlab > .env.ci
        - cat .env.ci >> $CI_PROJECT_DIR/.env
    artifacts:
        reports:
            dotenv: .env.ci
        expire_in: 1 hour

build:app:
    stage: build
    image: node:18-alpine
    needs: ["setup:secrets"]
    script:
        - npm ci
        - npm run build
    artifacts:
        paths:
            - dist/
        expire_in: 1 week

test:unit:
    stage: test
    image: node:18-alpine
    needs: ["build:app"]
    script:
        - npm run test:unit
    coverage: '/Coverage: \d+\.\d+%/'

test:integration:
    stage: test
    image: node:18-alpine
    needs: ["build:app"]
    services:
        - postgres:14
        - redis:7
    variables:
        POSTGRES_DB: test
        POSTGRES_USER: test
        POSTGRES_PASSWORD: test
    script:
        - npm run test:integration

deploy:production:
    stage: deploy
    extends: .dotenv_setup
    needs: ["test:unit", "test:integration"]
    only:
        - main
    environment:
        name: production
        url: https://app.example.com
    script:
        - dotenv pull production --project $DOTENV_PROJECT
        - source .env
        - |
            helm upgrade --install my-app ./charts/my-app \
              --namespace production \
              --set image.tag=$CI_COMMIT_SHA \
              --set-string secrets.database=$DATABASE_URL \
              --set-string secrets.redis=$REDIS_URL
```

### 2. Secure Variables

```yaml
# .gitlab/dotenv-template.yml
.dotenv_variables: &dotenv_variables
    DOTENV_API_KEY:
        description: "DotEnv API key for CI/CD"
        value: "$DOTENV_API_KEY"
        masked: true
        protected: true
        environment_scope: "*"

.production_variables: &production_variables
    <<: *dotenv_variables
    DEPLOY_KEY:
        description: "Production deployment key"
        value: "$PRODUCTION_DEPLOY_KEY"
        masked: true
        protected: true
        environment_scope: "production"
```

## Jenkins Pipeline

### 1. Declarative Pipeline

```groovy
// Jenkinsfile
@Library('shared-pipeline-library') _

pipeline {
    agent any

    environment {
        DOTENV_PROJECT = 'my-app'
        DOTENV_CLI_PATH = "${WORKSPACE}/.dotenv/bin"
    }

    stages {
        stage('Setup') {
            steps {
                script {
                    // Install DotEnv CLI if not cached
                    if (!fileExists("${DOTENV_CLI_PATH}/dotenv")) {
                        sh '''
                            curl -fsSL https://cli.dotenv.cloud/install.sh | \
                            DOTENV_INSTALL_DIR=${WORKSPACE}/.dotenv sh
                        '''
                    }

                    // Add to PATH
                    env.PATH = "${DOTENV_CLI_PATH}:${env.PATH}"
                }
            }
        }

        stage('Load Secrets') {
            steps {
                withCredentials([
                    string(credentialsId: 'dotenv-api-key', variable: 'DOTENV_API_KEY')
                ]) {
                    script {
                        def environment = env.BRANCH_NAME == 'main' ? 'production' : 'staging'

                        sh """
                            dotenv auth login --api-key \$DOTENV_API_KEY
                            dotenv pull ${environment} --project ${DOTENV_PROJECT}
                        """

                        // Load secrets into Jenkins environment
                        def secrets = readProperties file: '.env'
                        secrets.each { key, value ->
                            env[key] = value
                            // Mask in console output
                            maskPasswords(varPasswordPairs: [[password: value]])
                        }
                    }
                }
            }
        }

        stage('Build') {
            steps {
                sh '''
                    npm ci
                    npm run build
                '''
            }
        }

        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'npm run test:unit'
                    }
                }

                stage('Integration Tests') {
                    steps {
                        sh 'npm run test:integration'
                    }
                }

                stage('Security Scan') {
                    steps {
                        sh '''
                            npm audit --production
                            trivy fs --severity HIGH,CRITICAL .
                        '''
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                script {
                    docker.withRegistry("https://${env.DOCKER_REGISTRY}", 'docker-credentials') {
                        def app = docker.build("my-app:${env.BUILD_ID}")
                        app.push()
                        app.push('latest')
                    }

                    // Deploy to Kubernetes
                    kubernetesDeploy(
                        configs: 'k8s/*.yaml',
                        kubeconfigId: 'kubeconfig',
                        enableConfigSubstitution: true
                    )
                }
            }
        }
    }

    post {
        always {
            // Clean up secrets
            sh 'rm -f .env'
            cleanWs()
        }
        success {
            slackSend(
                color: 'good',
                message: "Pipeline succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: "Pipeline failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            )
        }
    }
}
```

### 2. Shared Library

```groovy
// vars/dotenvSecrets.groovy
def load(String project, String environment) {
    withCredentials([string(credentialsId: 'dotenv-api-key', variable: 'DOTENV_API_KEY')]) {
        sh """
            if ! command -v dotenv &> /dev/null; then
                curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                export PATH="\$HOME/.dotenv/bin:\$PATH"
            fi

            dotenv auth login --api-key \$DOTENV_API_KEY
            dotenv pull ${environment} --project ${project} --format json > secrets.json
        """

        def secrets = readJSON file: 'secrets.json'
        secrets.each { key, value ->
            env[key] = value
        }

        sh 'rm -f secrets.json'
    }
}

def rotate(String project, String environment, String secretKey) {
    withCredentials([string(credentialsId: 'dotenv-api-key', variable: 'DOTENV_API_KEY')]) {
        sh """
            dotenv auth login --api-key \$DOTENV_API_KEY
            dotenv secret rotate --key ${secretKey} \
                --project ${project} \
                --environment ${environment}
        """
    }
}
```

## CircleCI Integration

### 1. Config

```yaml
# .circleci/config.yml
version: 2.1

orbs:
    dotenv: myorg/dotenv@1.0.0

executors:
    node:
        docker:
            - image: cimg/node:18.0
        environment:
            DOTENV_PROJECT: my-app

commands:
    load-secrets:
        parameters:
            environment:
                type: string
        steps:
            - run:
                  name: Install DotEnv CLI
                  command: |
                      if [ ! -f "$HOME/.dotenv/bin/dotenv" ]; then
                        curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                      fi
                      echo 'export PATH="$HOME/.dotenv/bin:$PATH"' >> $BASH_ENV

            - run:
                  name: Load secrets
                  command: |
                      dotenv auth login --api-key $DOTENV_API_KEY
                      dotenv pull << parameters.environment >> \
                        --project $DOTENV_PROJECT \
                        --format shell > $BASH_ENV
                      source $BASH_ENV

jobs:
    test:
        executor: node
        steps:
            - checkout
            - load-secrets:
                  environment: test
            - run:
                  name: Run tests
                  command: |
                      npm ci
                      npm test
            - store_test_results:
                  path: test-results

    build:
        executor: node
        steps:
            - checkout
            - load-secrets:
                  environment: staging
            - run:
                  name: Build application
                  command: |
                      npm ci
                      npm run build
            - persist_to_workspace:
                  root: .
                  paths:
                      - dist

    deploy:
        executor: node
        steps:
            - checkout
            - attach_workspace:
                  at: .
            - load-secrets:
                  environment: production
            - run:
                  name: Deploy to production
                  command: |
                      npm run deploy:production

workflows:
    version: 2
    build-and-deploy:
        jobs:
            - test:
                  context: dotenv-secrets
            - build:
                  context: dotenv-secrets
                  requires:
                      - test
            - deploy:
                  context: dotenv-secrets
                  requires:
                      - build
                  filters:
                      branches:
                          only: main
```

### 2. Orb Definition

```yaml
# orb.yml
version: 2.1
description: DotEnv integration for CircleCI

commands:
    install:
        description: Install DotEnv CLI
        steps:
            - run:
                  name: Install DotEnv CLI
                  command: |
                      curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                      echo 'export PATH="$HOME/.dotenv/bin:$PATH"' >> $BASH_ENV
                      source $BASH_ENV
                      dotenv --version

    load:
        description: Load secrets from DotEnv
        parameters:
            project:
                type: string
                description: DotEnv project name
            environment:
                type: string
                description: Environment to load
            format:
                type: enum
                enum: ["shell", "json", "yaml", "docker"]
                default: shell
        steps:
            - run:
                  name: Load DotEnv secrets
                  command: |
                      dotenv auth login --api-key $DOTENV_API_KEY
                      dotenv pull << parameters.environment >> \
                        --project << parameters.project >> \
                        --format << parameters.format >> > secrets.env

                      if [ "<< parameters.format >>" = "shell" ]; then
                        cat secrets.env >> $BASH_ENV
                        source $BASH_ENV
                      fi
```

## Security Best Practices

### 1. API Key Management

```javascript
// ci/secure-setup.js
class SecureAPIKeyManager {
    constructor() {
        this.providers = {
            github: this.getGitHubSecret,
            gitlab: this.getGitLabVariable,
            jenkins: this.getJenkinsCredential,
            circleci: this.getCircleCIContext,
        };
    }

    async getAPIKey(provider) {
        const getSecret = this.providers[provider];
        if (!getSecret) {
            throw new Error(`Unsupported CI provider: ${provider}`);
        }

        const apiKey = await getSecret();

        // Validate API key format
        if (!this.isValidAPIKey(apiKey)) {
            throw new Error("Invalid API key format");
        }

        // Check key permissions
        await this.validateKeyPermissions(apiKey);

        return apiKey;
    }

    isValidAPIKey(key) {
        // API keys should match expected format
        return /^dotenv_[a-zA-Z0-9]{32,}$/.test(key);
    }

    async validateKeyPermissions(apiKey) {
        // Verify key has only necessary permissions
        const client = new DotEnvClient({ apiKey });
        const permissions = await client.getKeyPermissions();

        const allowedPermissions = ["secrets:read"];
        const hasExtraPermissions = permissions.some(
            (p) => !allowedPermissions.includes(p),
        );

        if (hasExtraPermissions) {
            console.warn(
                "API key has more permissions than necessary for CI/CD",
            );
        }
    }

    async getGitHubSecret() {
        return process.env.DOTENV_API_KEY;
    }

    async getGitLabVariable() {
        return process.env.DOTENV_API_KEY;
    }

    async getJenkinsCredential() {
        // Jenkins stores in a different way
        return process.env.DOTENV_CREDENTIALS_PSW;
    }

    async getCircleCIContext() {
        return process.env.DOTENV_API_KEY;
    }
}
```

### 2. Secret Isolation

```yaml
# k8s/ci-runner-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
    name: ci-runner-config
data:
    config.toml: |
        [[runners]]
          [runners.kubernetes]
            namespace = "ci-runners"
            image = "alpine:latest"
            
            # Isolate secrets per job
            [runners.kubernetes.pod_security_context]
              run_as_non_root = true
              run_as_user = 1000
              fs_group = 1000
            
            # Clean up after each job
            [runners.kubernetes.pod_annotations]
              "dotenv.cloud/cleanup" = "true"
            
            # Mount secrets as temporary volume
            [[runners.kubernetes.volumes.empty_dir]]
              name = "dotenv-secrets"
              mount_path = "/secrets"
              medium = "Memory"
```

## Monitoring and Auditing

### 1. Pipeline Metrics

```javascript
// monitoring/pipeline-metrics.js
const prometheus = require("prom-client");

const pipelineMetrics = {
    secretLoadDuration: new prometheus.Histogram({
        name: "ci_secret_load_duration_seconds",
        help: "Time to load secrets from DotEnv",
        labelNames: ["pipeline", "environment", "status"],
        buckets: [0.1, 0.5, 1, 2, 5, 10],
    }),

    secretRotations: new prometheus.Counter({
        name: "ci_secret_rotations_total",
        help: "Total number of secret rotations",
        labelNames: ["pipeline", "secret_type"],
    }),

    pipelineRuns: new prometheus.Counter({
        name: "ci_pipeline_runs_total",
        help: "Total number of pipeline runs",
        labelNames: ["pipeline", "branch", "status"],
    }),

    deployments: new prometheus.Counter({
        name: "ci_deployments_total",
        help: "Total number of deployments",
        labelNames: ["environment", "status", "pipeline"],
    }),
};

function recordSecretLoad(pipeline, environment, duration, success) {
    pipelineMetrics.secretLoadDuration.observe(
        {
            pipeline,
            environment,
            status: success ? "success" : "failure",
        },
        duration,
    );
}

function recordPipelineRun(pipeline, branch, status) {
    pipelineMetrics.pipelineRuns.inc({
        pipeline,
        branch,
        status,
    });
}

module.exports = {
    metrics: pipelineMetrics,
    recordSecretLoad,
    recordPipelineRun,
};
```

### 2. Audit Integration

```javascript
// ci/audit-logger.js
class CIAuditLogger {
    async logPipelineAccess(event) {
        const entry = {
            timestamp: new Date().toISOString(),
            pipeline: {
                id: event.pipelineId,
                name: event.pipelineName,
                run: event.runNumber,
            },
            ci_system: event.ciSystem,
            trigger: {
                type: event.triggerType, // manual, schedule, webhook
                user: event.triggerUser,
                branch: event.branch,
            },
            secrets_accessed: event.secretsAccessed,
            environment: event.environment,
            result: event.result,
        };

        // Send to audit system
        await this.sendToAuditLog(entry);

        // Alert on suspicious activity
        if (this.isSuspicious(event)) {
            await this.alertSecurityTeam(entry);
        }
    }

    isSuspicious(event) {
        // Check for unusual patterns
        return (
            (event.triggerType === "manual" &&
                event.environment === "production" &&
                !this.isAuthorizedUser(event.triggerUser)) ||
            event.secretsAccessed.length > 50 || // Unusual number of secrets
            (event.branch !== "main" && event.environment === "production")
        );
    }
}
```

## Best Practices

### 1. Pipeline Security

```yaml
# Security checklist for CI/CD
security_practices:
    api_keys:
        - Use read-only API keys for CI/CD
        - Rotate API keys every 90 days
        - Scope keys to specific projects/environments

    secret_handling:
        - Never echo secrets in logs
        - Clean up secrets after use
        - Use masked variables
        - Implement secret scanning

    access_control:
        - Limit who can modify pipelines
        - Review pipeline changes
        - Audit pipeline executions
        - Use branch protection

    deployment:
        - Require manual approval for production
        - Implement deployment windows
        - Use canary deployments
        - Enable automatic rollback
```

### 2. Performance Optimization

```javascript
// ci/performance.js
class PipelineOptimizer {
    async optimizeSecretLoading(projects) {
        // Parallel secret loading
        const loadPromises = projects.map((project) =>
            this.loadProjectSecrets(project),
        );

        const results = await Promise.allSettled(loadPromises);

        // Cache frequently used secrets
        const cache = new Map();
        results.forEach((result, index) => {
            if (result.status === "fulfilled") {
                cache.set(projects[index], result.value);
            }
        });

        return cache;
    }

    async cacheSecrets(environment, ttl = 3600) {
        const cacheKey = `secrets-${environment}-${Date.now()}`;

        // Store encrypted cache
        const secrets = await this.loadSecrets(environment);
        const encrypted = await this.encrypt(secrets);

        await this.cache.set(cacheKey, encrypted, ttl);

        return cacheKey;
    }
}
```

## Troubleshooting

### Common Issues

1. **API Key Authentication Failures**

    - Verify key is not expired
    - Check key has correct permissions
    - Ensure key is properly masked

2. **Secret Loading Timeouts**

    - Implement retry logic
    - Use regional endpoints
    - Cache secrets when appropriate

3. **Pipeline Permission Errors**
    - Verify service account permissions
    - Check environment restrictions
    - Review audit logs

## Resources

- [CI/CD Best Practices](https://www.atlassian.com/continuous-delivery/principles)
- [Pipeline Security](https://owasp.org/www-project-devsecops-pipeline/)
- [Secret Management in CI/CD](https://www.hashicorp.com/resources/secrets-management-in-ci-cd)
- [DotEnv CLI Documentation](/documentation/v1/cli/overview)

## Next Steps

- [Choose your CI/CD platform](#pipeline-architecture)
- [Set up secure authentication](#security-best-practices)
- [Configure pipeline integration](#github-actions-integration)
- [Enable monitoring](#monitoring-and-auditing)
