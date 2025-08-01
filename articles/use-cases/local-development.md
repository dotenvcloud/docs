---
title: Local Development
slug: local-development
order: 6
tags: [use-cases, development, local, workflows]
---

# Local Development

Learn how to set up and manage local development environments using DotEnv. This guide covers best practices for developer productivity, environment consistency, and seamless integration with your development workflow.

## Overview

Local development challenges:

- Environment variable management
- Secrets security on developer machines
- Team consistency
- Quick environment switching
- Integration with IDEs and tools
- Offline development support

## Initial Setup

### 1. Developer Onboarding

```bash
#!/bin/bash
# scripts/dev-onboard.sh

echo "🚀 DotEnv Local Development Setup"
echo "================================"

# Check prerequisites
check_prerequisites() {
  local missing=()

  command -v git >/dev/null 2>&1 || missing+=("git")
  command -v node >/dev/null 2>&1 || missing+=("node")
  command -v docker >/dev/null 2>&1 || missing+=("docker")

  if [ ${#missing[@]} -ne 0 ]; then
    echo "❌ Missing prerequisites: ${missing[*]}"
    echo "Please install them before continuing."
    exit 1
  fi
}

# Install DotEnv CLI
install_dotenv_cli() {
  if ! command -v dotenv >/dev/null 2>&1; then
    echo "Installing DotEnv CLI..."
    curl -fsSL https://cli.dotenv.cloud/install.sh | sh
    echo 'export PATH="$HOME/.dotenv/bin:$PATH"' >> ~/.bashrc
    echo 'export PATH="$HOME/.dotenv/bin:$PATH"' >> ~/.zshrc
    export PATH="$HOME/.dotenv/bin:$PATH"
  fi
}

# Authenticate
authenticate() {
  echo "Authenticating with DotEnv..."
  dotenv auth login
}

# Set up local environment
setup_environment() {
  echo "Setting up local development environment..."

  # Create local config directory
  mkdir -p ~/.dotenv/local

  # Set default environment
  echo "development" > ~/.dotenv/default-environment

  # Configure CLI
  cat > ~/.dotenv/config.yaml << EOF
default_environment: development
auto_sync: true
offline_mode: false
cache_duration: 3600
security:
  encrypt_local_cache: true
  require_mfa: false
EOF
}

# Main execution
check_prerequisites
install_dotenv_cli
authenticate
setup_environment

echo "✅ Local development setup complete!"
```

### 2. Project Configuration

```yaml
# .dotenv.yaml
project: my-app
organization: my-company

environments:
    - name: local
      description: Local development
      parent: development

    - name: development
      description: Shared development

    - name: staging
      description: Staging environment

    - name: production
      description: Production environment
      restricted: true

local_development:
    default_environment: local
    auto_pull: true
    cache_secrets: true
    sync_on_change: true

required_tools:
    - name: node
      version: ">=18.0.0"
    - name: docker
      version: ">=20.0.0"
    - name: postgresql
      version: ">=14.0"
```

## Environment Management

### 1. Environment Switcher

