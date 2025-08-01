---
title: Heroku Integration
slug: heroku
order: 9
tags: [integrations, heroku, paas, deployment]
---

# Heroku Integration

Integrate DotEnv with Heroku to manage secrets for your cloud applications. This guide covers setup, deployment pipelines, and best practices for Heroku apps.

## Overview

The DotEnv Heroku integration provides:

- Automatic config var synchronization
- Pipeline-based secret management
- Review app configuration
- Multi-environment support
- Dyno formation secrets
- Add-on integration

## Installation

### 1. Heroku Buildpack

Add the DotEnv buildpack:

```bash
heroku buildpacks:add --index 1 https://github.com/dotenv/heroku-buildpack-dotenv
```

Configure buildpack:

```bash
heroku config:set DOTENV_API_KEY=your-api-key
heroku config:set DOTENV_PROJECT=my-app
heroku config:set DOTENV_ENVIRONMENT=production
```

### 2. Manual CLI Setup

Install during build:

```json
// package.json
{
    "scripts": {
        "heroku-prebuild": "curl -fsSL https://cli.dotenv.cloud/install.sh | sh",
        "build": "dotenv pull $DOTENV_ENVIRONMENT --project $DOTENV_PROJECT && npm run compile"
    }
}
```

### 3. Release Phase

Configure in `Procfile`:

```procfile
release: ./scripts/load-secrets.sh
web: node index.js
worker: node worker.js
```

`scripts/load-secrets.sh`:

```bash
#!/bin/bash
export PATH="$HOME/.dotenv/bin:$PATH"
dotenv auth login --api-key $DOTENV_API_KEY
dotenv pull $DOTENV_ENVIRONMENT --project $DOTENV_PROJECT
```

## Configuration

### Basic Setup

Configure app settings:

```bash
# Set DotEnv configuration
heroku config:set DOTENV_API_KEY=your-api-key
heroku config:set DOTENV_PROJECT=my-app
heroku config:set DOTENV_ENVIRONMENT=production

# Enable buildpack
heroku buildpacks:add https://github.com/dotenv/heroku-buildpack-dotenv
```

### Environment-Based Configuration

Different environments for different apps:

```bash
# Production app
heroku config:set DOTENV_ENVIRONMENT=production --app my-app-prod

# Staging app
heroku config:set DOTENV_ENVIRONMENT=staging --app my-app-staging

# Development app
heroku config:set DOTENV_ENVIRONMENT=development --app my-app-dev
```

## Usage Patterns

### 1. Node.js Applications

Configure for Node.js:

```javascript
// config/dotenv.js
const { execSync } = require("child_process");

function loadSecrets() {
    if (!process.env.DOTENV_API_KEY) {
        console.log("No DotEnv API key, using Heroku config vars");
        return;
    }

    try {
        // Load secrets from DotEnv
        execSync(
            `dotenv pull ${process.env.DOTENV_ENVIRONMENT} --project ${process.env.DOTENV_PROJECT}`,
            { stdio: "inherit" },
        );

        // Load into process.env
        require("dotenv").config();
    } catch (error) {
        console.error("Failed to load DotEnv secrets:", error);
        // Fall back to Heroku config vars
    }
}

// Load on startup
loadSecrets();

// index.js
require("./config/dotenv");
const express = require("express");
const app = express();

app.get("/", (req, res) => {
    res.json({
        environment: process.env.NODE_ENV,
        hasDatabase: !!process.env.DATABASE_URL,
    });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT);
```

### 2. Ruby on Rails Applications

Configure for Rails:

