---
title: CI/CD and Containers
slug: ci-cd-and-containers
order: 3
description: Run the DotEnv CLI in a shell step on GitLab CI, Jenkins, CircleCI, and Docker — one correct recipe each, with where to store the read-only key.
tags: [integrations, ci-cd, gitlab, jenkins, circleci, docker, buildkit, cli, secrets]
---

# CI/CD and Containers

Outside GitHub Actions, you integrate DotEnv by running the **CLI in a shell step**. The recipe is
always the same: store a **read-only** API key in the platform's secret store, install the CLI,
then `dotenv pull` into a `.env` file. Below is one concise, correct example for each common
platform, including where each stores the key.

Every example assumes a read-only key created with
[`dotenv apikeys create`](/documentation/cli/commands). CI only reads secrets, so the key must not
be able to modify or delete anything.

## GitLab CI

**Store the key as:** a CI/CD variable, marked **Masked** and **Protected**
(*Settings → CI/CD → Variables*). Name it `DOTENV_API_KEY`.

```yaml
deploy:
  image: ubuntu:latest
  variables:
    DOTENV_API_KEY: $DOTENV_API_KEY   # masked + protected project variable
  script:
    - apt-get update && apt-get install -y curl
    - curl -sSL https://dotenv.cloud/install.sh | bash
    - dotenv pull myapp/production/api --output .env --quiet
    - set -a; . ./.env; set +a
    - ./scripts/deploy.sh
```

Masked keeps the value out of job logs; Protected restricts it to protected branches and tags.

## Jenkins

**Store the key as:** a **Secret text** credential in the Jenkins Credentials store, with ID
`dotenv-api-key`. Bind it with `withCredentials` so it's masked in the console.

```groovy
pipeline {
  agent any
  stages {
    stage('Deploy') {
      steps {
        withCredentials([string(credentialsId: 'dotenv-api-key',
                                variable: 'DOTENV_API_KEY')]) {
          sh '''
            curl -sSL https://dotenv.cloud/install.sh | bash
            dotenv pull myapp/production/api --output .env --quiet
            set -a; . ./.env; set +a
            ./scripts/deploy.sh
          '''
        }
      }
    }
  }
}
```

## CircleCI

**Store the key as:** an environment variable inside a **Context** (*Organization Settings →
Contexts*), or a project environment variable. Name it `DOTENV_API_KEY` and reference the context
from the job.

```yaml
version: 2.1
jobs:
  deploy:
    docker:
      - image: cimg/base:current
    steps:
      - checkout
      - run:
          name: Pull secrets and deploy
          command: |
            curl -sSL https://dotenv.cloud/install.sh | bash
            dotenv pull myapp/production/api --output .env --quiet
            set -a; . ./.env; set +a
            ./scripts/deploy.sh

workflows:
  deploy:
    jobs:
      - deploy:
          context: dotenv   # context that holds DOTENV_API_KEY
```

## Docker (BuildKit secrets)

**Store the key as:** a BuildKit build secret, passed at build time with `--secret`. It is mounted
only for the steps that need it and **never** baked into an image layer or its history.

```dockerfile
# syntax=docker/dockerfile:1
FROM ubuntu:latest

RUN apt-get update && apt-get install -y curl \
 && curl -sSL https://dotenv.cloud/install.sh | bash

# Mount the key only for this step; pull secrets into .env
RUN --mount=type=secret,id=dotenv_api_key \
    DOTENV_API_KEY="$(cat /run/secrets/dotenv_api_key)" \
    dotenv pull myapp/production/api --output .env --quiet
```

Build it, supplying the key from your CI's secret store:

```bash
DOCKER_BUILDKIT=1 docker build \
  --secret id=dotenv_api_key,env=DOTENV_API_KEY \
  -t myapp .
```

Because the secret is mounted, not copied, it does not persist in the final image. Prefer pulling
secrets at **runtime** (entrypoint) over build time when you can, so images stay free of any
environment-specific values.

## Loading the .env

After `dotenv pull`, load the file into the shell so subsequent commands see the variables:

```bash
set -a; . ./.env; set +a
```

Or hand `.env` to a tool that reads it directly (a framework, `docker compose`, a test runner).

## Always

- **Read-only keys only** — a leaked read-only key can't change or delete your secrets.
- **Keep `.env` ephemeral** — don't cache it or upload it as an artifact.
- **Gitignore `.env`** so a checkout in CI can never commit it.

## Where to go next

- [GitHub Actions](/documentation/integrations/github-actions) — the official action for GitHub.
- [Integrations overview](/documentation/integrations/overview) — Action vs CLI-in-shell.
- [CI/CD pipelines](/documentation/use-cases/ci-cd) — the general pattern and why read-only.
- [Pulling and pushing](/documentation/cli/pulling-and-pushing) — formats, merge, and flags.