```javascript
// tools/env-switch.js
const { execSync } = require("child_process");
const inquirer = require("inquirer");
const chalk = require("chalk");
const fs = require("fs");

class EnvironmentSwitcher {
    constructor() {
        this.configPath = "./.dotenv.yaml";
        this.currentEnvPath = "./.env.current";
    }

    async getCurrentEnvironment() {
        try {
            return fs.readFileSync(this.currentEnvPath, "utf8").trim();
        } catch {
            return "development";
        }
    }

    async getAvailableEnvironments() {
        try {
            const output = execSync("dotenv env list --format json", {
                encoding: "utf8",
            });
            return JSON.parse(output);
        } catch (error) {
            console.error("Failed to get environments:", error.message);
            return [];
        }
    }

    async switchEnvironment() {
        const current = await this.getCurrentEnvironment();
        const environments = await this.getAvailableEnvironments();

        console.log(chalk.blue(`Current environment: ${current}`));

        const { environment } = await inquirer.prompt([
            {
                type: "list",
                name: "environment",
                message: "Select environment:",
                choices: environments.map((env) => ({
                    name: `${env.name} - ${env.description}`,
                    value: env.name,
                })),
            },
        ]);

        if (environment === current) {
            console.log(chalk.yellow("Already on this environment"));
            return;
        }

        // Backup current environment
        if (fs.existsSync(".env")) {
            fs.copyFileSync(".env", `.env.backup.${current}`);
        }

        // Switch environment
        console.log(chalk.green(`Switching to ${environment}...`));

        try {
            execSync(`dotenv pull ${environment}`, { stdio: "inherit" });
            fs.writeFileSync(this.currentEnvPath, environment);

            // Restart services if needed
            await this.restartServices();

            console.log(chalk.green(`✅ Switched to ${environment}`));
        } catch (error) {
            console.error(
                chalk.red("Failed to switch environment:"),
                error.message,
            );

            // Restore backup
            if (fs.existsSync(`.env.backup.${current}`)) {
                fs.copyFileSync(`.env.backup.${current}`, ".env");
            }
        }
    }

    async restartServices() {
        const services = [
            { name: "webpack-dev-server", command: "npm run dev:restart" },
            { name: "database", command: "docker-compose restart db" },
            { name: "redis", command: "docker-compose restart redis" },
        ];

        for (const service of services) {
            try {
                console.log(chalk.gray(`Restarting ${service.name}...`));
                execSync(service.command, { stdio: "ignore" });
            } catch {
                // Service might not be running
            }
        }
    }
}

// CLI usage
if (require.main === module) {
    const switcher = new EnvironmentSwitcher();
    switcher.switchEnvironment().catch(console.error);
}

module.exports = EnvironmentSwitcher;
```

### 2. Git Integration

```bash
# .git/hooks/post-checkout
#!/bin/bash

# Auto-switch environment based on branch
branch=$(git rev-parse --abbrev-ref HEAD)

case "$branch" in
  "main")
    environment="production"
    ;;
  "staging")
    environment="staging"
    ;;
  "develop")
    environment="development"
    ;;
  feature/*)
    environment="local"
    ;;
  *)
    environment="development"
    ;;
esac

echo "Branch: $branch → Environment: $environment"

# Only switch if different from current
current=$(cat .env.current 2>/dev/null || echo "none")
if [ "$current" != "$environment" ]; then
  dotenv pull $environment
  echo "$environment" > .env.current
fi
```

## IDE Integration

### 1. VS Code Extension Configuration

```json
// .vscode/settings.json
{
    "dotenv.defaultEnvironment": "local",
    "dotenv.autoSync": true,
    "dotenv.showInlineValues": true,
    "dotenv.enableCodeLens": true,

    // Environment-specific launch configs
    "launch": {
        "version": "0.2.0",
        "configurations": [
            {
                "type": "node",
                "request": "launch",
                "name": "Debug (Local)",
                "program": "${workspaceFolder}/src/index.js",
                "envFile": "${workspaceFolder}/.env",
                "preLaunchTask": "dotenv-sync-local"
            },
            {
                "type": "node",
                "request": "launch",
                "name": "Debug (Development)",
                "program": "${workspaceFolder}/src/index.js",
                "preLaunchTask": "dotenv-sync-development"
            }
        ]
    },

    // Tasks for environment sync
    "tasks": {
        "version": "2.0.0",
        "tasks": [
            {
                "label": "dotenv-sync-local",
                "type": "shell",
                "command": "dotenv pull local"
            },
            {
                "label": "dotenv-sync-development",
                "type": "shell",
                "command": "dotenv pull development"
            }
        ]
    }
}
```

### 2. IntelliJ IDEA Configuration

