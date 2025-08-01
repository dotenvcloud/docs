---
title: Installation Guide
slug: installation
order: 2
tags: [installation, setup, cli, sdk]
---

# Installation Guide

DotEnv provides multiple installation methods to fit your development workflow. Choose the approach that best suits your needs.

## CLI Installation

The DotEnv CLI is the primary tool for managing secrets across all platforms.

### System Requirements

- **macOS**: 10.15 or later
- **Linux**: Any modern distribution (Ubuntu 18.04+, CentOS 7+, etc.)
- **Windows**: Windows 10 version 1903 or later
- **Architecture**: x64, ARM64

### Recommended Installation

#### macOS and Linux

Using our installation script (recommended):

```bash
curl -sSL https://dotenv.cloud/install.sh | sh
```

Using Homebrew (macOS):

```bash
brew tap dotenv/tap
brew install dotenv-cli
```

#### Windows

Using PowerShell (Run as Administrator):

```powershell
iwr -useb https://dotenv.cloud/install.ps1 | iex
```

Using Chocolatey:

```powershell
choco install dotenv-cli
```

Using Scoop:

```powershell
scoop bucket add dotenv https://github.com/dotenv/scoop-bucket
scoop install dotenv-cli
```

### Package Managers

#### npm/yarn/pnpm

```bash
# npm
npm install -g @dotenv/cli

# yarn
yarn global add @dotenv/cli

# pnpm
pnpm add -g @dotenv/cli
```

#### Go

```bash
go install github.com/dotenv/cli@latest
```

### Binary Installation