```ruby
# config/initializers/dotenv.rb
if ENV['DOTENV_API_KEY']
  require 'open3'

  # Install CLI if needed
  unless system('which dotenv > /dev/null 2>&1')
    system('curl -fsSL https://cli.dotenv.cloud/install.sh | sh')
    ENV['PATH'] = "#{ENV['HOME']}/.dotenv/bin:#{ENV['PATH']}"
  end

  # Load secrets
  project = ENV['DOTENV_PROJECT'] || Rails.application.class.module_parent_name.underscore
  environment = ENV['DOTENV_ENVIRONMENT'] || Rails.env

  stdout, stderr, status = Open3.capture3(
    "dotenv pull #{environment} --project #{project}"
  )

  if status.success?
    # Parse and set environment variables
    File.read('.env').each_line do |line|
      next if line.strip.empty? || line.start_with?('#')
      key, value = line.strip.split('=', 2)
      ENV[key] = value.gsub(/^["']|["']$/, '') if key && value
    end
  else
    Rails.logger.error "Failed to load DotEnv secrets: #{stderr}"
  end
end
```

### 3. Python Applications

Configure for Python:

```python
# dotenv_loader.py
import os
import subprocess
import sys

def load_dotenv_secrets():
    """Load secrets from DotEnv if available."""
    api_key = os.environ.get('DOTENV_API_KEY')
    if not api_key:
        print("No DotEnv API key found, using Heroku config vars")
        return

    project = os.environ.get('DOTENV_PROJECT', 'my-app')
    environment = os.environ.get('DOTENV_ENVIRONMENT', 'production')

    try:
        # Install CLI if needed
        subprocess.run(
            "which dotenv || curl -fsSL https://cli.dotenv.cloud/install.sh | sh",
            shell=True,
            check=True
        )

        # Add to PATH
        os.environ['PATH'] = f"{os.environ['HOME']}/.dotenv/bin:{os.environ['PATH']}"

        # Load secrets
        subprocess.run(
            f"dotenv pull {environment} --project {project}",
            shell=True,
            check=True
        )

        # Load .env file
        if os.path.exists('.env'):
            with open('.env') as f:
                for line in f:
                    if '=' in line and not line.startswith('#'):
                        key, value = line.strip().split('=', 1)
                        os.environ[key] = value.strip('"\'')

    except subprocess.CalledProcessError as e:
        print(f"Failed to load DotEnv secrets: {e}")
        sys.exit(1)

# app.py
from dotenv_loader import load_dotenv_secrets
load_dotenv_secrets()

from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return {
        'database': bool(os.environ.get('DATABASE_URL')),
        'redis': bool(os.environ.get('REDIS_URL'))
    }
```

### 4. Java Applications

Configure for Java:

```java
// DotEnvLoader.java
import java.io.*;
import java.util.Properties;

public class DotEnvLoader {
    public static void load() {
        String apiKey = System.getenv("DOTENV_API_KEY");
        if (apiKey == null) {
            System.out.println("No DotEnv API key, using Heroku config");
            return;
        }

        try {
            // Install CLI if needed
            Process checkDotenv = Runtime.getRuntime().exec("which dotenv");
            if (checkDotenv.waitFor() != 0) {
                Runtime.getRuntime().exec(
                    "curl -fsSL https://cli.dotenv.cloud/install.sh | sh"
                ).waitFor();
            }

            // Load secrets
            String project = System.getenv().getOrDefault("DOTENV_PROJECT", "my-app");
            String environment = System.getenv().getOrDefault("DOTENV_ENVIRONMENT", "production");

            ProcessBuilder pb = new ProcessBuilder(
                "sh", "-c",
                String.format("dotenv pull %s --project %s", environment, project)
            );
            pb.environment().put("PATH",
                System.getenv("HOME") + "/.dotenv/bin:" + System.getenv("PATH"));

            Process process = pb.start();
            if (process.waitFor() == 0) {
                loadEnvFile();
            }
        } catch (Exception e) {
            System.err.println("Failed to load DotEnv: " + e.getMessage());
        }
    }

    private static void loadEnvFile() throws IOException {
        File envFile = new File(".env");
        if (envFile.exists()) {
            Properties props = new Properties();
            props.load(new FileInputStream(envFile));
            props.forEach((key, value) ->
                System.setProperty(key.toString(), value.toString())
            );
        }
    }
}
```

## Heroku Pipelines

### Pipeline Configuration

Set up pipeline-based deployments:

