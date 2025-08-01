---
title: Monorepo Management
slug: monorepo
order: 2
tags: [use-cases, monorepo, management, patterns]
---

# Monorepo Management

Learn how to effectively manage secrets across monorepo architectures using DotEnv. This guide covers strategies for organizing secrets, handling shared dependencies, and maintaining consistency across multiple applications and packages.

## Overview

Monorepo challenges for secret management:

- Multiple applications in one repository
- Shared packages and libraries
- Environment consistency
- Developer onboarding
- CI/CD complexity
- Secret inheritance patterns

## Monorepo Structure

### Project Organization

```
my-monorepo/
├── apps/
│   ├── web/              # Main web application
│   ├── admin/            # Admin dashboard
│   ├── mobile-api/       # Mobile API backend
│   └── worker/           # Background job processor
├── packages/
│   ├── shared/           # Shared utilities
│   ├── ui/               # UI component library
│   ├── database/         # Database models
│   └── auth/             # Authentication logic
├── services/
│   ├── email/            # Email service
│   └── storage/          # File storage service
└── tools/
    ├── scripts/          # Build scripts
    └── cli/              # Internal CLI tools
```

### DotEnv Project Mapping

```yaml
# .dotenv-config.yaml
organization: my-company
projects:
    # Applications
    - name: web-app
      path: apps/web
      environments: [development, staging, production]

    - name: admin-app
      path: apps/admin
      environments: [development, staging, production]

    - name: mobile-api
      path: apps/mobile-api
      environments: [development, staging, production]

    # Services
    - name: email-service
      path: services/email
      environments: [development, staging, production]

    # Shared configuration
    - name: shared-config
      path: ./
      environments: [development, staging, production]
```

## Implementation Patterns

### 1. Root Configuration

```javascript
// dotenv.config.js
const path = require("path");
const { execSync } = require("child_process");

class MonorepoSecrets {
    constructor() {
        this.root = process.cwd();
        this.environment = process.env.NODE_ENV || "development";
    }

    async loadForPackage(packageName) {
        const packagePath = this.getPackagePath(packageName);
        const projectName = this.getProjectName(packageName);

        // Load shared secrets first
        await this.loadSharedSecrets();

        // Load package-specific secrets
        execSync(`dotenv pull ${this.environment} --project ${projectName}`, {
            cwd: packagePath,
            stdio: "inherit",
        });

        // Merge with process.env
        require("dotenv").config({
            path: path.join(packagePath, ".env"),
        });
    }

    async loadSharedSecrets() {
        execSync(
            `dotenv pull ${this.environment} --project shared-config --output .env.shared`,
            {
                cwd: this.root,
                stdio: "inherit",
            },
        );

        require("dotenv").config({
            path: path.join(this.root, ".env.shared"),
        });
    }

    getPackagePath(packageName) {
        // Resolve package location
        if (packageName.startsWith("@app/")) {
            return path.join(this.root, "apps", packageName.slice(5));
        }
        if (packageName.startsWith("@pkg/")) {
            return path.join(this.root, "packages", packageName.slice(5));
        }
        return path.join(this.root, packageName);
    }

    getProjectName(packageName) {
        // Map package names to DotEnv projects
        const mapping = {
            "@app/web": "web-app",
            "@app/admin": "admin-app",
            "@app/mobile-api": "mobile-api",
            "@service/email": "email-service",
        };

        return mapping[packageName] || packageName;
    }
}

module.exports = MonorepoSecrets;
```

### 2. Package Configuration

```json
// apps/web/package.json
{
    "name": "@app/web",
    "version": "1.0.0",
    "scripts": {
        "dev": "dotenv-load && next dev",
        "build": "dotenv-load && next build",
        "start": "dotenv-load && next start",
        "dotenv-load": "node ../../scripts/load-secrets.js @app/web"
    },
    "dependencies": {
        "@pkg/shared": "workspace:*",
        "@pkg/ui": "workspace:*",
        "@pkg/auth": "workspace:*"
    }
}
```

### 3. Shared Package Secrets

```typescript
// packages/database/src/config.ts
export interface DatabaseConfig {
    host: string;
    port: number;
    database: string;
    username: string;
    password: string;
    pool: {
        min: number;
        max: number;
    };
}

export function getDatabaseConfig(): DatabaseConfig {
    // These come from shared-config project
    return {
        host: process.env.DB_HOST || "localhost",
        port: parseInt(process.env.DB_PORT || "5432"),
        database: process.env.DB_NAME || "myapp",
        username: process.env.DB_USER || "postgres",
        password: process.env.DB_PASSWORD || "",
        pool: {
            min: parseInt(process.env.DB_POOL_MIN || "2"),
            max: parseInt(process.env.DB_POOL_MAX || "10"),
        },
    };
}
```

## Build Configuration

### Turborepo Setup