```xml
<!-- .idea/runConfigurations/Local_Development.xml -->
<component name="ProjectRunConfigurationManager">
  <configuration default="false" name="Local Development" type="Application">
    <envs>
      <env name="DOTENV_ENVIRONMENT" value="local" />
    </envs>
    <option name="MAIN_CLASS_NAME" value="com.example.Application" />
    <option name="PROGRAM_PARAMETERS" value="--spring.profiles.active=local" />
    <option name="WORKING_DIRECTORY" value="$PROJECT_DIR$" />
    <method v="2">
      <option name="RunConfigurationTask" enabled="true"
              run_configuration_name="Sync DotEnv"
              run_configuration_type="ShellScriptRunConfiguration" />
    </method>
  </configuration>
</component>

<!-- .idea/runConfigurations/Sync_DotEnv.xml -->
<component name="ProjectRunConfigurationManager">
  <configuration default="false" name="Sync DotEnv" type="ShellScriptRunConfiguration">
    <option name="SCRIPT_TEXT" value="dotenv pull $DOTENV_ENVIRONMENT" />
    <option name="SCRIPT_PATH" value="" />
    <option name="SCRIPT_OPTIONS" value="" />
    <option name="WORKING_DIRECTORY" value="$PROJECT_DIR$" />
  </configuration>
</component>
```

## Local Services

### 1. Docker Compose Integration

```yaml
# docker-compose.local.yml
version: "3.8"

services:
    app:
        build: .
        ports:
            - "3000:3000"
        env_file:
            - .env
        environment:
            - NODE_ENV=development
            - DOTENV_ENVIRONMENT=local
        volumes:
            - .:/app
            - /app/node_modules
        command: npm run dev
        depends_on:
            - db
            - redis
            - maildev

    db:
        image: postgres:14
        ports:
            - "5432:5432"
        environment:
            POSTGRES_DB: ${DB_NAME:-myapp_dev}
            POSTGRES_USER: ${DB_USER:-postgres}
            POSTGRES_PASSWORD: ${DB_PASSWORD:-postgres}
        volumes:
            - postgres_data:/var/lib/postgresql/data

    redis:
        image: redis:7-alpine
        ports:
            - "6379:6379"
        command: redis-server --requirepass ${REDIS_PASSWORD:-local_redis_pass}

    maildev:
        image: maildev/maildev
        ports:
            - "1080:1080" # Web UI
            - "1025:1025" # SMTP

volumes:
    postgres_data:
```

### 2. Local Service Manager

