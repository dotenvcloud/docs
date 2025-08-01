---
title: Vercel Integration
slug: vercel
order: 7
tags: [integrations, vercel, serverless, deployment]
---

# Vercel Integration

Integrate DotEnv with Vercel to manage secrets for your serverless applications and static sites. This guide covers setup, deployment workflows, and best practices.

## Overview

The DotEnv Vercel integration enables:

- Automatic environment variable synchronization
- Preview deployment secrets
- Production secret management
- Multi-project support
- Secure secret rotation

## Installation

### 1. Vercel Integration (Recommended)

Install via Vercel Marketplace:

1. Visit [Vercel Integrations Marketplace](https://vercel.com/integrations)
2. Search for "DotEnv"
3. Click "Add Integration"
4. Authorize DotEnv access
5. Select projects to integrate

### 2. Manual Configuration

Configure using Vercel CLI:

```bash
# Install Vercel CLI
npm i -g vercel

# Login to Vercel
vercel login

# Add DotEnv API key as environment variable
vercel env add DOTENV_API_KEY production
# Enter your DotEnv API key when prompted
```

## Configuration

### Automatic Sync

Configure automatic synchronization in `vercel.json`:

```json
{
    "build": {
        "env": {
            "DOTENV_PROJECT": "my-app",
            "DOTENV_ENVIRONMENT": "production"
        }
    },
    "functions": {
        "api/*.js": {
            "maxDuration": 10
        }
    },
    "buildCommand": "npm run build:vercel"
}
```

Build script in `package.json`:

```json
{
    "scripts": {
        "build:vercel": "node scripts/load-dotenv.js && next build",
        "load-secrets": "dotenv pull $DOTENV_ENVIRONMENT --project $DOTENV_PROJECT"
    }
}
```

`scripts/load-dotenv.js`:

```javascript
const { execSync } = require("child_process");

// Install DotEnv CLI if not present
try {
    execSync("dotenv --version", { stdio: "ignore" });
} catch {
    console.log("Installing DotEnv CLI...");
    execSync("npm install -g @dotenv/cli", { stdio: "inherit" });
}

// Load secrets
const project = process.env.DOTENV_PROJECT;
const environment = process.env.DOTENV_ENVIRONMENT || "production";

console.log(`Loading ${environment} secrets for ${project}...`);
execSync(
    `dotenv auth login --api-key ${process.env.DOTENV_API_KEY} && ` +
        `dotenv pull ${environment} --project ${project}`,
    { stdio: "inherit" },
);
```

### Environment-Based Configuration

Different secrets for different deployment types:

```javascript
// scripts/vercel-env.js
const { execSync } = require("child_process");

function getEnvironment() {
    // Vercel environment variables
    const VERCEL_ENV = process.env.VERCEL_ENV;
    const VERCEL_GIT_COMMIT_REF = process.env.VERCEL_GIT_COMMIT_REF;

    if (VERCEL_ENV === "production") {
        return "production";
    } else if (VERCEL_ENV === "preview") {
        // Use branch name for preview deployments
        if (VERCEL_GIT_COMMIT_REF === "staging") {
            return "staging";
        }
        return "preview";
    }
    return "development";
}

const environment = getEnvironment();
const project = process.env.DOTENV_PROJECT || "my-app";

console.log(`Detected Vercel environment: ${environment}`);

// Load appropriate secrets
execSync(`dotenv pull ${environment} --project ${project}`, {
    stdio: "inherit",
});
```

## Usage Patterns

### 1. Next.js Applications

Configure for Next.js apps:

```javascript
// next.config.js
module.exports = {
    env: {
        // Public environment variables
        NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL,
        NEXT_PUBLIC_APP_NAME: process.env.NEXT_PUBLIC_APP_NAME,
    },
    // Server-side environment variables are automatically available
};
```

Load secrets during build:

```json
{
    "scripts": {
        "dev": "dotenv pull development && next dev",
        "build": "dotenv pull production && next build",
        "start": "next start"
    }
}
```

### 2. API Routes

Use secrets in API routes:

```javascript
// pages/api/data.js or app/api/data/route.js
export default async function handler(req, res) {
    // Secrets are available in process.env
    const apiKey = process.env.EXTERNAL_API_KEY;
    const databaseUrl = process.env.DATABASE_URL;

    // Use secrets
    const response = await fetch("https://api.external.com/data", {
        headers: {
            Authorization: `Bearer ${apiKey}`,
        },
    });

    const data = await response.json();
    res.status(200).json(data);
}
```

### 3. Edge Functions

Configure for Vercel Edge Functions:

```javascript
// api/edge-function.js
export const config = {
    runtime: "edge",
};

export default async function handler(request) {
    // Access secrets in edge runtime
    const secret = process.env.EDGE_SECRET;

    return new Response(
        JSON.stringify({
            message: "Edge function response",
            hasSecret: !!secret,
        }),
        {
            status: 200,
            headers: {
                "content-type": "application/json",
            },
        },
    );
}
```

### 4. Static Site Generation

Load secrets during build for SSG:

```javascript
// lib/get-static-props.js
export async function loadSecrets() {
    // Secrets are available during build
    return {
        apiEndpoint: process.env.API_ENDPOINT,
        publicKey: process.env.PUBLIC_KEY,
    };
}

// pages/index.js
export async function getStaticProps() {
    const secrets = await loadSecrets();

    // Fetch data using secrets
    const data = await fetch(secrets.apiEndpoint).then((r) => r.json());

    return {
        props: {
            data,
            // Don't expose sensitive secrets to client
            publicConfig: {
                publicKey: secrets.publicKey,
            },
        },
        revalidate: 3600, // ISR
    };
}
```

## Preview Deployments

### Automatic Preview Secrets

Configure preview-specific secrets:

```javascript
// scripts/preview-setup.js
const { execSync } = require("child_process");

const pullRequestNumber = process.env.VERCEL_GIT_PULL_REQUEST_NUMBER;
const commitRef = process.env.VERCEL_GIT_COMMIT_REF;

if (pullRequestNumber) {
    // Create preview environment for PR
    const previewEnv = `preview-pr-${pullRequestNumber}`;

    try {
        // Create preview environment from staging
        execSync(
            `dotenv env create ${previewEnv} --from staging --project ${process.env.DOTENV_PROJECT}`,
            { stdio: "inherit" },
        );
    } catch (e) {
        // Environment might already exist
        console.log("Preview environment might already exist");
    }

    // Pull preview secrets
    execSync(
        `dotenv pull ${previewEnv} --project ${process.env.DOTENV_PROJECT}`,
        { stdio: "inherit" },
    );
} else {
    // Regular branch preview
    execSync(`dotenv pull preview --project ${process.env.DOTENV_PROJECT}`, {
        stdio: "inherit",
    });
}
```

### Preview Comments

Add deployment info to PR comments:

```javascript
// scripts/preview-comment.js
const { Octokit } = require("@octokit/rest");

async function addPreviewComment() {
    const octokit = new Octokit({
        auth: process.env.GITHUB_TOKEN,
    });

    const deploymentUrl = process.env.VERCEL_URL;
    const prNumber = process.env.VERCEL_GIT_PULL_REQUEST_NUMBER;

    if (!prNumber) return;

    const [owner, repo] = process.env.VERCEL_GIT_REPO_SLUG.split("/");

    await octokit.issues.createComment({
        owner,
        repo,
        issue_number: prNumber,
        body: `
🚀 Preview deployment ready!

URL: https://${deploymentUrl}
Environment: preview-pr-${prNumber}
Secrets: Loaded from DotEnv

To update secrets for this preview:
\`\`\`bash
dotenv set KEY=value --environment preview-pr-${prNumber} --project ${process.env.DOTENV_PROJECT}
\`\`\`
    `,
    });
}

