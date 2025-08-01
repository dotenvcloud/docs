---
title: Netlify Integration
slug: netlify
order: 8
tags: [integrations, netlify, jamstack, deployment]
---

# Netlify Integration

Integrate DotEnv with Netlify to manage secrets for your JAMstack applications and static sites. This guide covers setup, deployment workflows, and best practices.

## Overview

The DotEnv Netlify integration enables:

- Automatic environment variable synchronization
- Build-time secret injection
- Deploy preview secrets
- Production secret management
- Secure serverless functions
- Multi-site support

## Installation

### 1. Netlify Plugin (Recommended)

Install the DotEnv Netlify plugin:

```toml
# netlify.toml
[[plugins]]
  package = "@dotenv/netlify-plugin"

  [plugins.inputs]
    project = "my-app"
    environment = "production"
```

### 2. Build Environment Variables

Add your DotEnv API key in Netlify:

1. Go to Site settings → Environment variables
2. Add new variable:
    - Key: `DOTENV_API_KEY`
    - Value: Your DotEnv API key
    - Scopes: All deploys or specific contexts

### 3. Manual Configuration

Use build commands to load secrets:

```toml
# netlify.toml
[build]
  command = """
    curl -fsSL https://cli.dotenv.cloud/install.sh | sh && \
    export PATH="$HOME/.dotenv/bin:$PATH" && \
    dotenv auth login --api-key $DOTENV_API_KEY && \
    dotenv pull production --project my-app && \
    npm run build
  """
```

## Configuration

### Basic Setup

Configure in `netlify.toml`:

```toml
[build]
  command = "npm run build"
  publish = "dist"

[[plugins]]
  package = "@dotenv/netlify-plugin"

  [plugins.inputs]
    project = "my-app"
    environment = "production"
    prefix = "REACT_APP_"  # Optional: prefix for React apps

[build.environment]
  NODE_VERSION = "18"
  DOTENV_PROJECT = "my-app"
```

### Context-Based Configuration

Different secrets for different deploy contexts:

```toml
# Production context
[context.production]
  command = "npm run build:prod"

  [[context.production.plugins]]
    package = "@dotenv/netlify-plugin"

    [context.production.plugins.inputs]
      project = "my-app"
      environment = "production"

# Deploy preview context
[context.deploy-preview]
  command = "npm run build:preview"

  [[context.deploy-preview.plugins]]
    package = "@dotenv/netlify-plugin"

    [context.deploy-preview.plugins.inputs]
      project = "my-app"
      environment = "preview"

# Branch deploy context
[context.branch-deploy]
  command = "npm run build:staging"

  [[context.branch-deploy.plugins]]
    package = "@dotenv/netlify-plugin"

    [context.branch-deploy.plugins.inputs]
      project = "my-app"
      environment = "staging"
```

## Usage Patterns

### 1. Static Site Generation

For build-time secrets:

```javascript
// build.js
require("dotenv").config();

const config = {
    apiUrl: process.env.API_URL,
    publicKey: process.env.PUBLIC_KEY,
    analyticsId: process.env.ANALYTICS_ID,
};

// Use secrets during build
const siteData = await fetch(config.apiUrl).then((r) => r.json());

// Generate static files
generateStaticSite({
    data: siteData,
    config: {
        publicKey: config.publicKey,
        analyticsId: config.analyticsId,
    },
});
```

### 2. React Applications

Configure for Create React App:

```toml
[[plugins]]
  package = "@dotenv/netlify-plugin"

  [plugins.inputs]
    project = "my-react-app"
    environment = "production"
    prefix = "REACT_APP_"
```

Access in React:

```javascript
// src/config.js
export const config = {
    apiUrl: process.env.REACT_APP_API_URL,
    authDomain: process.env.REACT_APP_AUTH_DOMAIN,
    publicKey: process.env.REACT_APP_PUBLIC_KEY,
};

// src/App.js
import { config } from "./config";

function App() {
    useEffect(() => {
        // Initialize with public config
        initializeApp({
            apiUrl: config.apiUrl,
            authDomain: config.authDomain,
        });
    }, []);

    return <div>My App</div>;
}
```

### 3. Serverless Functions

Load secrets for Netlify Functions:

```javascript
// netlify/functions/api.js
exports.handler = async (event, context) => {
    // Secrets are available in process.env
    const apiKey = process.env.EXTERNAL_API_KEY;
    const databaseUrl = process.env.DATABASE_URL;

    try {
        const response = await fetch("https://api.external.com/data", {
            headers: {
                Authorization: `Bearer ${apiKey}`,
            },
        });

        const data = await response.json();

        return {
            statusCode: 200,
            body: JSON.stringify(data),
            headers: {
                "Content-Type": "application/json",
            },
        };
    } catch (error) {
        return {
            statusCode: 500,
            body: JSON.stringify({ error: "Internal server error" }),
        };
    }
};
```

