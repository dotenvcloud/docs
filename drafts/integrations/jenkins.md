---
title: Jenkins Integration
slug: jenkins
order: 13
tags: [integrations, jenkins, ci-cd, automation]
---

# Jenkins Integration

Integrate DotEnv with Jenkins to manage secrets in your CI/CD pipelines. This guide covers setup, pipeline configuration, and best practices for Jenkins automation.

## Overview

The DotEnv Jenkins integration enables:

- Secure secret injection into pipelines
- Credentials provider integration
- Multi-branch pipeline support
- Job DSL configuration
- Folder-level secret management
- Audit logging and compliance

## Installation

### 1. Jenkins Plugin

Install the DotEnv Jenkins plugin:

```groovy
// In Jenkins Script Console
Jenkins.instance.pluginManager.plugins.each {
  plugin -> println("${plugin.getShortName()}: ${plugin.getVersion()}")
}

// Install via CLI
java -jar jenkins-cli.jar -s http://localhost:8080/ install-plugin dotenv
```

### 2. Global Tool Configuration

Configure DotEnv CLI as a global tool:

1. Navigate to Manage Jenkins → Global Tool Configuration
2. Add DotEnv CLI installation:
    - Name: `dotenv-cli`
    - Install automatically: Yes
    - Installer: Shell command
    - Command: `curl -fsSL https://cli.dotenv.cloud/install.sh | sh`

### 3. Credentials Setup

Add DotEnv API key as Jenkins credential:

```groovy
// Script to add credential
import jenkins.model.Jenkins
import com.cloudbees.plugins.credentials.domains.Domain
import com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl
import com.cloudbees.plugins.credentials.CredentialsScope

def jenkins = Jenkins.getInstance()
def store = jenkins.getExtensionList('com.cloudbees.plugins.credentials.SystemCredentialsProvider')[0].getStore()

def credential = new UsernamePasswordCredentialsImpl(
  CredentialsScope.GLOBAL,
  "dotenv-api-key",
  "DotEnv API Key",
  "api",
  "your-api-key-here"
)

store.addCredentials(Domain.global(), credential)
```

## Pipeline Configuration

### Declarative Pipeline

```groovy
pipeline {
    agent any

    tools {
        dotenv 'dotenv-cli'
    }

    environment {
        DOTENV_PROJECT = 'my-app'
        DOTENV_ENVIRONMENT = "${env.BRANCH_NAME == 'main' ? 'production' : 'staging'}"
    }

    stages {
        stage('Load Secrets') {
            steps {
                withCredentials([string(credentialsId: 'dotenv-api-key', variable: 'DOTENV_API_KEY')]) {
                    sh '''
                        dotenv auth login --api-key $DOTENV_API_KEY
                        dotenv pull $DOTENV_ENVIRONMENT --project $DOTENV_PROJECT
                        source .env
                    '''
                }
            }
        }

        stage('Build') {
            steps {
                sh '''
                    source .env
                    npm install
                    npm run build
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    source .env
                    npm test
                '''
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    source .env
                    npm run deploy
                '''
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

### Scripted Pipeline

```groovy
node {
    def dotenvLoaded = false

    try {
        stage('Checkout') {
            checkout scm
        }

        stage('Load Secrets') {
            withCredentials([string(credentialsId: 'dotenv-api-key', variable: 'DOTENV_API_KEY')]) {
                def environment = env.BRANCH_NAME == 'main' ? 'production' : 'staging'

                sh """
                    curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                    export PATH="\$HOME/.dotenv/bin:\$PATH"

                    dotenv auth login --api-key ${DOTENV_API_KEY}
                    dotenv pull ${environment} --project my-app
                """

                dotenvLoaded = true
            }
        }

        stage('Build') {
            sh '''
                source .env
                npm install
                npm run build
            '''
        }

        stage('Test') {
            sh '''
                source .env
                npm test
            '''
        }

        if (env.BRANCH_NAME == 'main') {
            stage('Deploy') {
                sh '''
                    source .env
                    npm run deploy
                '''
            }
        }
    } finally {
        if (dotenvLoaded) {
            sh 'shred -vfz .env 2>/dev/null || rm -f .env'
        }
    }
}
```

## Shared Libraries

### Global Pipeline Library

Create reusable DotEnv functions:

```groovy
// vars/dotenv.groovy
def loadSecrets(String project, String environment) {
    withCredentials([string(credentialsId: 'dotenv-api-key', variable: 'DOTENV_API_KEY')]) {
        sh """
            if ! command -v dotenv &> /dev/null; then
                curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                echo 'export PATH="\$HOME/.dotenv/bin:\$PATH"' >> ~/.bashrc
                source ~/.bashrc
            fi

            dotenv auth login --api-key \$DOTENV_API_KEY
            dotenv pull ${environment} --project ${project}
        """
    }
}