addPreviewComment();
```

## Security Best Practices

### 1. API Key Management

Store DotEnv API key securely:

```bash
# Add to Vercel project settings
vercel env add DOTENV_API_KEY production
vercel env add DOTENV_API_KEY preview
vercel env add DOTENV_API_KEY development

# Use different keys for different environments
vercel env add DOTENV_API_KEY_PROD production
vercel env add DOTENV_API_KEY_DEV development
```

### 2. Build-Time vs Runtime

Separate build and runtime secrets:

```javascript
// next.config.js
module.exports = {
    env: {
        // Build-time only (embedded in code)
        NEXT_PUBLIC_API_URL: process.env.API_URL,
    },
    serverRuntimeConfig: {
        // Server-side runtime only
        DATABASE_URL: process.env.DATABASE_URL,
        API_SECRET: process.env.API_SECRET,
    },
    publicRuntimeConfig: {
        // Client and server runtime
        APP_NAME: process.env.APP_NAME,
    },
};
```

### 3. Secret Validation

Validate required secrets:

```javascript
// scripts/validate-env.js
const required = {
    production: [
        "DATABASE_URL",
        "JWT_SECRET",
        "STRIPE_SECRET_KEY",
        "SENDGRID_API_KEY",
    ],
    preview: ["DATABASE_URL", "JWT_SECRET"],
    development: ["DATABASE_URL"],
};

const environment = process.env.VERCEL_ENV || "development";
const missing = required[environment].filter((key) => !process.env[key]);