Configure function secrets:

```toml
[functions]
  directory = "netlify/functions"

[[plugins]]
  package = "@dotenv/netlify-plugin"

  [plugins.inputs]
    project = "my-app"
    environment = "production"
    includeInFunctions = true
```

### 4. Edge Functions

Use secrets in Edge Functions:

```javascript
// netlify/edge-functions/geo-redirect.js
export default async (request, context) => {
    const apiKey = Deno.env.get("GEO_API_KEY");
    const country = context.geo.country?.code || "US";

    // Use secret for geo-based logic
    const geoData = await fetch(
        `https://api.geo.com/lookup?country=${country}`,
        {
            headers: { "X-API-Key": apiKey },
        },
    ).then((r) => r.json());

    if (geoData.redirect) {
        return Response.redirect(geoData.redirectUrl, 302);
    }

    return context.next();
};

export const config = {
    path: "/*",
};
```

## Deploy Previews

### Automatic Preview Secrets

Configure preview-specific environment:

```toml
[context.deploy-preview]
  [[context.deploy-preview.plugins]]
    package = "@dotenv/netlify-plugin"

    [context.deploy-preview.plugins.inputs]
      project = "my-app"
      environment = "preview"
      autoCreatePreview = true
      fromEnvironment = "staging"
```

### Pull Request Integration

```javascript
// scripts/preview-setup.js
const { PULL_REQUEST, REVIEW_ID, DEPLOY_URL } = process.env;

if (PULL_REQUEST && REVIEW_ID) {
    // Create preview environment
    const previewEnv = `preview-pr-${PULL_REQUEST}`;

    console.log(`Setting up preview for PR #${PULL_REQUEST}`);
    console.log(`Preview URL: ${DEPLOY_URL}`);

    // Preview environment will be created by plugin
}
```

### Dynamic Preview Configuration

```toml
[[plugins]]
  package = "@dotenv/netlify-plugin"

  [plugins.inputs]
    project = "my-app"
    environment = "preview-${PULL_REQUEST}"
    createIfNotExists = true
    copyFrom = "staging"
    dynamicValues = {
      "PREVIEW_URL" = "${DEPLOY_URL}",
      "PR_NUMBER" = "${PULL_REQUEST}",
      "COMMIT_REF" = "${COMMIT_REF}"
    }
```

## Build Plugins

### Custom DotEnv Plugin

Create custom build plugin:

```javascript
// plugins/dotenv-custom/index.js
module.exports = {
  onPreBuild: async ({ utils, inputs }) => {
    const { project, environment } = inputs;

    try {
      // Install DotEnv CLI
      await utils.run.command('curl -fsSL https://cli.dotenv.cloud/install.sh | sh');

      // Load secrets
      await utils.run.command(`
        export PATH="$HOME/.dotenv/bin:$PATH" &&
        dotenv auth login --api-key ${process.env.DOTENV_API_KEY} &&
        dotenv pull ${environment} --project ${project}
      `);

      // Validate required secrets
      const required = ['API_URL', 'AUTH_TOKEN'];
      for (const key of required) {
        if (!process.env[key]) {
          utils.build.failBuild(`Missing required secret: ${key}`);
        }
      }

      console.log('✅ DotEnv secrets loaded successfully');
    } catch (error) {
      utils.build.failBuild('Failed to load DotEnv secrets', { error });
    }
  },

  onPostBuild: async ({ utils }) => {
    // Clean up sensitive files
    await utils.run.command('rm -f .env');
  }
};

// plugins/dotenv-custom/manifest.yml
name: dotenv-custom
inputs:
  - name: project
    required: true
  - name: environment
    required: true
```

## Security Best Practices

### 1. Build-Time vs Runtime

Separate public and private configuration:

```javascript
// config/build.js
// Build-time only (not exposed to client)
const privateConfig = {
    databaseUrl: process.env.DATABASE_URL,
    apiSecret: process.env.API_SECRET,
    adminKey: process.env.ADMIN_KEY,
};

// Public config (embedded in build)
const publicConfig = {
    apiUrl: process.env.REACT_APP_API_URL,
    appName: process.env.REACT_APP_NAME,
    features: process.env.REACT_APP_FEATURES,
};

// Generate public config file
fs.writeFileSync("public/config.json", JSON.stringify(publicConfig));
```

### 2. Function Secret Isolation

Limit secret access per function:

```javascript
// netlify/functions/payment/payment.js
// Only has access to payment-related secrets
exports.handler = async (event) => {
    const stripeKey = process.env.STRIPE_SECRET_KEY;
    // Payment logic
};