```bash
# Create pipeline
heroku pipelines:create my-app-pipeline

# Add apps to pipeline
heroku pipelines:add my-app-pipeline --app my-app-dev --stage development
heroku pipelines:add my-app-pipeline --app my-app-staging --stage staging
heroku pipelines:add my-app-pipeline --app my-app-prod --stage production

# Configure each stage
heroku config:set DOTENV_ENVIRONMENT=development --app my-app-dev
heroku config:set DOTENV_ENVIRONMENT=staging --app my-app-staging
heroku config:set DOTENV_ENVIRONMENT=production --app my-app-prod
```

### Promote Between Stages

```bash
# Promote from staging to production
heroku pipelines:promote --app my-app-staging

# The production app will use its own DOTENV_ENVIRONMENT
```

## Review Apps

### Automatic Review App Configuration

Configure in `app.json`:

```json
{
    "name": "my-app",
    "scripts": {
        "postdeploy": "./scripts/setup-review-app.sh"
    },
    "env": {
        "DOTENV_API_KEY": {
            "required": true
        },
        "DOTENV_PROJECT": {
            "value": "my-app"
        },
        "DOTENV_ENVIRONMENT": {
            "generator": "secret"
        }
    },
    "formation": {
        "web": {
            "quantity": 1,
            "size": "standard-1x"
        }
    },
    "buildpacks": [
        {
            "url": "https://github.com/dotenv/heroku-buildpack-dotenv"
        },
        {
            "url": "heroku/nodejs"
        }
    ]
}
```

`scripts/setup-review-app.sh`:

```bash
#!/bin/bash
set -e

# Generate unique environment name
PR_NUMBER=$(echo $HEROKU_PR_NUMBER || echo "manual")
REVIEW_ENV="review-pr-${PR_NUMBER}"

echo "Setting up review app environment: $REVIEW_ENV"

# Install DotEnv CLI
curl -fsSL https://cli.dotenv.cloud/install.sh | sh
export PATH="$HOME/.dotenv/bin:$PATH"

# Create review environment from staging
dotenv auth login --api-key $DOTENV_API_KEY
dotenv env create $REVIEW_ENV --from staging --project $DOTENV_PROJECT || true

# Update environment variable
heroku config:set DOTENV_ENVIRONMENT=$REVIEW_ENV

# Load secrets
dotenv pull $REVIEW_ENV --project $DOTENV_PROJECT

echo "Review app setup complete"
```

### Review App Cleanup

```javascript
// scripts/cleanup-review-app.js
const { execSync } = require("child_process");

const reviewEnv = process.env.DOTENV_ENVIRONMENT;
if (reviewEnv && reviewEnv.startsWith("review-")) {
    console.log(`Cleaning up environment: ${reviewEnv}`);

    try {
        execSync(
            `dotenv env delete ${reviewEnv} --project ${process.env.DOTENV_PROJECT} --force`,
            { stdio: "inherit" },
        );
    } catch (error) {
        console.error("Failed to delete review environment:", error);
    }
}
```

## Add-on Integration

### Database Add-ons

Sync database credentials:

```javascript
// scripts/sync-addons.js
const { execSync } = require("child_process");

function syncDatabaseUrl() {
    // Get Heroku DATABASE_URL
    const dbUrl = process.env.DATABASE_URL;
    if (!dbUrl) return;

    // Parse and sync to DotEnv
    const url = new URL(dbUrl);
    const secrets = {
        DB_HOST: url.hostname,
        DB_PORT: url.port || "5432",
        DB_NAME: url.pathname.slice(1),
        DB_USER: url.username,
        DB_PASSWORD: url.password,
        DATABASE_URL: dbUrl,
    };

    Object.entries(secrets).forEach(([key, value]) => {
        execSync(
            `dotenv set ${key}="${value}" --project ${process.env.DOTENV_PROJECT} --environment ${process.env.DOTENV_ENVIRONMENT}`,
            { stdio: "inherit" },
        );
    });
}

// Run on release
syncDatabaseUrl();
```

### Redis Add-on

```bash
# Sync Redis URL
heroku addons:create heroku-redis:hobby-dev
REDIS_URL=$(heroku config:get REDIS_URL)
dotenv set REDIS_URL="$REDIS_URL" --project my-app --environment production
```