if (missing.length > 0) {
    console.error(
        `Missing required environment variables: ${missing.join(", ")}`,
    );
    process.exit(1);
}

console.log("✅ All required environment variables are set");
```

### 4. Audit Logging

Track secret access:

```javascript
// middleware.js (Next.js 12+)
export function middleware(request) {
    // Log secret access for audit
    if (request.nextUrl.pathname.startsWith("/api/")) {
        console.log({
            timestamp: new Date().toISOString(),
            path: request.nextUrl.pathname,
            method: request.method,
            deployment: process.env.VERCEL_DEPLOYMENT_ID,
            environment: process.env.VERCEL_ENV,
        });
    }
}

export const config = {
    matcher: "/api/:path*",
};
```

## Deployment Workflows

### 1. GitHub Actions Integration

Deploy to Vercel with DotEnv:

```yaml
name: Deploy to Vercel

on:
    push:
        branches: [main, staging]
    pull_request:
        types: [opened, synchronize]

jobs:
    deploy:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v3

            - name: Setup Node.js
              uses: actions/setup-node@v3
              with:
                  node-version: 18

            - name: Install DotEnv CLI
              run: npm install -g @dotenv/cli

            - name: Load DotEnv Secrets
              env:
                  DOTENV_API_KEY: ${{ secrets.DOTENV_API_KEY }}
              run: |
                  if [ "${{ github.ref }}" = "refs/heads/main" ]; then
                    dotenv pull production --project my-app
                  else
                    dotenv pull staging --project my-app
                  fi

            - name: Deploy to Vercel
              uses: amondnet/vercel-action@v25
              with:
                  vercel-token: ${{ secrets.VERCEL_TOKEN }}
                  vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
                  vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
                  vercel-args: ${{ github.ref == 'refs/heads/main' && '--prod' || '' }}
```

### 2. Automatic Rollback

Rollback on deployment failure:

```javascript
// scripts/deployment-check.js
const axios = require("axios");

async function checkDeployment() {
    const deploymentUrl = process.env.VERCEL_URL;
    const maxRetries = 10;
    const delay = 30000; // 30 seconds

    for (let i = 0; i < maxRetries; i++) {
        try {
            const response = await axios.get(
                `https://${deploymentUrl}/api/health`,
            );

            if (response.status === 200 && response.data.status === "healthy") {
                console.log("✅ Deployment is healthy");
                return true;
            }
        } catch (error) {
            console.log(
                `Attempt ${i + 1}/${maxRetries} failed:`,
                error.message,
            );
        }

        if (i < maxRetries - 1) {
            await new Promise((resolve) => setTimeout(resolve, delay));
        }
    }

    console.error("❌ Deployment health check failed");

    // Trigger rollback
    await axios.post(
        "https://api.vercel.com/v1/deployments/rollback",
        {
            deploymentId: process.env.VERCEL_DEPLOYMENT_ID,
            projectId: process.env.VERCEL_PROJECT_ID,
        },
        {
            headers: {
                Authorization: `Bearer ${process.env.VERCEL_TOKEN}`,
            },
        },
    );

    return false;
}

checkDeployment();
```

### 3. Multi-Region Deployment

Deploy to multiple regions:

```json
{
    "regions": ["iad1", "sfo1", "lhr1"],
    "functions": {
        "api/data.js": {
            "regions": ["iad1", "lhr1"]
        }
    }
}
```

Load region-specific secrets:

```javascript
// api/regional-data.js
export default async function handler(req, res) {
    const region = process.env.VERCEL_REGION || "iad1";

    // Use region-specific configuration
    const dbUrl =
        process.env[`DATABASE_URL_${region.toUpperCase()}`] ||
        process.env.DATABASE_URL;

    // Connect to regional database
    const data = await fetchFromDatabase(dbUrl);

    res.status(200).json({
        region,
        data,
    });
}
```

## Monitoring

### 1. Function Logs

Access function logs with secret masking:

```javascript
// utils/logger.js
const sensitiveKeys = [
    "DATABASE_URL",
    "API_KEY",
    "JWT_SECRET",
    "STRIPE_SECRET_KEY",
];