Download pre-compiled binaries from our [releases page](https://github.com/dotenv/cli/releases):

```bash
# Linux/macOS example
wget https://github.com/dotenv/cli/releases/latest/download/dotenv-linux-amd64.tar.gz
tar -xzf dotenv-linux-amd64.tar.gz
sudo mv dotenv /usr/local/bin/
```

### Verify Installation

After installation, verify the CLI is working:

```bash
dotenv --version
# Output: dotenv version 1.0.0
```

## SDK Installation

### JavaScript/TypeScript

```bash
# npm
npm install @dotenv/sdk

# yarn
yarn add @dotenv/sdk

# pnpm
pnpm add @dotenv/sdk
```

**TypeScript types are included.**

### Python

```bash
pip install dotenv-sdk

# or using poetry
poetry add dotenv-sdk

# or using pipenv
pipenv install dotenv-sdk
```

### Go

```bash
go get github.com/dotenv/sdk-go
```

### PHP

```bash
composer require dotenv/sdk
```

### Ruby

```bash
gem install dotenv-sdk

# or add to Gemfile
gem 'dotenv-sdk'
```

### Java

Maven:

```xml
<dependency>
    <groupId>com.dotenv</groupId>
    <artifactId>sdk</artifactId>
    <version>1.0.0</version>
</dependency>
```

Gradle:

```gradle
implementation 'com.dotenv:sdk:1.0.0'
```

### .NET

```bash
dotnet add package DotEnv.SDK

# or using Package Manager
Install-Package DotEnv.SDK
```

## GitHub Action

Add to your workflow:

```yaml
- name: Load DotEnv Secrets
  uses: dotenv/actions@v1
  with:
      api-key: ${{ secrets.DOTENV_API_KEY }}
      project: my-app
      environment: ${{ github.ref == 'refs/heads/main' && 'production' || 'development' }}
```

## Docker Integration

### Using CLI in Dockerfile

```dockerfile
FROM alpine:latest

# Install DotEnv CLI
RUN apk add --no-cache curl bash && \
    curl -sSL https://dotenv.cloud/install.sh | sh

# Your application setup
WORKDIR /app
COPY . .

# Load secrets at runtime
ENTRYPOINT ["sh", "-c", "dotenv secrets pull > .env && exec $@"]
CMD ["./your-app"]
```

### Using Multi-stage Build

```dockerfile
# Build stage
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

# Runtime stage
FROM alpine:latest
RUN apk add --no-cache ca-certificates

# Install DotEnv CLI
RUN wget -O- https://dotenv.cloud/install.sh | sh

WORKDIR /app
COPY --from=builder /app/myapp .

# Load secrets and run
CMD ["sh", "-c", "dotenv secrets pull --format export > .env && source .env && ./myapp"]
```

## CI/CD Installation

### GitHub Actions

```yaml
name: Deploy
on: [push]

jobs:
    deploy:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v3

            - name: Install DotEnv CLI
              run: curl -sSL https://dotenv.cloud/install.sh | sh

            - name: Load Secrets
              env:
                  DOTENV_API_KEY: ${{ secrets.DOTENV_API_KEY }}
              run: |
                  dotenv secrets pull \
                    --project my-app \
                    --environment production \
                    > .env
```

### GitLab CI

```yaml
deploy:
    image: alpine:latest
    before_script:
        - apk add --no-cache curl bash
        - curl -sSL https://dotenv.cloud/install.sh | sh
    script:
        - dotenv secrets pull --project $CI_PROJECT_NAME > .env
        - source .env
        - ./deploy.sh
```

### Jenkins

```groovy
pipeline {
    agent any

    environment {
        DOTENV_API_KEY = credentials('dotenv-api-key')
    }

    stages {
        stage('Setup') {
            steps {
                sh 'curl -sSL https://dotenv.cloud/install.sh | sh'
            }
        }

        stage('Load Secrets') {
            steps {
                sh 'dotenv secrets pull --project my-app > .env'
            }
        }
    }
}
```

## Platform-Specific Notes

### AWS Lambda

Install as a Lambda Layer:

```bash
# Create layer package
mkdir -p layer/bin
cd layer
curl -sSL https://dotenv.cloud/install.sh | INSTALL_DIR=./bin sh
zip -r dotenv-layer.zip .

# Upload to AWS
aws lambda publish-layer-version \
  --layer-name dotenv-cli \
  --zip-file fileb://dotenv-layer.zip
```

### Kubernetes

As an init container:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: my-app
spec:
    template:
        spec:
            initContainers:
                - name: dotenv-init
                  image: alpine:latest
                  command: ["/bin/sh", "-c"]
                  args:
                      - |
                          apk add --no-cache curl
                          curl -sSL https://dotenv.cloud/install.sh | sh
                          dotenv secrets pull --project my-app > /shared/.env
                  env:
                      - name: DOTENV_API_KEY
                        valueFrom:
                            secretKeyRef:
                                name: dotenv-credentials
                                key: api-key
                  volumeMounts:
                      - name: env-volume
                        mountPath: /shared
            containers:
                - name: app
                  image: my-app:latest
                  volumeMounts:
                      - name: env-volume
                        mountPath: /app/.env
                        subPath: .env
            volumes:
                - name: env-volume
                  emptyDir: {}
```

## Troubleshooting Installation

### Permission Denied

On Unix systems, you may need to use sudo:

```bash
curl -sSL https://dotenv.cloud/install.sh | sudo sh
```

### SSL Certificate Issues

If you encounter SSL errors:

```bash
# Temporarily bypass SSL (not recommended for production)
curl -sSLk https://dotenv.cloud/install.sh | sh

# Better: Update certificates
# macOS
brew install ca-certificates

# Ubuntu/Debian
sudo apt-get update && sudo apt-get install ca-certificates

# CentOS/RHEL
sudo yum install ca-certificates
```

### Path Issues

If `dotenv` command is not found after installation:

```bash
# Add to PATH manually
echo 'export PATH="$HOME/.dotenv/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Or for zsh
echo 'export PATH="$HOME/.dotenv/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### Proxy Configuration

For corporate environments:

```bash
# Set proxy before installation
export HTTP_PROXY=http://proxy.company.com:8080
export HTTPS_PROXY=http://proxy.company.com:8080

# Install with proxy
curl -sSL https://dotenv.cloud/install.sh | sh

# Configure CLI to use proxy
dotenv config set proxy http://proxy.company.com:8080
```

## Uninstallation

### CLI Removal

```bash
# If installed via script
rm -rf ~/.dotenv
# Remove from PATH in .bashrc/.zshrc

# If installed via Homebrew
brew uninstall dotenv-cli

# If installed via npm
npm uninstall -g @dotenv/cli
```

### SDK Removal

```bash
# JavaScript
npm uninstall @dotenv/sdk

# Python
pip uninstall dotenv-sdk

# Go
go clean -i github.com/dotenv/sdk-go
```

## Next Steps

- [Quick Start Guide](./quick-start) - Get up and running quickly
- [First Project Setup](./first-project) - Create and configure your first project
- [Authentication](./authentication) - Set up API keys and authentication
- [CLI Commands](/documentation/v1/cli/commands) - Explore available commands
