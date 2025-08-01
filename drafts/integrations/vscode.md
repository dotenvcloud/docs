---
title: VS Code Integration
slug: vscode
order: 15
tags: [integrations, vscode, editor, development]
---

# VS Code Integration

Integrate DotEnv with Visual Studio Code to enhance your development workflow. This guide covers extension setup, workspace configuration, and development best practices.

## Overview

The DotEnv VS Code integration provides:

- Syntax highlighting for .env files
- IntelliSense for environment variables
- Secret validation and linting
- Quick actions and commands
- Integrated terminal support
- Multi-root workspace support

## Installation

### 1. VS Code Extension

Install the DotEnv extension:

```bash
# Via VS Code
1. Open VS Code
2. Press Cmd+Shift+X (Extensions)
3. Search for "DotEnv"
4. Click Install

# Via command line
code --install-extension dotenv.vscode-dotenv
```

### 2. CLI Integration

Configure VS Code to use DotEnv CLI:

```json
// .vscode/settings.json
{
    "dotenv.cli.path": "/usr/local/bin/dotenv",
    "dotenv.defaultProject": "${workspaceFolderBasename}",
    "dotenv.defaultEnvironment": "development",
    "dotenv.autoSync": true,
    "dotenv.syncOnSave": true
}
```

### 3. Workspace Configuration

Create workspace settings:

```json
// my-project.code-workspace
{
    "folders": [
        {
            "path": ".",
            "name": "My Project"
        }
    ],
    "settings": {
        "dotenv.enableAutoComplete": true,
        "dotenv.enableValidation": true,
        "dotenv.showInlineValues": true,
        "files.associations": {
            ".env*": "dotenv"
        }
    }
}
```

## Features

### Syntax Highlighting

Enhanced syntax highlighting for .env files:

```env
# Database Configuration
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
DATABASE_POOL_SIZE=10

# API Keys
API_KEY=sk_live_abc123def456  # Highlighted as sensitive
JWT_SECRET=your-256-bit-secret

# Feature Flags
ENABLE_FEATURE_X=true
DEBUG_MODE=false

# URLs and Endpoints
API_BASE_URL=https://api.example.com
WEBHOOK_URL=https://hooks.example.com/webhook
```

### IntelliSense

Auto-completion for environment variables:

```javascript
// When you type process.env.
// VS Code suggests available variables from .env files

const dbUrl = process.env.DATABASE_URL; // Auto-completed
const apiKey = process.env.API_KEY; // With type information
```

### Validation & Linting

Real-time validation of .env files:

```env
# ❌ Error: Duplicate key
API_KEY=abc123
API_KEY=def456

# ⚠️ Warning: Empty value
DATABASE_URL=

# ❌ Error: Invalid format
INVALID LINE WITHOUT EQUALS

# ✅ Valid with quotes
MULTILINE_KEY="line1\nline2\nline3"
```

## Commands

### Command Palette

Access DotEnv commands (Cmd+Shift+P):

- `DotEnv: Pull Environment` - Pull secrets from DotEnv
- `DotEnv: Push Environment` - Push local changes
- `DotEnv: Switch Environment` - Change active environment
- `DotEnv: Compare Environments` - Diff between environments
- `DotEnv: Validate Secrets` - Check secret format
- `DotEnv: Sync All` - Sync all environments

### Keyboard Shortcuts

Default keybindings:

```json
// keybindings.json
[
    {
        "key": "cmd+shift+e",
        "command": "dotenv.pullEnvironment",
        "when": "resourceExtname == .env"
    },
    {
        "key": "cmd+shift+p",
        "command": "dotenv.pushEnvironment",
        "when": "resourceExtname == .env"
    },
    {
        "key": "cmd+shift+s",
        "command": "dotenv.switchEnvironment"
    }
]
```

## Tasks Integration

### Pull Secrets Task

```json
// .vscode/tasks.json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Pull DotEnv Secrets",
            "type": "shell",
            "command": "dotenv",
            "args": [
                "pull",
                "${input:environment}",
                "--project",
                "${workspaceFolderBasename}"
            ],
            "group": "build",
            "presentation": {
                "reveal": "always",
                "panel": "new"
            },
            "problemMatcher": []
        },
        {
            "label": "Sync All Environments",
            "type": "shell",
            "command": "dotenv",
            "args": ["sync", "--all"],
            "group": "build"
        }
    ],
    "inputs": [
        {
            "id": "environment",
            "type": "pickString",
            "description": "Select environment",
            "options": ["development", "staging", "production"]
        }
    ]
}
```