```json
// turbo.json
{
    "$schema": "https://turbo.build/schema.json",
    "pipeline": {
        "build": {
            "dependsOn": ["^build", "dotenv:load"],
            "outputs": ["dist/**", ".next/**"]
        },
        "dotenv:load": {
            "cache": false,
            "env": ["DOTENV_API_KEY", "DOTENV_ENVIRONMENT", "NODE_ENV"]
        },
        "dev": {
            "cache": false,
            "persistent": true,
            "dependsOn": ["dotenv:load"]
        },
        "test": {
            "dependsOn": ["dotenv:load"],
            "env": ["NODE_ENV"]
        }
    }
}
```

### Nx Configuration

```typescript
// tools/executors/dotenv/impl.ts
import { ExecutorContext } from "@nrwl/devkit";
import { execSync } from "child_process";

export interface DotEnvExecutorOptions {
    project: string;
    environment?: string;
    output?: string;
}

export default async function dotEnvExecutor(
    options: DotEnvExecutorOptions,
    context: ExecutorContext,
): Promise<{ success: boolean }> {
    const projectRoot = context.workspace.projects[context.projectName].root;
    const environment =
        options.environment || process.env.NODE_ENV || "development";

    try {
        execSync(
            `dotenv pull ${environment} --project ${options.project} --output ${options.output || ".env"}`,
            {
                cwd: projectRoot,
                stdio: "inherit",
            },
        );

        return { success: true };
    } catch (error) {
        console.error(`Failed to load secrets for ${options.project}`, error);
        return { success: false };
    }
}
```

```json
// workspace.json
{
    "projects": {
        "web-app": {
            "targets": {
                "dotenv": {
                    "executor": "./tools/executors/dotenv:default",
                    "options": {
                        "project": "web-app"
                    }
                },
                "serve": {
                    "executor": "@nrwl/next:server",
                    "dependsOn": ["dotenv"]
                }
            }
        }
    }
}
```

## Development Workflow

### 1. Initial Setup Script

```bash
#!/bin/bash
# scripts/setup-dev.sh

echo "🔐 Setting up DotEnv for monorepo development..."

# Check for API key
if [ -z "$DOTENV_API_KEY" ]; then
  echo "Please set DOTENV_API_KEY environment variable"
  exit 1
fi

# Authenticate
dotenv auth login --api-key $DOTENV_API_KEY

# Load secrets for all applications
apps=("web-app" "admin-app" "mobile-api")
services=("email-service" "storage-service")

echo "Loading shared configuration..."
dotenv pull development --project shared-config

for app in "${apps[@]}"; do
  echo "Loading secrets for $app..."
  cd "apps/${app#*-}" 2>/dev/null || continue
  dotenv pull development --project $app
  cd - > /dev/null
done

for service in "${services[@]}"; do
  echo "Loading secrets for $service..."
  cd "services/${service%-*}" 2>/dev/null || continue
  dotenv pull development --project $service
  cd - > /dev/null
done

echo "✅ Development environment ready!"
```

### 2. Package Scripts

```typescript
// scripts/dotenv-sync.ts
import { execSync } from "child_process";
import { readdirSync, statSync } from "fs";
import { join } from "path";

interface PackageInfo {
    name: string;
    path: string;
    dotenvProject?: string;
}

function findPackages(dir: string): PackageInfo[] {
    const packages: PackageInfo[] = [];

    const scanDirectory = (currentDir: string) => {
        const items = readdirSync(currentDir);

        for (const item of items) {
            const fullPath = join(currentDir, item);

            if (statSync(fullPath).isDirectory()) {
                if (item === "node_modules") continue;

                try {
                    const packageJson = require(join(fullPath, "package.json"));
                    if (packageJson.dotenv) {
                        packages.push({
                            name: packageJson.name,
                            path: fullPath,
                            dotenvProject: packageJson.dotenv.project,
                        });
                    }
                } catch {
                    // Not a package directory, continue scanning
                    scanDirectory(fullPath);
                }
            }
        }
    };

    scanDirectory(dir);
    return packages;
}

async function syncAllSecrets() {
    const packages = findPackages(process.cwd());
    const environment = process.env.NODE_ENV || "development";

    console.log(`Found ${packages.length} packages with DotEnv configuration`);

    for (const pkg of packages) {
        console.log(`\nSyncing ${pkg.name}...`);

        try {
            execSync(
                `dotenv pull ${environment} --project ${pkg.dotenvProject}`,
                {
                    cwd: pkg.path,
                    stdio: "inherit",
                },
            );
        } catch (error) {
            console.error(`Failed to sync ${pkg.name}:`, error.message);
        }
    }
}

syncAllSecrets().catch(console.error);
```

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/monorepo-ci.yml
name: Monorepo CI

on:
    push:
        branches: [main, develop]
    pull_request:

env:
    TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
    TURBO_TEAM: ${{ secrets.TURBO_TEAM }}

jobs:
    determine-changes:
        runs-on: ubuntu-latest
        outputs:
            apps: ${{ steps.filter.outputs.changes }}
        steps:
            - uses: actions/checkout@v3
            - uses: dorny/paths-filter@v2
              id: filter
              with:
                  filters: |
                      web-app:
                        - 'apps/web/**'
                        - 'packages/**'
                      admin-app:
                        - 'apps/admin/**'
                        - 'packages/**'
                      mobile-api:
                        - 'apps/mobile-api/**'
                        - 'packages/**'

    build-and-test:
        needs: determine-changes
        strategy:
            matrix:
                app: ${{ fromJson(needs.determine-changes.outputs.apps) }}
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v3

            - name: Setup Node.js
              uses: actions/setup-node@v3
              with:
                  node-version: 18
                  cache: "npm"

            - name: Install DotEnv CLI
              run: |
                  curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                  echo "$HOME/.dotenv/bin" >> $GITHUB_PATH

            - name: Load secrets for app
              env:
                  DOTENV_API_KEY: ${{ secrets.DOTENV_API_KEY }}
              run: |
                  dotenv auth login --api-key $DOTENV_API_KEY

                  # Load shared secrets
                  dotenv pull ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }} \
                    --project shared-config

                  # Load app-specific secrets
                  cd apps/${{ matrix.app }}
                  dotenv pull ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }} \
                    --project ${{ matrix.app }}

            - name: Install dependencies
              run: npm ci

            - name: Build packages
              run: npx turbo run build --filter=...${{ matrix.app }}

            - name: Test
              run: npx turbo run test --filter=${{ matrix.app }}
```

### Secret Inheritance

```javascript
// tools/dotenv/inherit.js
const yaml = require("js-yaml");
const fs = require("fs");

class SecretInheritance {
    constructor(configPath = ".dotenv-inheritance.yaml") {
        this.config = yaml.load(fs.readFileSync(configPath, "utf8"));
    }

    async resolveSecrets(projectName, environment) {
        const project = this.config.projects[projectName];
        if (!project) {
            throw new Error(`Project ${projectName} not found in config`);
        }

        const secrets = {};

        // Load inherited secrets first
        if (project.inherits) {
            for (const parent of project.inherits) {
                const parentSecrets = await this.loadSecrets(
                    parent,
                    environment,
                );
                Object.assign(secrets, parentSecrets);
            }
        }

        // Load project-specific secrets (override inherited)
        const projectSecrets = await this.loadSecrets(projectName, environment);
        Object.assign(secrets, projectSecrets);

        // Apply transformations if defined
        if (project.transform) {
            for (const [key, transform] of Object.entries(project.transform)) {
                if (secrets[transform.from]) {
                    secrets[key] = this.applyTransform(
                        secrets[transform.from],
                        transform,
                    );
                }
            }
        }

        return secrets;
    }

    async loadSecrets(projectName, environment) {
        // Implementation to load from DotEnv
        const { execSync } = require("child_process");
        const output = execSync(
            `dotenv pull ${environment} --project ${projectName} --format json`,
            { encoding: "utf8" },
        );
        return JSON.parse(output);
    }

    applyTransform(value, transform) {
        switch (transform.type) {
            case "prefix":
                return `${transform.value}${value}`;
            case "suffix":
                return `${value}${transform.value}`;
            case "replace":
                return value.replace(transform.pattern, transform.replacement);
            default:
                return value;
        }
    }
}

// .dotenv-inheritance.yaml
/*
projects:
  web-app:
    inherits:
      - shared-config
      - web-common
    transform:
      PUBLIC_API_URL:
        from: API_BASE_URL
        type: suffix
        value: /public
  
  admin-app:
    inherits:
      - shared-config
      - admin-common
    transform:
      ADMIN_API_URL:
        from: API_BASE_URL
        type: suffix
        value: /admin
*/
```

## Testing Strategy

### Environment Isolation

```typescript
// packages/testing/src/test-env.ts
import { execSync } from "child_process";
import { randomBytes } from "crypto";

export class TestEnvironment {
    private testId: string;
    private originalEnv: NodeJS.ProcessEnv;

    constructor() {
        this.testId = randomBytes(8).toString("hex");
        this.originalEnv = { ...process.env };
    }

    async setup(projectName: string) {
        // Create isolated test environment
        const testEnvName = `test-${this.testId}`;

        // Clone from development environment
        execSync(
            `dotenv env create ${testEnvName} --from development --project ${projectName}`,
            { stdio: "inherit" },
        );

        // Load test secrets
        execSync(`dotenv pull ${testEnvName} --project ${projectName}`, {
            stdio: "inherit",
        });

        // Override specific values for testing
        process.env.DATABASE_URL = `postgresql://test:test@localhost:5432/test_${this.testId}`;
        process.env.REDIS_URL = `redis://localhost:6379/${this.testId}`;
    }