## Security Best Practices

### 1. API Key Management

Store API key securely:

```bash
# Use different API keys per environment
heroku config:set DOTENV_API_KEY=$DOTENV_PROD_KEY --app my-app-prod
heroku config:set DOTENV_API_KEY=$DOTENV_DEV_KEY --app my-app-dev

# Rotate API keys regularly
dotenv auth rotate-key --force
heroku config:set DOTENV_API_KEY=$NEW_API_KEY
```

### 2. Build-Time Isolation

Keep secrets out of build logs:

```javascript
// scripts/secure-build.js
const maskSecrets = (str) => {
    const secrets = [
        process.env.DATABASE_URL,
        process.env.API_KEY,
        process.env.JWT_SECRET,
    ].filter(Boolean);

    let masked = str;
    secrets.forEach((secret) => {
        if (secret.length > 4) {
            masked = masked.replace(secret, "[REDACTED]");
        }
    });

    return masked;
};

// Override console.log during build
const originalLog = console.log;
console.log = (...args) => {
    originalLog(
        ...args.map((arg) =>
            typeof arg === "string" ? maskSecrets(arg) : arg,
        ),
    );
};
```

### 3. Dyno Security

Secure dyno configuration:

```json
{
    "stack": "heroku-22",
    "formation": {
        "web": {
            "quantity": 1,
            "size": "standard-1x",
            "command": "node --max-old-space-size=2048 index.js"
        }
    },
    "features": ["runtime-dyno-metadata", "http-session-affinity"]
}
```

### 4. Network Security

Configure private spaces:

```bash
# Create private space
heroku spaces:create my-space --region virginia

# Create app in private space
heroku create my-app-private --space my-space

# Configure internal routing
heroku features:enable internal-routing --app my-app-private
```

## Performance Optimization

### 1. Boot Time Optimization

Cache DotEnv CLI:

```dockerfile
# Dockerfile for container deploys
FROM node:18-alpine

# Install DotEnv CLI
RUN curl -fsSL https://cli.dotenv.cloud/install.sh | sh
ENV PATH="/root/.dotenv/bin:$PATH"

# Cache CLI binary
RUN dotenv --version

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY . .

CMD ["node", "index.js"]
```

### 2. Memory Management

Optimize for Heroku dynos:

```javascript
// config/memory.js
const dynoSize = process.env.DYNO_SIZE || "standard-1x";
const memoryLimits = {
    "standard-1x": 512,
    "standard-2x": 1024,
    "performance-m": 2560,
    "performance-l": 14336,
};

const maxMemory = memoryLimits[dynoSize] || 512;
const v8 = require("v8");

// Set heap limit to 75% of dyno memory
v8.setFlagsFromString(`--max-old-space-size=${Math.floor(maxMemory * 0.75)}`);

console.log(`Memory limit set to ${maxMemory}MB for ${dynoSize} dyno`);
```

### 3. Connection Pooling

Manage database connections:

```javascript
// config/database.js
const { Pool } = require("pg");

// Parse DATABASE_URL
const connectionString = process.env.DATABASE_URL;

// Heroku Postgres connection limits
const dynoConnections = {
    "hobby-dev": 20,
    "standard-0": 120,
    "premium-0": 500,
};

const pool = new Pool({
    connectionString,
    ssl: { rejectUnauthorized: false },
    max: process.env.DB_POOL_SIZE || 20,
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 2000,
});

module.exports = pool;
```

## Monitoring

### 1. Heroku Metrics

Track secret loading:

```javascript
// metrics/dotenv.js
const StatsD = require("node-statsd");
const client = new StatsD({
    host: process.env.STATSD_HOST || "localhost",
    port: 8125,
    prefix: "dotenv.",
});

function trackSecretLoad(success, duration) {
    client.timing("secret_load_time", duration);
    client.increment(success ? "secret_load_success" : "secret_load_failure");

    if (!success) {
        console.error("DotEnv secret load failed");
        // Fall back to Heroku config vars
    }
}

module.exports = { trackSecretLoad };
```