### Launch Configuration

```json
// .vscode/launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "node",
            "request": "launch",
            "name": "Launch with DotEnv",
            "program": "${workspaceFolder}/index.js",
            "preLaunchTask": "Pull DotEnv Secrets",
            "envFile": "${workspaceFolder}/.env",
            "env": {
                "NODE_ENV": "development",
                "DOTENV_PROJECT": "${workspaceFolderBasename}"
            }
        }
    ]
}
```

## Debugging

### Environment Variable Viewer

View all environment variables during debugging:

```json
// launch.json
{
    "type": "node",
    "request": "launch",
    "name": "Debug with Env Viewer",
    "program": "${workspaceFolder}/index.js",
    "env": {
        "NODE_OPTIONS": "--inspect-brk"
    },
    "console": "integratedTerminal",
    "outputCapture": "std",
    "showAsyncStacks": true
}
```

### Conditional Breakpoints

Use environment variables in breakpoints:

```javascript
// Set breakpoint condition: process.env.DEBUG_MODE === 'true'
function processPayment(amount) {
    debugger; // Only triggers when DEBUG_MODE is true

    if (process.env.PAYMENT_PROVIDER === "stripe") {
        return processStripePayment(amount);
    }
}
```

## Multi-Root Workspaces

### Workspace Configuration

Configure multiple projects:

```json
// company.code-workspace
{
    "folders": [
        {
            "path": "apps/frontend",
            "name": "Frontend"
        },
        {
            "path": "apps/backend",
            "name": "Backend"
        },
        {
            "path": "packages/shared",
            "name": "Shared"
        }
    ],
    "settings": {
        "dotenv.projects": {
            "Frontend": {
                "project": "frontend-app",
                "defaultEnvironment": "development"
            },
            "Backend": {
                "project": "backend-api",
                "defaultEnvironment": "development"
            },
            "Shared": {
                "project": "shared-lib",
                "defaultEnvironment": "development"
            }
        }
    }
}
```

### Per-Folder Settings

```json
// apps/frontend/.vscode/settings.json
{
  "dotenv.project": "frontend-app",
  "dotenv.environments": ["development", "staging", "production"],
  "dotenv.autoSync": true
}

// apps/backend/.vscode/settings.json
{
  "dotenv.project": "backend-api",
  "dotenv.environments": ["development", "staging", "production"],
  "dotenv.syncOnSave": false
}
```

## Extension Settings

### Global Settings

```json
// User settings.json
{
    "dotenv.enableTelemetry": false,
    "dotenv.showStatusBar": true,
    "dotenv.confirmBeforePush": true,
    "dotenv.maskSecrets": true,
    "dotenv.colorizeOutput": true,
    "dotenv.defaultOrganization": "my-org",
    "dotenv.apiEndpoint": "https://api.dotenv.cloud"
}
```

### Project Settings

```json
// Workspace settings.json
{
    "dotenv.includeFiles": [".env", ".env.local", ".env.*.local"],
    "dotenv.excludeFiles": [".env.example", ".env.test"],
    "dotenv.validation": {
        "requireValues": true,
        "checkDuplicates": true,
        "validateFormat": true
    }
}
```

## Security Features

### Secret Masking

Automatically mask sensitive values:

```json
{
    "dotenv.maskPatterns": [
        "*_KEY",
        "*_SECRET",
        "*_TOKEN",
        "*_PASSWORD",
        "DATABASE_URL"
    ],
    "dotenv.maskInOutput": true,
    "dotenv.maskInDebugConsole": true
}
```

### Git Integration

Prevent accidental commits:

```json
{
    "dotenv.gitCheck": true,
    "dotenv.preventCommit": [".env", ".env.local", ".env.*.local"],
    "dotenv.autoAddToGitignore": true
}
```

## Code Snippets

### Custom Snippets

Create DotEnv snippets:

```json
// .vscode/dotenv.code-snippets
{
    "Load DotEnv": {
        "prefix": "dotenv-load",
        "body": [
            "const { execSync } = require('child_process');",
            "",
            "// Load secrets from DotEnv",
            "execSync('dotenv pull ${1:environment} --project ${2:project}');",
            "require('dotenv').config();",
            "",
            "$0"
        ],
        "description": "Load DotEnv secrets"
    },

    "Environment Check": {
        "prefix": "dotenv-check",
        "body": [
            "if (!process.env.${1:REQUIRED_VAR}) {",
            "  throw new Error('Missing required environment variable: ${1:REQUIRED_VAR}');",
            "}",
            "",
            "$0"
        ],
        "description": "Check for required environment variable"
    }
}
```