    async teardown(projectName: string) {
        // Restore original environment
        process.env = this.originalEnv;

        // Clean up test environment
        const testEnvName = `test-${this.testId}`;
        execSync(
            `dotenv env delete ${testEnvName} --project ${projectName} --yes`,
            { stdio: "inherit" },
        );
    }
}
```

### Jest Configuration

```javascript
// jest.config.base.js
module.exports = {
  setupFilesAfterEnv: ['<rootDir>/../../tools/testing/jest-setup.js'],
  testEnvironment: 'node',
  testMatch: ['**/*.test.ts', '**/*.spec.ts'],
  transform: {
    '^.+\\.tsx?$': 'ts-jest'
  }
};

// apps/web/jest.config.js
const base = require('../../jest.config.base');

module.exports = {
  ...base,
  displayName: 'web-app',
  globalSetup: '<rootDir>/test/global-setup.ts',
  globalTeardown: '<rootDir>/test/global-teardown.ts'
};

// apps/web/test/global-setup.ts
import { TestEnvironment } from '@pkg/testing';

export default async function globalSetup() {
  const testEnv = new TestEnvironment();
  await testEnv.setup('web-app');

  // Store reference for teardown
  (global as any).__TEST_ENV__ = testEnv;
}
```

## Security Considerations

### 1. Access Control

```yaml
# .dotenv-permissions.yaml
permissions:
    # Developers get access to development/staging
    developers:
        projects:
            - name: "*"
              environments: [development, staging]
              permissions: [read]

    # Senior developers can write to staging
    senior-developers:
        projects:
            - name: "*"
              environments: [development, staging]
              permissions: [read, write]

    # DevOps team has production access
    devops:
        projects:
            - name: "*"
              environments: [development, staging, production]
              permissions: [read, write, admin]

    # Service accounts for CI/CD
    ci-service:
        projects:
            - name: "*"
              environments: [staging, production]
              permissions: [read]
```

### 2. Git Hooks

```bash
#!/bin/bash
# .husky/pre-commit

# Check for .env files
env_files=$(git diff --cached --name-only | grep -E '\.env(\..+)?$')

if [ -n "$env_files" ]; then
  echo "❌ Error: Attempting to commit .env files:"
  echo "$env_files"
  echo ""
  echo "Remove them with: git reset HEAD <file>"
  exit 1
fi

# Scan for potential secrets
if command -v gitleaks &> /dev/null; then
  gitleaks protect --staged --verbose
fi
```

## Best Practices

### 1. Package Organization

```json
// package.json fields for DotEnv integration
{
    "name": "@app/web",
    "dotenv": {
        "project": "web-app",
        "inherits": ["shared-config"],
        "required": ["DATABASE_URL", "REDIS_URL", "API_KEY"]
    }
}
```

### 2. Environment Validation

```typescript
// packages/shared/src/env-validator.ts
import { z } from "zod";

export function validateEnvironment<T extends z.ZodRawShape>(
    schema: z.ZodObject<T>,
): z.infer<z.ZodObject<T>> {
    try {
        return schema.parse(process.env);
    } catch (error) {
        console.error("❌ Invalid environment variables:");
        console.error(error.errors);
        process.exit(1);
    }
}

// Usage in apps/web/src/env.ts
const envSchema = z.object({
    NODE_ENV: z.enum(["development", "test", "production"]),
    DATABASE_URL: z.string().url(),
    REDIS_URL: z.string().url(),
    API_KEY: z.string().min(32),
    PORT: z.string().transform(Number).default("3000"),
});

export const env = validateEnvironment(envSchema);
```

## Troubleshooting

### Common Issues

1. **Circular Dependencies**

    ```bash
    # Check dependency graph
    npx madge --circular --extensions ts,tsx,js,jsx packages/
    ```

2. **Secret Loading Order**

    ```javascript
    // Ensure shared secrets load first
    await loadSecrets("shared-config");
    await loadSecrets("app-specific");
    ```

3. **Build Cache Issues**

    ```bash
    # Clear Turborepo cache
    npx turbo daemon clean
    rm -rf .turbo

    # Rebuild with fresh secrets
    npx turbo run build --force
    ```

## Resources

- [Monorepo Tools Comparison](https://monorepo.tools/)
- [Turborepo Documentation](https://turbo.build/repo/docs)
- [Nx Documentation](https://nx.dev/)
- [DotEnv Project Management](/documentation/v1/core-concepts/organizations-projects)

## Next Steps

- [Set up monorepo structure](#monorepo-structure)
- [Configure build tools](#build-configuration)
- [Implement CI/CD](#cicd-integration)
- [Set up development workflow](#development-workflow)