```javascript
// tools/local-services.js
const { spawn } = require("child_process");
const chalk = require("chalk");

class LocalServiceManager {
    constructor() {
        this.services = {
            database: {
                name: "PostgreSQL",
                check: "pg_isready -h localhost -p 5432",
                start: "docker-compose up -d db",
                stop: "docker-compose stop db",
                logs: "docker-compose logs -f db",
            },
            redis: {
                name: "Redis",
                check: "redis-cli -h localhost -p 6379 ping",
                start: "docker-compose up -d redis",
                stop: "docker-compose stop redis",
                logs: "docker-compose logs -f redis",
            },
            mail: {
                name: "MailDev",
                check: "curl -s http://localhost:1080 > /dev/null",
                start: "docker-compose up -d maildev",
                stop: "docker-compose stop maildev",
                logs: "docker-compose logs -f maildev",
            },
        };
    }

    async checkService(serviceName) {
        return new Promise((resolve) => {
            spawn(this.services[serviceName].check, [], {
                shell: true,
                stdio: "ignore",
            }).on("exit", (code) => {
                resolve(code === 0);
            });
        });
    }

    async startService(serviceName) {
        const service = this.services[serviceName];
        console.log(chalk.blue(`Starting ${service.name}...`));

        return new Promise((resolve, reject) => {
            const proc = spawn(service.start, [], {
                shell: true,
                stdio: "inherit",
            });

            proc.on("exit", (code) => {
                if (code === 0) {
                    console.log(chalk.green(`✅ ${service.name} started`));
                    resolve();
                } else {
                    reject(new Error(`Failed to start ${service.name}`));
                }
            });
        });
    }

    async startAll() {
        console.log(chalk.bold("Starting local services..."));

        for (const [name, service] of Object.entries(this.services)) {
            const isRunning = await this.checkService(name);

            if (isRunning) {
                console.log(chalk.yellow(`${service.name} already running`));
            } else {
                await this.startService(name);
            }
        }

        console.log(chalk.green("\n✅ All services ready!"));
        console.log(chalk.gray("\nService URLs:"));
        console.log("  Database: postgresql://localhost:5432");
        console.log("  Redis: redis://localhost:6379");
        console.log("  Mail UI: http://localhost:1080");
    }

    async stopAll() {
        console.log(chalk.bold("Stopping local services..."));

        for (const [name, service] of Object.entries(this.services)) {
            console.log(chalk.blue(`Stopping ${service.name}...`));
            spawn(service.stop, [], { shell: true, stdio: "ignore" });
        }

        console.log(chalk.green("✅ All services stopped"));
    }
}

// CLI commands
const manager = new LocalServiceManager();
const command = process.argv[2];

switch (command) {
    case "start":
        manager.startAll().catch(console.error);
        break;
    case "stop":
        manager.stopAll().catch(console.error);
        break;
    default:
        console.log("Usage: node local-services.js [start|stop]");
}
```

## Development Workflows

### 1. Feature Development

```bash
#!/bin/bash
# scripts/start-feature.sh

feature_name=$1

if [ -z "$feature_name" ]; then
  echo "Usage: ./start-feature.sh <feature-name>"
  exit 1
fi

echo "🚀 Starting feature: $feature_name"

# Create feature branch
git checkout -b "feature/$feature_name"

# Create local environment
dotenv env create "local-$feature_name" --from local

# Set up feature-specific secrets
cat > .env.feature << EOF
# Feature: $feature_name
# Created: $(date)
FEATURE_NAME=$feature_name
FEATURE_FLAGS={"$feature_name": true}
EOF

# Load environment
dotenv pull "local-$feature_name"
cat .env.feature >> .env

# Start services
docker-compose -f docker-compose.local.yml up -d

echo "✅ Feature environment ready!"
echo ""
echo "Next steps:"
echo "1. Start development server: npm run dev"
echo "2. View logs: docker-compose logs -f"
echo "3. Access database: psql -h localhost -U postgres myapp_dev"
```

### 2. Secret Templates

```javascript
// config/secret-templates.js
const templates = {
    development: {
        // Database
        DATABASE_URL: "postgresql://postgres:postgres@localhost:5432/myapp_dev",
        DATABASE_POOL_MIN: "2",
        DATABASE_POOL_MAX: "10",

        // Redis
        REDIS_URL: "redis://:local_redis_pass@localhost:6379/0",

        // API Keys (development)
        STRIPE_KEY: "sk_test_...",
        SENDGRID_KEY: "SG.test...",

        // Feature flags
        FEATURE_FLAGS: JSON.stringify({
            debug_mode: true,
            verbose_logging: true,
            mock_external_apis: true,
        }),

        // URLs
        API_URL: "http://localhost:3000",
        FRONTEND_URL: "http://localhost:3001",

        // Security (relaxed for development)
        JWT_SECRET: "development-secret-key",
        SESSION_SECRET: "development-session-secret",
        CORS_ORIGINS: "http://localhost:3000,http://localhost:3001",
    },

    test: {
        DATABASE_URL:
            "postgresql://postgres:postgres@localhost:5432/myapp_test",
        REDIS_URL: "redis://localhost:6379/1",
        JWT_SECRET: "test-secret-key",
        MOCK_EXTERNAL_APIS: "true",
    },
};

// Generate .env.example
function generateEnvExample() {
    const example = Object.keys(templates.development)
        .map((key) => `${key}=`)
        .join("\n");

    require("fs").writeFileSync(".env.example", example);
}

module.exports = { templates, generateEnvExample };
```