def withSecrets(String project, String environment, Closure body) {
    try {
        loadSecrets(project, environment)
        sh 'source .env && env > .env.exported'
        def envVars = readProperties file: '.env.exported'
        withEnv(envVars.collect { k, v -> "${k}=${v}" }) {
            body()
        }
    } finally {
        sh 'rm -f .env .env.exported'
    }
}

// Usage in pipeline
@Library('jenkins-shared-library') _

pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                script {
                    dotenv.withSecrets('my-app', 'production') {
                        sh 'npm run build'
                    }
                }
            }
        }
    }
}
```

## Multi-Branch Pipelines

### Jenkinsfile

```groovy
pipeline {
    agent any

    environment {
        DOTENV_PROJECT = "${env.GIT_URL.tokenize('/')[-1].replace('.git', '')}"
        DOTENV_ENVIRONMENT = getBranchEnvironment()
    }

    stages {
        stage('Setup') {
            steps {
                script {
                    // Install DotEnv CLI if needed
                    if (!fileExists("${env.HOME}/.dotenv/bin/dotenv")) {
                        sh 'curl -fsSL https://cli.dotenv.cloud/install.sh | sh'
                    }
                }
            }
        }

        stage('Load Secrets') {
            steps {
                withCredentials([string(credentialsId: "dotenv-api-key-${env.DOTENV_ENVIRONMENT}", variable: 'DOTENV_API_KEY')]) {
                    sh '''
                        export PATH="$HOME/.dotenv/bin:$PATH"
                        dotenv auth login --api-key $DOTENV_API_KEY
                        dotenv pull $DOTENV_ENVIRONMENT --project $DOTENV_PROJECT
                    '''
                }
            }
        }

        stage('Build & Test') {
            parallel {
                stage('Build') {
                    steps {
                        sh '''
                            source .env
                            npm install
                            npm run build
                        '''
                    }
                }

                stage('Lint') {
                    steps {
                        sh '''
                            source .env
                            npm run lint
                        '''
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                anyOf {
                    branch 'main'
                    branch 'staging'
                    branch 'develop'
                }
            }
            steps {
                sh '''
                    source .env
                    npm run deploy:$DOTENV_ENVIRONMENT
                '''
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}

def getBranchEnvironment() {
    switch(env.BRANCH_NAME) {
        case 'main':
            return 'production'
        case 'staging':
            return 'staging'
        case 'develop':
            return 'development'
        default:
            return 'preview'
    }
}
```

## Job DSL

### Automated Job Creation

```groovy
// job-dsl/dotenv-jobs.groovy
def projects = [
    [name: 'frontend', repo: 'github.com/myorg/frontend'],
    [name: 'backend', repo: 'github.com/myorg/backend'],
    [name: 'api', repo: 'github.com/myorg/api']
]

projects.each { project ->
    multibranchPipelineJob(project.name) {
        branchSources {
            git {
                id(project.name)
                remote("https://${project.repo}.git")
                credentialsId('github-credentials')
                includes('main staging develop feature/*')
            }
        }

        configure { node ->
            node / sources / data / 'jenkins.branch.BranchSource' / source / traits {
                'jenkins.plugins.git.traits.BranchDiscoveryTrait'()
            }
        }

        factory {
            workflowBranchProjectFactory {
                scriptPath('Jenkinsfile')
            }
        }

        orphanedItemStrategy {
            discardOldItems {
                numToKeep(10)
            }
        }

        triggers {
            periodic(5)
        }

        properties {
            folderCredentialsProperty {
                domainCredentials {
                    domainCredentials {
                        domain {
                            name(project.name)
                            description("${project.name} credentials")
                        }
                        credentials {
                            usernamePassword {
                                scope('SYSTEM')
                                id("dotenv-api-key-${project.name}")
                                description("DotEnv API key for ${project.name}")
                                username('api')
                                password("${DOTENV_API_KEY_PREFIX}${project.name}")
                            }
                        }
                    }
                }
            }
        }
    }
}
```

## Docker Integration

### Pipeline with Docker

```groovy
pipeline {
    agent {
        docker {
            image 'node:18-alpine'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    environment {
        DOCKER_REGISTRY = 'myregistry.io'
        IMAGE_NAME = "${env.GIT_COMMIT.take(7)}"
    }

    stages {
        stage('Build Docker Image') {
            steps {
                withCredentials([string(credentialsId: 'dotenv-api-key', variable: 'DOTENV_API_KEY')]) {
                    sh '''
                        # Install DotEnv in container
                        apk add --no-cache curl bash
                        curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                        export PATH="$HOME/.dotenv/bin:$PATH"

                        # Get secrets
                        dotenv auth login --api-key $DOTENV_API_KEY
                        dotenv pull production --project my-app --format docker > .env.docker

                        # Build with secrets
                        docker build \
                            --secret id=dotenv,src=.env.docker \
                            -t ${DOCKER_REGISTRY}/my-app:${IMAGE_NAME} \
                            .
                    '''
                }
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-registry',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin $DOCKER_REGISTRY
                        docker push ${DOCKER_REGISTRY}/my-app:${IMAGE_NAME}
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'rm -f .env.docker'
        }
    }
}
```

## Security Best Practices

### 1. Credentials Management

```groovy
// Use folder-level credentials
folder('production') {
    properties {
        folderCredentialsProperty {
            domainCredentials {
                domainCredentials {
                    domain {
                        name('dotenv')
                        description('DotEnv credentials')
                    }
                    credentials {
                        string {
                            scope('SYSTEM')
                            id('dotenv-api-key-prod')
                            description('Production DotEnv API key')
                            secret('${DOTENV_PROD_API_KEY}')
                        }
                    }
                }
            }
        }
    }
}
```

### 2. Audit Logging

```groovy
// Add audit logging to pipeline
def auditLog(String action, String status) {
    def timestamp = new Date().format("yyyy-MM-dd HH:mm:ss")
    def logEntry = "${timestamp} | ${env.BUILD_NUMBER} | ${env.JOB_NAME} | ${action} | ${status} | ${env.BUILD_USER}"

    sh "echo '${logEntry}' >> /var/jenkins_home/dotenv-audit.log"
}

pipeline {
    stages {
        stage('Load Secrets') {
            steps {
                script {
                    try {
                        // Load secrets
                        withCredentials([string(credentialsId: 'dotenv-api-key', variable: 'DOTENV_API_KEY')]) {
                            sh 'dotenv pull production --project my-app'
                        }
                        auditLog('SECRET_LOAD', 'SUCCESS')
                    } catch (Exception e) {
                        auditLog('SECRET_LOAD', 'FAILURE')
                        throw e
                    }
                }
            }
        }
    }
}
```

### 3. Secret Masking

```groovy
// Custom step to mask secrets in logs
def maskSecrets(List<String> secrets) {
    def maskedSecrets = secrets.collect { secret ->
        [password(variable: secret, value: env[secret])]
    }.flatten()

    wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: maskedSecrets]) {
        yield
    }
}

// Usage
stage('Deploy') {
    steps {
        script {
            def secretKeys = sh(
                script: "grep -v '^#' .env | cut -d'=' -f1",
                returnStdout: true
            ).trim().split('\n')

            maskSecrets(secretKeys) {
                sh 'npm run deploy'
            }
        }
    }
}
```

### 4. Node Security

```groovy
// Restrict to secure nodes
pipeline {
    agent {
        label 'secure && docker'
    }

    options {
        // Limit build time
        timeout(time: 30, unit: 'MINUTES')

        // Don't keep secrets in build history
        skipDefaultCheckout()

        // Limit concurrent builds
        disableConcurrentBuilds()
    }
}
```

## Advanced Patterns

### 1. Blue-Green Deployment

```groovy
pipeline {
    parameters {
        choice(
            name: 'DEPLOY_TARGET',
            choices: ['blue', 'green'],
            description: 'Deployment target environment'
        )
    }

    stages {
        stage('Load Secrets') {
            steps {
                script {
                    def targetEnv = "${params.DEPLOY_TARGET}-production"
                    withCredentials([string(credentialsId: 'dotenv-api-key', variable: 'DOTENV_API_KEY')]) {
                        sh """
                            dotenv auth login --api-key \$DOTENV_API_KEY
                            dotenv pull ${targetEnv} --project my-app
                        """
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sh """
                    source .env
                    kubectl set image deployment/my-app-${params.DEPLOY_TARGET} \
                        app=myregistry/my-app:${env.BUILD_NUMBER}
                """
            }
        }

        stage('Health Check') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitUntil {
                        script {
                            def response = sh(
                                script: "curl -s -o /dev/null -w '%{http_code}' http://${params.DEPLOY_TARGET}.example.com/health",
                                returnStdout: true
                            ).trim()
                            return response == '200'
                        }
                    }
                }
            }
        }

        stage('Switch Traffic') {
            input {
                message "Switch traffic to ${params.DEPLOY_TARGET}?"
                ok "Switch"
            }
            steps {
                sh """
                    source .env
                    kubectl patch service my-app -p '{"spec":{"selector":{"version":"${params.DEPLOY_TARGET}"}}}'
                """
            }
        }
    }
}
```

### 2. Matrix Builds

```groovy
pipeline {
    agent none

    stages {
        stage('Matrix Build') {
            matrix {
                agent any
                axes {
                    axis {
                        name 'NODE_VERSION'
                        values '14', '16', '18'
                    }
                    axis {
                        name 'ENVIRONMENT'
                        values 'staging', 'production'
                    }
                }
                excludes {
                    exclude {
                        axis {
                            name 'NODE_VERSION'
                            values '14'
                        }
                        axis {
                            name 'ENVIRONMENT'
                            values 'production'
                        }
                    }
                }
                stages {
                    stage('Test') {
                        steps {
                            withCredentials([string(credentialsId: "dotenv-api-key-${ENVIRONMENT}", variable: 'DOTENV_API_KEY')]) {
                                sh """
                                    docker run --rm \
                                        -e DOTENV_API_KEY=\$DOTENV_API_KEY \
                                        -e DOTENV_PROJECT=my-app \
                                        -e DOTENV_ENVIRONMENT=${ENVIRONMENT} \
                                        node:${NODE_VERSION}-alpine \
                                        sh -c "
                                            apk add --no-cache curl bash
                                            curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                                            export PATH=\\"\\\$HOME/.dotenv/bin:\\\$PATH\\"
                                            dotenv auth login --api-key \\\$DOTENV_API_KEY
                                            dotenv pull \\\$DOTENV_ENVIRONMENT --project \\\$DOTENV_PROJECT
                                            source .env
                                            npm test
                                        "
                                """
                            }
                        }
                    }
                }
            }
        }
    }
}
```

## Monitoring

### Pipeline Metrics

```groovy
// Send metrics to monitoring system
def sendMetrics(String stage, long duration, String status) {
    def metrics = [
        stage: stage,
        duration: duration,
        status: status,
        job: env.JOB_NAME,
        build: env.BUILD_NUMBER
    ]

    httpRequest(
        url: 'http://metrics.example.com/api/v1/jenkins',
        httpMode: 'POST',
        contentType: 'APPLICATION_JSON',
        requestBody: groovy.json.JsonOutput.toJson(metrics)
    )
}

pipeline {
    stages {
        stage('Build') {
            steps {
                script {
                    def startTime = System.currentTimeMillis()
                    try {
                        sh 'npm run build'
                        sendMetrics('build', System.currentTimeMillis() - startTime, 'success')
                    } catch (Exception e) {
                        sendMetrics('build', System.currentTimeMillis() - startTime, 'failure')
                        throw e
                    }
                }
            }
        }
    }
}
```

## Troubleshooting

### Common Issues

#### CLI Installation Failures

```groovy
// Retry logic for CLI installation
def installDotEnvCLI(int maxRetries = 3) {
    def retries = 0
    while (retries < maxRetries) {
        try {
            sh '''
                if ! command -v dotenv &> /dev/null; then
                    curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                    export PATH="$HOME/.dotenv/bin:$PATH"
                    dotenv --version
                fi
            '''
            return
        } catch (Exception e) {
            retries++
            if (retries >= maxRetries) {
                throw new Exception("Failed to install DotEnv CLI after ${maxRetries} attempts")
            }
            sleep(time: 10, unit: 'SECONDS')
        }
    }
}
```

#### Permission Issues

```groovy
// Fix permission issues
stage('Fix Permissions') {
    steps {
        sh '''
            # Fix home directory permissions
            if [ -w "$HOME" ]; then
                chmod 755 "$HOME"
                mkdir -p "$HOME/.dotenv"
                chmod 755 "$HOME/.dotenv"
            else
                # Use workspace directory
                export DOTENV_HOME="$WORKSPACE/.dotenv"
                mkdir -p "$DOTENV_HOME"
                export PATH="$DOTENV_HOME/bin:$PATH"
            fi
        '''
    }
}
```

## Examples

### Complete CI/CD Pipeline

```groovy
@Library('jenkins-shared-library') _

pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: node
    image: node:18-alpine
    command: ['cat']
    tty: true
  - name: docker
    image: docker:dind
    securityContext:
      privileged: true
"""
        }
    }

    environment {
        APP_NAME = 'my-app'
        DOCKER_REGISTRY = 'myregistry.io'
    }

    stages {
        stage('Setup') {
            steps {
                container('node') {
                    checkout scm
                    sh '''
                        apk add --no-cache curl bash git
                        curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                    '''
                }
            }
        }

        stage('Load Secrets') {
            steps {
                container('node') {
                    script {
                        def environment = env.BRANCH_NAME == 'main' ? 'production' : 'staging'
                        dotenv.loadSecrets(env.APP_NAME, environment)
                    }
                }
            }
        }

        stage('Build & Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        container('node') {
                            sh '''
                                source .env
                                npm ci
                                npm run test:unit
                            '''
                        }
                    }
                }

                stage('Integration Tests') {
                    steps {
                        container('node') {
                            sh '''
                                source .env
                                npm run test:integration
                            '''
                        }
                    }
                }

                stage('Build Docker') {
                    steps {
                        container('docker') {
                            sh '''
                                source .env
                                docker build -t ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER} .
                            '''
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                container('node') {
                    sh '''
                        source .env
                        kubectl apply -f k8s/
                        kubectl set image deployment/${APP_NAME} ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER}
                    '''
                }
            }
        }
    }

    post {
        always {
            container('node') {
                sh 'rm -f .env'
            }
            cleanWs()
        }
        success {
            slackSend(
                color: 'good',
                message: "Build #${env.BUILD_NUMBER} succeeded for ${env.JOB_NAME}"
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: "Build #${env.BUILD_NUMBER} failed for ${env.JOB_NAME}"
            )
        }
    }
}
```

## Resources

- [Jenkins Documentation](https://www.jenkins.io/doc/)
- [Pipeline Syntax Reference](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Credentials Plugin](https://plugins.jenkins.io/credentials/)
- [DotEnv CLI Reference](/documentation/v1/cli/commands)

## Next Steps

- [Configure Jenkins credentials](#credentials-setup)
- [Create your first pipeline](#pipeline-configuration)
- [Set up shared libraries](#shared-libraries)
- [Enable security features](#security-best-practices)