export function log(message, data = {}) {
    // Mask sensitive data
    const masked = Object.entries(data).reduce((acc, [key, value]) => {
        if (sensitiveKeys.some((k) => key.includes(k))) {
            acc[key] = "[REDACTED]";
        } else {
            acc[key] = value;
        }
        return acc;
    }, {});

    console.log({
        timestamp: new Date().toISOString(),
        deployment: process.env.VERCEL_DEPLOYMENT_ID,
        message,
        ...masked,
    });
}
```

### 2. Custom Metrics

Track secret usage:

```javascript
// utils/metrics.js
export async function trackSecretAccess(secretName) {
    // Send to analytics
    await fetch("https://api.analytics.com/track", {
        method: "POST",
        headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${process.env.ANALYTICS_KEY}`,
        },
        body: JSON.stringify({
            event: "secret_accessed",
            properties: {
                secret_name: secretName,
                deployment_id: process.env.VERCEL_DEPLOYMENT_ID,
                environment: process.env.VERCEL_ENV,
                timestamp: new Date().toISOString(),
            },
        }),
    });
}
```

## Troubleshooting

### Common Issues

#### Secrets Not Available

```javascript
// Debug script
console.log("Vercel Environment:", process.env.VERCEL_ENV);
console.log("Available env vars:", Object.keys(process.env).sort());
console.log("DotEnv project:", process.env.DOTENV_PROJECT);
console.log("Has DATABASE_URL:", !!process.env.DATABASE_URL);
```

#### Build Failures

```json
{
    "scripts": {
        "build": "node scripts/debug-build.js && next build"
    }
}
```

```javascript
// scripts/debug-build.js
console.log("Build environment:");
console.log("- VERCEL:", !!process.env.VERCEL);
console.log("- VERCEL_ENV:", process.env.VERCEL_ENV);
console.log("- CI:", !!process.env.CI);

// Check DotEnv
try {
    require("child_process").execSync("dotenv --version", { stdio: "inherit" });
    console.log("✅ DotEnv CLI is available");
} catch {
    console.error("❌ DotEnv CLI not found");
}
```

## Examples

### Complete Next.js Setup

```javascript
// package.json
{
  "scripts": {
    "dev": "next dev",
    "build": "npm run load-env && next build",
    "start": "next start",
    "load-env": "node scripts/load-env.js"
  }
}

// scripts/load-env.js
const { execSync } = require('child_process');

function loadEnvironment() {
  if (!process.env.VERCEL) {
    console.log('Not running on Vercel, skipping DotEnv sync');
    return;
  }

  const apiKey = process.env.DOTENV_API_KEY;
  if (!apiKey) {
    console.error('DOTENV_API_KEY not set');
    process.exit(1);
  }

  const project = process.env.DOTENV_PROJECT || 'my-app';
  const environment = process.env.VERCEL_ENV === 'production' ? 'production' : 'preview';

  console.log(`Loading ${environment} secrets for ${project}`);

  try {
    execSync(
      `npx @dotenv/cli@latest auth login --api-key ${apiKey} && ` +
      `npx @dotenv/cli@latest pull ${environment} --project ${project}`,
      { stdio: 'inherit' }
    );
  } catch (error) {
    console.error('Failed to load secrets:', error);
    process.exit(1);
  }
}

loadEnvironment();

// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,

  // Environment variables
  env: {
    NEXT_PUBLIC_APP_URL: process.env.NEXT_PUBLIC_APP_URL,
    NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL,
  },

  // Serverless function config
  serverRuntimeConfig: {
    DATABASE_URL: process.env.DATABASE_URL,
    JWT_SECRET: process.env.JWT_SECRET,
    STRIPE_SECRET_KEY: process.env.STRIPE_SECRET_KEY,
  },

  // Headers for security
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'X-Content-Type-Options',
            value: 'nosniff',
          },
          {
            key: 'X-Frame-Options',
            value: 'DENY',
          },
          {
            key: 'X-XSS-Protection',
            value: '1; mode=block',
          },
        ],
      },
    ];
  },
};

module.exports = nextConfig;

// pages/api/health.js
export default function handler(req, res) {
  const checks = {
    database: !!process.env.DATABASE_URL,
    jwt: !!process.env.JWT_SECRET,
    stripe: !!process.env.STRIPE_SECRET_KEY,
  };

  const healthy = Object.values(checks).every(Boolean);

  res.status(healthy ? 200 : 503).json({
    status: healthy ? 'healthy' : 'unhealthy',
    checks,
    environment: process.env.VERCEL_ENV,
    deployment: process.env.VERCEL_DEPLOYMENT_ID,
  });
}
```

## Resources

- [Vercel Documentation](https://vercel.com/docs)
- [Vercel Environment Variables](https://vercel.com/docs/environment-variables)
- [DotEnv CLI Reference](/documentation/v1/cli/commands)
- [Next.js Environment Variables](https://nextjs.org/docs/basic-features/environment-variables)

## Next Steps

- [Set up Vercel integration](#installation)
- [Configure automatic sync](#automatic-sync)
- [Implement preview deployments](#preview-deployments)
- [Add monitoring](#monitoring)