## Terminal Integration

### Integrated Terminal

Auto-load environment in terminal:

```json
{
    "terminal.integrated.env.osx": {
        "DOTENV_AUTO_LOAD": "true"
    },
    "terminal.integrated.profiles.osx": {
        "DotEnv Shell": {
            "path": "bash",
            "args": ["-c", "dotenv pull development && exec bash"],
            "icon": "terminal-bash"
        }
    }
}
```

### Task Runner

```json
// tasks.json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Run with DotEnv",
            "type": "shell",
            "command": "dotenv",
            "args": ["run", "--", "npm", "start"],
            "options": {
                "env": {
                    "DOTENV_PROJECT": "${workspaceFolderBasename}",
                    "DOTENV_ENVIRONMENT": "${input:environment}"
                }
            }
        }
    ]
}
```

## Troubleshooting

### Common Issues

#### Extension Not Loading

```bash
# Check extension status
code --list-extensions | grep dotenv

# Reinstall extension
code --uninstall-extension dotenv.vscode-dotenv
code --install-extension dotenv.vscode-dotenv

# Clear extension cache
rm -rf ~/.vscode/extensions/dotenv.vscode-dotenv-*
```

#### IntelliSense Not Working

```json
// Ensure file associations
{
    "files.associations": {
        ".env": "dotenv",
        ".env.*": "dotenv"
    },
    "dotenv.enableAutoComplete": true,
    "editor.quickSuggestions": {
        "other": true,
        "comments": false,
        "strings": true
    }
}
```

#### Sync Issues

Debug sync problems:

```javascript
// .vscode/debug-dotenv.js
const { execSync } = require("child_process");

try {
    console.log("Testing DotEnv CLI...");
    const version = execSync("dotenv --version").toString();
    console.log("DotEnv version:", version);

    console.log("Testing API connection...");
    const result = execSync("dotenv status").toString();
    console.log("Status:", result);
} catch (error) {
    console.error("Error:", error.message);
}
```

## Best Practices

### 1. Project Structure

Organize environment files:

```
my-project/
├── .env.example          # Template with all variables
├── .env                  # Local development (gitignored)
├── .env.local           # Local overrides (gitignored)
├── .vscode/
│   ├── settings.json    # Project settings
│   ├── tasks.json       # DotEnv tasks
│   └── launch.json      # Debug configurations
└── environments/
    ├── development.json  # Environment configs
    ├── staging.json
    └── production.json
```

### 2. Team Settings

Share VS Code configuration:

```json
// .vscode/settings.json (committed to repo)
{
    "dotenv.project": "${workspaceFolderBasename}",
    "dotenv.environments": ["development", "staging", "production"],
    "recommendations": ["dotenv.vscode-dotenv"]
}
```

### 3. Development Workflow

```json
// .vscode/tasks.json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Setup Development",
            "dependsOrder": "sequence",
            "dependsOn": [
                "Install Dependencies",
                "Pull DotEnv Secrets",
                "Start Development Server"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            }
        }
    ]
}
```

## Examples

### Complete Setup

```json
// .vscode/settings.json
{
    // DotEnv settings
    "dotenv.project": "my-app",
    "dotenv.defaultEnvironment": "development",
    "dotenv.autoSync": true,
    "dotenv.syncOnSave": true,

    // Editor settings
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
        "source.fixAll": true
    },

    // File associations
    "files.associations": {
        ".env*": "dotenv"
    },

    // Exclude from search
    "search.exclude": {
        ".env": true,
        ".env.local": true
    },

    // Git settings
    "git.ignoreLimitWarning": true
}
```

### Extension Recommendations

```json
// .vscode/extensions.json
{
    "recommendations": [
        "dotenv.vscode-dotenv",
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode",
        "streetsidesoftware.code-spell-checker"
    ],
    "unwantedRecommendations": []
}
```

## Resources

- [VS Code Extension Marketplace](https://marketplace.visualstudio.com/items?itemName=dotenv.vscode-dotenv)
- [Extension Documentation](https://github.com/dotenv/vscode-dotenv)
- [VS Code API](https://code.visualstudio.com/api)
- [DotEnv CLI Reference](/documentation/v1/cli/commands)

## Next Steps

- [Install the VS Code extension](#installation)
- [Configure your workspace](#workspace-configuration)
- [Set up team settings](#team-settings)
- [Create custom tasks](#tasks-integration)
