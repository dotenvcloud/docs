---
title: Your First Project
slug: first-project
order: 3
tags: [project, setup, tutorial]
---

# Your First Project

This guide walks you through creating and configuring your first DotEnv project step by step.

## Understanding Projects

A project in DotEnv represents a single application or service. Each project can have:

- Multiple environments (development, staging, production)
- Team members with specific permissions
- Its own set of secrets
- Unique API keys for access

## Creating a Project

### Using the Web Dashboard

1. Log in to [dotenv.cloud](https://dotenv.cloud)
2. Click "Create Project"
3. Enter your project details:
    - **Name**: A unique identifier (e.g., `my-app`)
    - **Display Name**: Human-readable name (e.g., "My Application")
    - **Description**: Optional project description

### Using the CLI

```bash
# Basic project creation
dotenv projects create my-app

# With additional details
dotenv projects create my-app \
  --display-name "My Application" \
  --description "E-commerce platform backend"
```

## Project Structure

Once created, your project has this structure:

```
my-app/
├── environments/
│   ├── development
│   ├── staging
│   └── production
├── secrets/
├── team/
└── settings/
```

## Adding Your First Secrets

### Individual Secrets

Add secrets one at a time:

```bash
# Add to default environment (development)
dotenv secrets set API_KEY="sk-1234567890" --project my-app

# Add to specific environment
dotenv secrets set DATABASE_URL="postgresql://prod-db.example.com/myapp" \
  --project my-app \
  --environment production
```

### Bulk Import

Import from an existing .env file:

```bash
# Import from .env file
dotenv secrets import .env --project my-app

# Import to specific environment
dotenv secrets import .env.production \
  --project my-app \
  --environment production
```

### Interactive Mode

Use interactive mode for sensitive values:

```bash
dotenv secrets set --interactive --project my-app
# Prompts for key and value, hiding value input
```

## Environment Configuration

### Default Environments

Every project starts with three environments:

- **development**: Local development
- **staging**: Testing and QA
- **production**: Live application

### Creating Custom Environments

```bash
# Create a new environment
dotenv environments create qa --project my-app

# Clone an existing environment
dotenv environments clone staging preview --project my-app
```

### Environment Hierarchy

Environments can inherit values:

```bash
# Set up inheritance
dotenv environments config staging \
  --inherits-from development \
  --project my-app

# Override specific values
dotenv secrets set API_URL="https://staging-api.example.com" \
  --project my-app \
  --environment staging
```

## Team Collaboration

### Inviting Team Members

```bash
# Invite with default permissions
dotenv projects invite john@example.com --project my-app

# Invite with specific role
dotenv projects invite jane@example.com \
  --project my-app \
  --role developer
```

### Permission Levels

- **Owner**: Full project control
- **Admin**: Manage settings and team
- **Developer**: Read/write secrets
- **Viewer**: Read-only access

## Accessing Secrets

### Local Development

```bash
# Generate .env file
dotenv secrets pull --project my-app > .env

# Export to shell
eval $(dotenv secrets pull --project my-app --format shell)

# Run command with secrets
dotenv run --project my-app -- npm start
```

### In Your Application

**Node.js Example:**

```javascript
import { DotEnv } from "@dotenv/sdk";

const dotenv = new DotEnv({
    apiKey: process.env.DOTENV_API_KEY,
    project: "my-app",
    environment: process.env.NODE_ENV || "development",
});

// Load all secrets
await dotenv.load();

// Access secrets
console.log(process.env.DATABASE_URL);
console.log(process.env.API_KEY);
```

**Python Example:**

```python
from dotenv_sdk import DotEnv
import os

dotenv = DotEnv(
    api_key=os.getenv('DOTENV_API_KEY'),
    project='my-app',
    environment=os.getenv('ENV', 'development')
)

# Load secrets
dotenv.load()

# Use secrets
database_url = os.getenv('DATABASE_URL')
api_key = os.getenv('API_KEY')
```

## Project Settings

### Configure Project Defaults

```bash
# Set default environment
dotenv projects config my-app --default-environment staging

# Enable audit logging
dotenv projects config my-app --audit-logs enabled

# Set secret expiration
dotenv projects config my-app --secret-ttl 90d
```

### Security Settings

Enable additional security features:

```bash
# Require 2FA for all team members
dotenv projects security my-app --require-2fa

# Enable IP allowlist
dotenv projects security my-app --ip-allowlist "192.168.1.0/24,10.0.0.0/8"

# Set up webhook for changes
dotenv projects webhooks create \
  --project my-app \
  --url https://api.example.com/webhooks/dotenv \
  --events secret.created,secret.updated,secret.deleted
```

## Example: Complete Setup

Here's a complete example setting up a Node.js web application:

```bash
#!/bin/bash
# setup-project.sh

PROJECT_NAME="todo-app"

# 1. Create project
echo "Creating project..."
dotenv projects create $PROJECT_NAME \
  --display-name "Todo List Application" \
  --description "Simple task management app"

# 2. Add development secrets
echo "Adding development secrets..."
dotenv secrets set DATABASE_URL="postgresql://localhost/todo_dev" \
  --project $PROJECT_NAME \
  --environment development

dotenv secrets set REDIS_URL="redis://localhost:6379/0" \
  --project $PROJECT_NAME \
  --environment development

dotenv secrets set JWT_SECRET="dev-secret-change-in-prod" \
  --project $PROJECT_NAME \
  --environment development

dotenv secrets set PORT="3000" \
  --project $PROJECT_NAME \
  --environment development

# 3. Add production secrets
echo "Adding production secrets..."
dotenv secrets set DATABASE_URL="postgresql://prod.db.example.com/todo" \
  --project $PROJECT_NAME \
  --environment production

dotenv secrets set REDIS_URL="redis://prod.redis.example.com:6379/0" \
  --project $PROJECT_NAME \
  --environment production

dotenv secrets set --interactive JWT_SECRET \
  --project $PROJECT_NAME \
  --environment production

# 4. Set up staging to inherit from development
echo "Configuring staging environment..."
dotenv environments config staging \
  --inherits-from development \
  --project $PROJECT_NAME

# Override database for staging
dotenv secrets set DATABASE_URL="postgresql://staging.db.example.com/todo" \
  --project $PROJECT_NAME \
  --environment staging

# 5. Invite team members
echo "Inviting team members..."
dotenv projects invite alice@example.com --project $PROJECT_NAME --role admin
dotenv projects invite bob@example.com --project $PROJECT_NAME --role developer

echo "✅ Project setup complete!"
echo ""
echo "To use in development:"
echo "  dotenv run --project $PROJECT_NAME -- npm start"
echo ""
echo "Or generate .env file:"
echo "  dotenv secrets pull --project $PROJECT_NAME > .env"
```

## Best Practices

### 1. Use Descriptive Names

```bash
# Good
DATABASE_URL
STRIPE_SECRET_KEY
REDIS_CONNECTION_STRING

# Avoid
DB
KEY
CONN_STR
```

### 2. Separate by Environment

Never use production secrets in development:

```bash
# Development
API_URL=http://localhost:3000
DEBUG=true

# Production
API_URL=https://api.example.com
DEBUG=false
```

### 3. Regular Rotation

Set up automatic rotation for sensitive keys:

```bash
dotenv secrets rotate JWT_SECRET \
  --project my-app \
  --all-environments \
  --schedule "0 0 1 * *"  # Monthly
```

### 4. Document Secrets

Add descriptions to help team members:

```bash
dotenv secrets describe DATABASE_URL \
  "PostgreSQL connection string. Format: postgresql://user:pass@host/db" \
  --project my-app
```

## Troubleshooting

### Secret Not Loading

Check environment and project:

```bash
# Verify secret exists
dotenv secrets list --project my-app --environment development

# Check current environment
echo $NODE_ENV
```

### Permission Denied

Verify your access level:

```bash
# Check your permissions
dotenv projects members --project my-app

# Request access from admin
dotenv projects request-access my-app
```

### Environment Mismatch

Ensure you're using the correct environment:

```javascript
// Always specify environment explicitly
const dotenv = new DotEnv({
    apiKey: process.env.DOTENV_API_KEY,
    project: "my-app",
    environment: process.env.NODE_ENV || "development", // Explicit fallback
});
```

## Next Steps

- [Team Setup](./team-setup) - Configure team access and permissions
- [Environment Management](./environments) - Deep dive into environments
- [Security Best Practices](./best-practices) - Secure your secrets
- [CI/CD Integration](/documentation/v1/integrations/ci-cd) - Automate deployments
