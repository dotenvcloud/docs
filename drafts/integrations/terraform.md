---
title: Terraform Provider
slug: terraform
order: 30
tags: [integrations, terraform, infrastructure, iac]
---

# Terraform Provider

Manage DotEnv secrets and configurations using Terraform infrastructure as code.

## Overview

The DotEnv Terraform provider enables:

- Infrastructure as Code for secrets management
- Automated secret provisioning
- Environment configuration management
- Secret rotation automation
- Team and permission management

## Provider Configuration

```hcl
terraform {
  required_providers {
    dotenv = {
      source  = "dotenv/dotenv"
      version = "~> 1.0"
    }
  }
}

provider "dotenv" {
  api_key = var.dotenv_api_key
  api_url = "https://api.dotenv.cloud"
}
```

## Resources

### dotenv_project

```hcl
resource "dotenv_project" "main" {
  name        = "my-application"
  description = "Production application secrets"
  
  settings {
    encryption_mode = "client_managed"
    audit_logging   = true
  }
}
```

### dotenv_secret

```hcl
resource "dotenv_secret" "database_url" {
  project_id = dotenv_project.main.id
  key        = "DATABASE_URL"
  value      = var.database_url
  encrypted  = true
  
  tags = {
    environment = "production"
    service     = "database"
  }
}
```

### dotenv_environment

```hcl
resource "dotenv_environment" "production" {
  project_id = dotenv_project.main.id
  name       = "production"
  
  secrets = {
    DATABASE_URL = dotenv_secret.database_url.id
    API_KEY      = dotenv_secret.api_key.id
  }
}
```

## Data Sources

### dotenv_project

```hcl
data "dotenv_project" "existing" {
  name = "my-existing-project"
}
```

### dotenv_secrets

```hcl
data "dotenv_secrets" "all" {
  project_id = data.dotenv_project.existing.id
  
  filter {
    tags = {
      environment = "production"
    }
  }
}
```

## Examples

### Complete Infrastructure Setup

```hcl
# Create project
resource "dotenv_project" "app" {
  name = "terraform-managed-app"
}

# Create secrets
resource "dotenv_secret" "app_secrets" {
  for_each = var.app_secrets
  
  project_id = dotenv_project.app.id
  key        = each.key
  value      = each.value
  encrypted  = true
}

# Create environments
resource "dotenv_environment" "environments" {
  for_each = toset(["development", "staging", "production"])
  
  project_id = dotenv_project.app.id
  name       = each.key
  
  secrets = {
    for k, v in dotenv_secret.app_secrets : k => v.id
  }
}
```

## Implementation Status

This feature is currently in planning. The Terraform provider will be available in the Terraform Registry once implemented.