## Testing Integration

### 1. Test Environment Setup

```javascript
// test/setup/env.js
const { execSync } = require("child_process");
const fs = require("fs");
const path = require("path");

class TestEnvironment {
    static async setup() {
        // Create test environment if it doesn't exist
        const testEnvName = `test-${process.pid}`;

        try {
            // Create isolated test environment
            execSync(`dotenv env create ${testEnvName} --from test`, {
                stdio: "ignore",
            });

            // Load test secrets
            execSync(`dotenv pull ${testEnvName} --output .env.test`, {
                stdio: "ignore",
            });

            // Override with test-specific values
            const testOverrides = {
                NODE_ENV: "test",
                DATABASE_URL: `postgresql://localhost:5432/test_${process.pid}`,
                REDIS_URL: `redis://localhost:6379/${process.pid % 16}`,
                LOG_LEVEL: "error",
                MOCK_EXTERNAL_APIS: "true",
            };

            // Apply overrides
            const env = this.parseEnvFile(".env.test");
            Object.assign(env, testOverrides);

            // Write final test environment
            const envContent = Object.entries(env)
                .map(([key, value]) => `${key}=${value}`)
                .join("\n");

            fs.writeFileSync(".env.test", envContent);

            // Set process.env
            Object.assign(process.env, env);

            // Store cleanup info
            process.env.__TEST_ENV_NAME__ = testEnvName;
        } catch (error) {
            console.error("Failed to setup test environment:", error);
            throw error;
        }
    }

    static async teardown() {
        const testEnvName = process.env.__TEST_ENV_NAME__;

        if (testEnvName) {
            try {
                // Clean up test environment
                execSync(`dotenv env delete ${testEnvName} --yes`, {
                    stdio: "ignore",
                });
            } catch {
                // Ignore cleanup errors
            }
        }

        // Remove test files
        try {
            fs.unlinkSync(".env.test");
        } catch {
            // File might not exist
        }
    }

    static parseEnvFile(filePath) {
        const content = fs.readFileSync(filePath, "utf8");
        const env = {};

        content.split("\n").forEach((line) => {
            const match = line.match(/^([^=]+)=(.*)$/);
            if (match) {
                env[match[1]] = match[2];
            }
        });

        return env;
    }
}

module.exports = TestEnvironment;
```

### 2. Mock Service Integration

```javascript
// test/mocks/external-services.js
const nock = require("nock");

class ExternalServiceMocks {
    static setupAll() {
        // Only mock in test environment
        if (process.env.NODE_ENV !== "test") return;

        this.setupStripe();
        this.setupSendGrid();
        this.setupTwilio();
    }

    static setupStripe() {
        nock("https://api.stripe.com")
            .persist()
            .post("/v1/charges")
            .reply(200, {
                id: "ch_test_" + Date.now(),
                amount: 2000,
                currency: "usd",
                status: "succeeded",
            })
            .post("/v1/customers")
            .reply(200, {
                id: "cus_test_" + Date.now(),
                email: "test@example.com",
            });
    }

    static setupSendGrid() {
        nock("https://api.sendgrid.com")
            .persist()
            .post("/v3/mail/send")
            .reply(202, {
                message: "success",
            });
    }

    static setupTwilio() {
        nock("https://api.twilio.com")
            .persist()
            .post(/\/2010-04-01\/Accounts\/.*\/Messages\.json/)
            .reply(201, {
                sid: "SM" + Date.now(),
                status: "sent",
            });
    }

    static teardownAll() {
        nock.cleanAll();
    }
}