### 2. Logging

Structured logging:

```javascript
// config/logger.js
const winston = require("winston");

const logger = winston.createLogger({
    level: process.env.LOG_LEVEL || "info",
    format: winston.format.json(),
    defaultMeta: {
        app: process.env.HEROKU_APP_NAME,
        dyno: process.env.DYNO,
        release: process.env.HEROKU_RELEASE_VERSION,
    },
    transports: [
        new winston.transports.Console({
            format: winston.format.simple(),
        }),
    ],
});

// Log secret loading
logger.info("Loading secrets from DotEnv", {
    project: process.env.DOTENV_PROJECT,
    environment: process.env.DOTENV_ENVIRONMENT,
});

module.exports = logger;
```

## Troubleshooting

### Common Issues

#### Buildpack Not Running

```bash
# Check buildpack order
heroku buildpacks

# DotEnv buildpack should be first
heroku buildpacks:add --index 1 https://github.com/dotenv/heroku-buildpack-dotenv

# Clear build cache
heroku builds:cache:purge
```

#### Secrets Not Loading

Debug script:

```bash
#!/bin/bash
# debug-secrets.sh
echo "=== DotEnv Debug ==="
echo "API Key present: $([ -n "$DOTENV_API_KEY" ] && echo "Yes" || echo "No")"
echo "Project: $DOTENV_PROJECT"
echo "Environment: $DOTENV_ENVIRONMENT"
echo "PATH: $PATH"
echo "DotEnv CLI: $(which dotenv || echo "Not found")"

# Test API connection
dotenv auth status || echo "Auth failed"
dotenv project list || echo "Cannot list projects"
```

#### Memory Issues

```javascript
// Monitor memory usage
setInterval(() => {
    const used = process.memoryUsage();
    console.log("Memory Usage:", {
        rss: `${Math.round(used.rss / 1024 / 1024)}MB`,
        heapTotal: `${Math.round(used.heapTotal / 1024 / 1024)}MB`,
        heapUsed: `${Math.round(used.heapUsed / 1024 / 1024)}MB`,
        external: `${Math.round(used.external / 1024 / 1024)}MB`,
    });
}, 30000);
```

## Examples

### Complete Node.js App

```json
// package.json
{
    "name": "my-heroku-app",
    "version": "1.0.0",
    "engines": {
        "node": "18.x",
        "npm": "9.x"
    },
    "scripts": {
        "start": "node index.js",
        "heroku-prebuild": "npm run install-dotenv",
        "build": "npm run load-secrets && npm run compile",
        "install-dotenv": "curl -fsSL https://cli.dotenv.cloud/install.sh | sh",
        "load-secrets": "dotenv pull $DOTENV_ENVIRONMENT --project $DOTENV_PROJECT || echo 'Using Heroku config'",
        "compile": "webpack --mode production"
    },
    "dependencies": {
        "express": "^4.18.0",
        "pg": "^8.11.0"
    }
}
```

```javascript
// index.js
require("dotenv").config();
const express = require("express");
const { Pool } = require("pg");

const app = express();
const pool = new Pool({
    connectionString: process.env.DATABASE_URL,
    ssl: { rejectUnauthorized: false },
});

app.get("/", async (req, res) => {
    try {
        const result = await pool.query("SELECT NOW()");
        res.json({
            status: "healthy",
            time: result.rows[0].now,
            environment: process.env.NODE_ENV,
        });
    } catch (error) {
        res.status(500).json({ error: "Database connection failed" });
    }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
```

## Resources

- [Heroku Dev Center](https://devcenter.heroku.com/)
- [Buildpack API](https://devcenter.heroku.com/articles/buildpack-api)
- [Config Vars](https://devcenter.heroku.com/articles/config-vars)
- [DotEnv CLI Reference](/documentation/v1/cli/commands)

## Next Steps

- [Add DotEnv buildpack](#heroku-buildpack)
- [Configure pipeline stages](#pipeline-configuration)
- [Set up review apps](#review-apps)
- [Enable monitoring](#monitoring)