// netlify/functions/email/email.js
// Only has access to email-related secrets
exports.handler = async (event) => {
    const sendgridKey = process.env.SENDGRID_API_KEY;
    // Email logic
};
```

Configure in plugin:

```toml
[[plugins]]
  package = "@dotenv/netlify-plugin"

  [plugins.inputs]
    project = "my-app"
    environment = "production"
    functionSecrets = {
      "payment" = ["STRIPE_SECRET_KEY", "PAYMENT_WEBHOOK_SECRET"],
      "email" = ["SENDGRID_API_KEY", "EMAIL_DOMAIN"]
    }
```

### 3. Secret Validation

Validate secrets during build:

```javascript
// scripts/validate-env.js
const required = {
    build: ["API_URL", "BUILD_KEY"],
    runtime: ["REACT_APP_API_URL", "REACT_APP_AUTH_DOMAIN"],
    functions: ["DATABASE_URL", "JWT_SECRET"],
};

// Check build secrets
const missingBuild = required.build.filter((key) => !process.env[key]);
if (missingBuild.length > 0) {
    console.error(`Missing build secrets: ${missingBuild.join(", ")}`);
    process.exit(1);
}

console.log("✅ All required secrets are present");
```

### 4. Audit Logging

Track secret access:

```toml
[[plugins]]
  package = "@dotenv/netlify-plugin"

  [plugins.inputs]
    project = "my-app"
    environment = "production"
    audit = true
    auditMetadata = {
      "site_id" = "${SITE_ID}",
      "deploy_id" = "${DEPLOY_ID}",
      "build_id" = "${BUILD_ID}",
      "context" = "${CONTEXT}"
    }
```

## Advanced Features

### 1. Monorepo Support

Configure for monorepo:

```toml
# Base configuration
[build]
  base = "apps/web"

# Site-specific secrets
[[plugins]]
  package = "@dotenv/netlify-plugin"

  [plugins.inputs]
    project = "my-monorepo-web"
    environment = "production"
    workingDirectory = "apps/web"

# Shared secrets
[build.environment]
  DOTENV_SHARED_PROJECT = "my-monorepo-shared"
  DOTENV_SHARED_ENV = "production"
```

### 2. Multi-Region Deployment

Deploy to different regions:

```javascript
// scripts/deploy-region.js
const region = process.env.DEPLOY_REGION || "us-east";
const regionConfig = {
    "us-east": {
        apiUrl: process.env.US_EAST_API_URL,
        cdnUrl: process.env.US_EAST_CDN_URL,
    },
    "eu-west": {
        apiUrl: process.env.EU_WEST_API_URL,
        cdnUrl: process.env.EU_WEST_CDN_URL,
    },
    "ap-south": {
        apiUrl: process.env.AP_SOUTH_API_URL,
        cdnUrl: process.env.AP_SOUTH_CDN_URL,
    },
};

// Generate region-specific config
generateConfig(regionConfig[region]);
```

### 3. A/B Testing Configuration

Configure feature flags:

```toml
[[plugins]]
  package = "@dotenv/netlify-plugin"

  [plugins.inputs]
    project = "my-app"
    environment = "production"
    splitTests = {
      "new-checkout" = {
        branches = ["main", "feature/new-checkout"],
        secrets = ["CHECKOUT_API_KEY", "PAYMENT_PROVIDER"]
      }
    }
```

## Monitoring

### 1. Build Notifications

Configure build notifications:

```javascript
// plugins/build-notify/index.js
module.exports = {
    onSuccess: async ({ utils, inputs }) => {
        const { SLACK_WEBHOOK, DEPLOY_URL, COMMIT_REF } = process.env;

        await fetch(SLACK_WEBHOOK, {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
                text: `✅ Deploy successful!`,
                attachments: [
                    {
                        color: "good",
                        fields: [
                            { title: "Site", value: DEPLOY_URL },
                            { title: "Commit", value: COMMIT_REF },
                            { title: "Secrets", value: "Loaded from DotEnv" },
                        ],
                    },
                ],
            }),
        });
    },

    onError: async ({ utils, error }) => {
        console.error("Build failed:", error);
        // Notify about failure
    },
};
```

### 2. Analytics Integration

Track deployments:

```javascript
// scripts/track-deploy.js
const analytics = require("@segment/analytics-node");

const client = new analytics(process.env.SEGMENT_WRITE_KEY);