module.exports = ExternalServiceMocks;
```

## Debugging Tools

### 1. Environment Inspector

```javascript
// tools/env-inspector.js
const chalk = require("chalk");
const Table = require("cli-table3");

class EnvironmentInspector {
    inspect() {
        console.log(chalk.bold("\n🔍 Environment Inspection\n"));

        // Current environment
        this.showCurrentEnvironment();

        // Loaded variables
        this.showLoadedVariables();

        // Missing required variables
        this.checkRequiredVariables();

        // Security audit
        this.performSecurityAudit();
    }

    showCurrentEnvironment() {
        const env = process.env.DOTENV_ENVIRONMENT || "unknown";
        const project = process.env.DOTENV_PROJECT || "unknown";

        console.log(chalk.blue("Current Configuration:"));
        console.log(`  Environment: ${chalk.green(env)}`);
        console.log(`  Project: ${chalk.green(project)}`);
        console.log(`  Node Environment: ${chalk.green(process.env.NODE_ENV)}`);
        console.log("");
    }

    showLoadedVariables() {
        const table = new Table({
            head: ["Variable", "Value", "Source"],
            colWidths: [30, 40, 20],
        });

        const envVars = this.categorizeVariables();

        Object.entries(envVars).forEach(([category, vars]) => {
            table.push([chalk.bold(category), "", ""]);

            vars.forEach(({ key, value, source }) => {
                table.push([
                    `  ${key}`,
                    this.maskSensitiveValue(key, value),
                    source,
                ]);
            });
        });

        console.log(chalk.blue("Loaded Variables:"));
        console.log(table.toString());
        console.log("");
    }

    categorizeVariables() {
        const categories = {
            Database: ["DATABASE_URL", "DB_HOST", "DB_PORT", "DB_NAME"],
            Redis: ["REDIS_URL", "REDIS_HOST", "REDIS_PORT"],
            "API Keys": ["STRIPE_KEY", "SENDGRID_KEY", "AWS_ACCESS_KEY_ID"],
            URLs: ["API_URL", "FRONTEND_URL", "WEBHOOK_URL"],
            Security: ["JWT_SECRET", "SESSION_SECRET", "ENCRYPTION_KEY"],
            Features: ["FEATURE_FLAGS", "DEBUG_MODE", "LOG_LEVEL"],
        };

        const result = {};

        Object.entries(categories).forEach(([category, keys]) => {
            const vars = [];

            keys.forEach((key) => {
                if (process.env[key]) {
                    vars.push({
                        key,
                        value: process.env[key],
                        source: this.getVariableSource(key),
                    });
                }
            });

            if (vars.length > 0) {
                result[category] = vars;
            }
        });

        return result;
    }

    maskSensitiveValue(key, value) {
        const sensitivePatterns = [
            "SECRET",
            "KEY",
            "TOKEN",
            "PASSWORD",
            "PRIVATE",
        ];

        const isSensitive = sensitivePatterns.some((pattern) =>
            key.toUpperCase().includes(pattern),
        );

        if (isSensitive) {
            return (
                value.substring(0, 4) +
                "****" +
                value.substring(value.length - 4)
            );
        }

        // Truncate long values
        if (value.length > 40) {
            return value.substring(0, 37) + "...";
        }

        return value;
    }

    getVariableSource(key) {
        // Try to determine source
        if (
            fs.existsSync(".env.local") &&
            fs.readFileSync(".env.local", "utf8").includes(key)
        ) {
            return chalk.yellow("local");
        }

        if (
            fs.existsSync(".env") &&
            fs.readFileSync(".env", "utf8").includes(key)
        ) {
            return chalk.green("dotenv");
        }

        return chalk.gray("system");
    }

    checkRequiredVariables() {
        const required = [
            "DATABASE_URL",
            "REDIS_URL",
            "JWT_SECRET",
            "NODE_ENV",
        ];

        const missing = required.filter((key) => !process.env[key]);

        if (missing.length > 0) {
            console.log(chalk.red("⚠️  Missing Required Variables:"));
            missing.forEach((key) => {
                console.log(`  - ${key}`);
            });
            console.log("");
        } else {
            console.log(chalk.green("✅ All required variables present\n"));
        }
    }

