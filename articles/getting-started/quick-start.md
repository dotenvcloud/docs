---
title: Quick Start Guide
slug: quick-start
order: 1
tags: [quickstart, getting-started, basics]
---

# Quick Start Guide

Get up and running with DotEnv in less than 5 minutes. This guide covers the essential steps to start managing your environment variables securely.

## Prerequisites

Before you begin, ensure you have:

- A DotEnv account (sign up at [dotenv.cloud](https://dotenv.cloud))
- Node.js 16+ or your preferred runtime environment
- Basic familiarity with environment variables

## Step 1: Install the CLI

The DotEnv CLI provides the easiest way to manage your secrets from the command line.

### macOS/Linux

```bash
curl -sSL https://dotenv.cloud/install.sh | sh
```

### Windows

```powershell
iwr -useb https://dotenv.cloud/install.ps1 | iex
```

### Alternative: Using npm

```bash
npm install -g @dotenv/cli
```

## Step 2: Authenticate

Log in to your DotEnv account:

```bash
dotenv login
```

This will open your browser for authentication. Once complete, your CLI will be authenticated.

## Step 3: Create Your First Project

Projects organize your secrets by application or service:

```bash
dotenv projects create my-app
```

## Step 4: Add Secrets

Add your first secret to the project:

```bash
dotenv secrets set DATABASE_URL="postgresql://user:pass@localhost/mydb" --project my-app
```

You can also add multiple secrets at once:

```bash
dotenv secrets set \
  API_KEY="sk-1234567890" \
  REDIS_URL="redis://localhost:6379" \
  --project my-app
```

## Step 5: Pull Secrets to Your Application

### Using the CLI

Generate a .env file for local development:

```bash
dotenv secrets pull --project my-app --environment development > .env
```

### Using SDKs

**Node.js/JavaScript:**

```javascript
import { DotEnv } from "@dotenv/sdk";

const dotenv = new DotEnv({
    apiKey: process.env.DOTENV_API_KEY,
    project: "my-app",
    environment: "development",
});

await dotenv.load();
// Your secrets are now available in process.env
```

**Python:**

```python
from dotenv_sdk import DotEnv

dotenv = DotEnv(
    api_key=os.getenv('DOTENV_API_KEY'),
    project='my-app',
    environment='development'
)

dotenv.load()
# Your secrets are now available in os.environ
```

## Step 6: Deploy with Docker

For production deployments, fetch secrets at container startup:

```dockerfile
FROM node:18-alpine

# Install DotEnv CLI
RUN curl -sSL https://dotenv.cloud/install.sh | sh

# Copy application files
WORKDIR /app
COPY . .

# Fetch secrets and start application
CMD dotenv secrets pull --project my-app --environment production > .env && \
    node server.js
```

## What's Next?

- **[Set up environments](./environments)** - Learn about development, staging, and production environments
- **[Team collaboration](./team-setup)** - Invite team members and manage permissions
- **[Best practices](./best-practices)** - Security recommendations and usage patterns
- **[CLI reference](/documentation/v1/cli/commands)** - Explore all available CLI commands

## Example: Complete Node.js Setup

Here's a complete example setting up a Node.js application:

```javascript
// install.js - First time setup
const { execSync } = require("child_process");

// Install DotEnv SDK
execSync("npm install @dotenv/sdk");

// Create project
execSync("dotenv projects create my-node-app");

// Add common secrets
const secrets = {
    DATABASE_URL: "postgresql://localhost/myapp",
    REDIS_URL: "redis://localhost:6379",
    JWT_SECRET: "your-secret-key",
    PORT: "3000",
};

for (const [key, value] of Object.entries(secrets)) {
    execSync(`dotenv secrets set ${key}="${value}" --project my-node-app`);
}

console.log("✅ DotEnv setup complete!");
```

```javascript
// app.js - Your application
import { DotEnv } from "@dotenv/sdk";
import express from "express";

// Load secrets
const dotenv = new DotEnv({
    apiKey: process.env.DOTENV_API_KEY,
    project: "my-node-app",
    environment: process.env.NODE_ENV || "development",
});

await dotenv.load();

// Your app now has access to all secrets
const app = express();
const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
```

## Troubleshooting

### Authentication Issues

If you encounter authentication problems:

```bash
dotenv logout
dotenv login --force
```

### Missing Secrets

Verify your secrets are set correctly:

```bash
dotenv secrets list --project my-app
```

### Permission Denied

Ensure you have the correct permissions:

```bash
dotenv projects members --project my-app
```

## Need Help?

- 📧 Email: support@dotenv.cloud
- 💬 Discord: [Join our community](https://discord.gg/dotenv)
- 📚 Full documentation: [docs.dotenv.cloud](https://docs.dotenv.cloud)