client.track({
    userId: process.env.SITE_ID,
    event: "Deployment Completed",
    properties: {
        environment: process.env.CONTEXT,
        deployId: process.env.DEPLOY_ID,
        branch: process.env.BRANCH,
        secretsLoaded: true,
        provider: "dotenv",
    },
});
```

## Troubleshooting

### Common Issues

#### Secrets Not Available

```toml
# Debug configuration
[[plugins]]
  package = "@dotenv/netlify-plugin"

  [plugins.inputs]
    project = "my-app"
    environment = "production"
    debug = true
    verbose = true
```

Check build logs:

```
8:30:00 AM: ────────────────────────────────────────────────
8:30:00 AM: DotEnv Netlify Plugin
8:30:00 AM: ────────────────────────────────────────────────
8:30:01 AM: Loading secrets for project: my-app
8:30:01 AM: Environment: production
8:30:02 AM: ✅ Loaded 15 secrets
```

#### Build Failures

Debug script:

```javascript
// scripts/debug-env.js
console.log("Build context:", process.env.CONTEXT);
console.log("Deploy context:", process.env.DEPLOY_ID);
console.log(
    "Available env vars:",
    Object.keys(process.env)
        .filter((k) => !k.includes("SECRET"))
        .sort(),
);
console.log("Has DOTENV_API_KEY:", !!process.env.DOTENV_API_KEY);
```

#### Function Timeouts

Optimize function startup:

```javascript
// netlify/functions/optimized.js
// Load heavy dependencies conditionally
let heavyDep;

exports.handler = async (event) => {
    // Lazy load on first request
    if (!heavyDep) {
        heavyDep = require("heavy-dependency");
    }

    // Function logic
    return {
        statusCode: 200,
        body: JSON.stringify({ success: true }),
    };
};
```

## Examples

### Complete Gatsby Site

```toml
# netlify.toml
[build]
  command = "gatsby build"
  publish = "public"

[[plugins]]
  package = "@dotenv/netlify-plugin"

  [plugins.inputs]
    project = "my-gatsby-site"
    environment = "production"
    prefix = "GATSBY_"

[[plugins]]
  package = "@netlify/plugin-gatsby"

[build.environment]
  NODE_VERSION = "18"
  GATSBY_CPU_COUNT = "2"

# Production context
[context.production.environment]
  GATSBY_ACTIVE_ENV = "production"

# Preview context
[context.deploy-preview.environment]
  GATSBY_ACTIVE_ENV = "preview"

[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-XSS-Protection = "1; mode=block"
    X-Content-Type-Options = "nosniff"
    Referrer-Policy = "strict-origin-when-cross-origin"
```

### Next.js with ISR

```javascript
// next.config.js
module.exports = {
    env: {
        // Public runtime config
        NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL,
        NEXT_PUBLIC_APP_NAME: process.env.NEXT_PUBLIC_APP_NAME,
    },

    // Server-side only
    serverRuntimeConfig: {
        apiSecret: process.env.API_SECRET,
        databaseUrl: process.env.DATABASE_URL,
    },

    // Incremental Static Regeneration
    experimental: {
        isrMemoryCacheSize: 0, // Disable in-memory cache for Netlify
    },
};
```

```toml
# netlify.toml
[build]
  command = "npm run build"
  publish = ".next"

[[plugins]]
  package = "@netlify/plugin-nextjs"

[[plugins]]
  package = "@dotenv/netlify-plugin"

  [plugins.inputs]
    project = "my-nextjs-app"
    environment = "production"
    includeInFunctions = true
    includeInEdge = true

[functions]
  included_files = ["!node_modules/@swc/core-linux-x64-gnu/**", "!node_modules/@swc/core-linux-x64-musl/**"]
```

## Migration Guide

### From Environment Variables UI

Export and migrate existing variables:

```javascript
// scripts/migrate-to-dotenv.js
const existingVars = {
    // Export from Netlify UI
    API_URL: "https://api.example.com",
    API_KEY: "sk_live_xxx",
    // ... more variables
};

// Import to DotEnv
const { execSync } = require("child_process");

Object.entries(existingVars).forEach(([key, value]) => {
    execSync(
        `dotenv set ${key}="${value}" --project my-app --environment production`,
        { stdio: "inherit" },
    );
});

console.log("✅ Migration complete");
```

## Resources

- [Netlify Documentation](https://docs.netlify.com/)
- [Build Plugins](https://docs.netlify.com/configure-builds/build-plugins/)
- [Environment Variables](https://docs.netlify.com/configure-builds/environment-variables/)
- [DotEnv CLI Reference](/documentation/v1/cli/commands)

## Next Steps

- [Install DotEnv plugin](#netlify-plugin-recommended)
- [Configure deploy contexts](#context-based-configuration)
- [Set up serverless functions](#serverless-functions)
- [Enable preview deployments](#deploy-previews)