    performSecurityAudit() {
        console.log(chalk.blue("Security Audit:"));

        const issues = [];

        // Check for hardcoded secrets
        if (process.env.JWT_SECRET === "development-secret-key") {
            issues.push("Using default development JWT secret");
        }

        // Check for exposed sensitive data
        const sensitiveKeys = Object.keys(process.env).filter(
            (key) => key.includes("SECRET") || key.includes("PRIVATE"),
        );

        sensitiveKeys.forEach((key) => {
            if (process.env[key].length < 32) {
                issues.push(`${key} appears to be weak (< 32 characters)`);
            }
        });

        if (issues.length > 0) {
            issues.forEach((issue) => {
                console.log(chalk.yellow(`  ⚠️  ${issue}`));
            });
        } else {
            console.log(chalk.green("  ✅ No security issues detected"));
        }
    }
}

// Run inspector
const inspector = new EnvironmentInspector();
inspector.inspect();
```

## Best Practices

### 1. Local Development Standards

```javascript
// .dotenv/local-standards.js
module.exports = {
    // Never commit these files
    gitignore: [
        ".env",
        ".env.*",
        "!.env.example",
        "!.env.test.example",
        ".dotenv/cache/",
        ".env.current",
    ],

    // Required local tools
    prerequisites: {
        "DotEnv CLI": "dotenv --version",
        "Node.js": "node --version",
        Docker: "docker --version",
        Git: "git --version",
    },

    // Development conventions
    conventions: {
        defaultEnvironment: "local",
        autoSync: true,
        cacheExpiry: 3600, // 1 hour
        offlineMode: false,
    },

    // Security policies
    security: {
        encryptLocalCache: true,
        requireStrongSecrets: false, // Relaxed for local
        allowDefaultValues: true,
        warnOnWeakSecrets: true,
    },
};
```

### 2. Team Synchronization

```bash
#!/bin/bash
# scripts/sync-team-env.sh

echo "🔄 Synchronizing team development environment..."

# Pull latest shared development secrets
dotenv pull development --output .env.shared

# Merge with local overrides
if [ -f .env.local ]; then
  echo "Preserving local overrides..."
  cp .env .env.backup
  cat .env.shared > .env

  # Apply local overrides
  while IFS= read -r line; do
    if [[ ! -z "$line" ]] && [[ ! "$line" =~ ^# ]]; then
      key=$(echo "$line" | cut -d'=' -f1)
      # Remove key from .env if it exists
      sed -i.bak "/^$key=/d" .env
      # Add override
      echo "$line" >> .env
    fi
  done < .env.local
fi

echo "✅ Environment synchronized"
```

## Troubleshooting

### Common Issues

1. **Port Conflicts**

    ```bash
    # Check for port usage
    lsof -i :3000
    lsof -i :5432

    # Kill process using port
    kill -9 $(lsof -t -i:3000)
    ```

2. **Docker Issues**

    ```bash
    # Reset Docker environment
    docker-compose down -v
    docker system prune -af
    docker-compose up -d
    ```

3. **Secret Loading Failures**
    ```bash
    # Clear cache and re-authenticate
    rm -rf ~/.dotenv/cache
    dotenv auth logout
    dotenv auth login
    ```

## Resources

- [Local Development Best Practices](https://12factor.net/dev-prod-parity)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Environment Variable Management](https://www.npmjs.com/package/dotenv)
- [DotEnv CLI Documentation](/documentation/v1/cli/overview)

## Next Steps

- [Set up local environment](#initial-setup)
- [Configure IDE integration](#ide-integration)
- [Start local services](#local-services)
- [Begin feature development](#development-workflows)
