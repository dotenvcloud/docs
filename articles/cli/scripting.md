---
title: CLI Scripting Guide
slug: scripting
order: 6
tags: [cli, scripting, automation]
---

# CLI Scripting Guide

This guide demonstrates how to use the DotEnv CLI in scripts and automation workflows.

## Shell Scripting Basics

### Exit Codes

The CLI follows standard Unix exit codes:

```bash
# Success
dotenv secrets list
echo $?  # 0

# Error
dotenv get NONEXISTENT_KEY
echo $?  # 1

# Check in scripts
if dotenv validate; then
  echo "Validation passed"
else
  echo "Validation failed with code $?"
  exit 1
fi
```

### Error Handling

Robust error handling patterns:

```bash
#!/bin/bash
set -euo pipefail  # Exit on error, undefined vars, pipe failures

# Trap errors
trap 'echo "Error on line $LINENO"' ERR

# Try-catch pattern
if ! dotenv pull; then
  echo "Failed to pull secrets, using cached values"
  if [ -f .env.cache ]; then
    cp .env.cache .env
  else
    echo "No cache available, exiting"
    exit 1
  fi
fi

# Ensure cleanup
trap 'rm -f .env.tmp' EXIT
dotenv pull > .env.tmp
mv .env.tmp .env
```

### Output Parsing

Parse CLI output safely:

```bash
# JSON output (recommended)
PROJECT_ID=$(dotenv projects list --format json | jq -r '.[0].id')

# TSV for simple cases
dotenv secrets list --format tsv | while IFS=$'\t' read -r key value updated; do
  echo "Processing $key..."
done

# Capture specific fields
API_KEY=$(dotenv get API_KEY --format raw)

# Handle multiline values
DATABASE_URL=$(dotenv get DATABASE_URL --format raw | head -n1)
```

## Common Automation Patterns

### Environment Setup

Automated environment initialization:

```bash
#!/bin/bash
# setup-env.sh

echo "Setting up DotEnv environment..."

# Check authentication
if ! dotenv whoami >/dev/null 2>&1; then
  echo "Not authenticated. Please run: dotenv login"
  exit 1
fi

# Select or create project
PROJECT=${1:-$(basename "$PWD")}
if ! dotenv projects list --format json | jq -e ".[] | select(.name==\"$PROJECT\")" >/dev/null; then
  echo "Creating project: $PROJECT"
  dotenv projects create "$PROJECT"
fi

# Set as active project
dotenv use "$PROJECT"

# Initialize environments
for env in development staging production; do
  if ! dotenv env list --format json | jq -e ".[] | select(.name==\"$env\")" >/dev/null; then
    echo "Creating environment: $env"
    dotenv env create "$env"
  fi
done

# Set default environment
dotenv env use development

echo "Setup complete! Project: $PROJECT"
```

### Secret Rotation

Automated secret rotation:

```bash
#!/bin/bash
# rotate-secrets.sh

# Configuration
ROTATION_AGE="${ROTATION_AGE:-90d}"
PROJECT="${DOTENV_PROJECT:-$(dotenv config get default_project)}"
NOTIFY_WEBHOOK="${SLACK_WEBHOOK_URL}"

# Function to notify
notify() {
  local message=$1
  if [ -n "$NOTIFY_WEBHOOK" ]; then
    curl -X POST "$NOTIFY_WEBHOOK" \
      -H 'Content-Type: application/json' \
      -d "{\"text\": \"$message\"}"
  fi
}

# Get secrets older than threshold
OLD_SECRETS=$(dotenv secrets list \
  --project "$PROJECT" \
  --format json | \
  jq -r --arg age "$ROTATION_AGE" \
  '.[] | select(.updated < (now - ($age | sub("d$"; "") | tonumber * 86400))) | .key')

if [ -z "$OLD_SECRETS" ]; then
  echo "No secrets need rotation"
  exit 0
fi

# Rotate each secret
echo "$OLD_SECRETS" | while read -r key; do
  echo "Rotating: $key"

  if dotenv rotate "$key" --generate --project "$PROJECT"; then
    notify "✅ Rotated secret: $key in project $PROJECT"
  else
    notify "❌ Failed to rotate: $key in project $PROJECT"
    exit 1
  fi
done

notify "🔄 Secret rotation complete for $PROJECT"
```

### Deployment Integration

Deploy with secrets:

```bash
#!/bin/bash
# deploy.sh

# Determine environment from branch
case "${GITHUB_REF:-$(git branch --show-current)}" in
  main|master)
    ENVIRONMENT="production"
    ;;
  staging)
    ENVIRONMENT="staging"
    ;;
  *)
    ENVIRONMENT="development"
    ;;
esac

echo "Deploying to $ENVIRONMENT..."

# Pull secrets
if ! dotenv pull --environment "$ENVIRONMENT" --output .env; then
  echo "Failed to pull secrets"
  exit 1
fi

# Validate required secrets
REQUIRED_SECRETS="DATABASE_URL API_KEY JWT_SECRET"
for secret in $REQUIRED_SECRETS; do
  if ! grep -q "^${secret}=" .env; then
    echo "Missing required secret: $secret"
    exit 1
  fi
done

# Build application
source .env
npm run build

# Deploy based on environment
case "$ENVIRONMENT" in
  production)
    npm run deploy:prod
    ;;
  staging)
    npm run deploy:staging
    ;;
  *)
    npm run deploy:dev
    ;;
esac

# Cleanup
rm -f .env
```

## CI/CD Scripts

### GitHub Actions

```bash
#!/bin/bash
# .github/scripts/dotenv-sync.sh

set -euo pipefail

# Install CLI if not present
if ! command -v dotenv &> /dev/null; then
  curl -sSL https://dotenv.cloud/install.sh | sh
  export PATH="$HOME/.dotenv/bin:$PATH"
fi

# Authenticate
export DOTENV_API_KEY="${DOTENV_API_KEY}"

# Determine environment
if [[ "$GITHUB_REF" == "refs/heads/main" ]]; then
  ENV="production"
elif [[ "$GITHUB_REF" == "refs/heads/staging" ]]; then
  ENV="staging"
else
  ENV="development"
fi

# Pull and export secrets
dotenv pull \
  --project "${DOTENV_PROJECT}" \
  --environment "$ENV" \
  --format shell > secrets.sh

# Make secrets available to next steps
{
  echo "source secrets.sh"
  cat secrets.sh
} >> "$GITHUB_ENV"
```

### Jenkins Pipeline

```groovy
// Jenkinsfile
def pullDotEnvSecrets(environment) {
  sh '''
    #!/bin/bash
    set -euo pipefail

    # Pull secrets
    dotenv pull \
      --project "$DOTENV_PROJECT" \
      --environment "''' + environment + '''" \
      --format jenkins > secrets.properties

    # Validate
    if [ ! -s secrets.properties ]; then
      echo "No secrets retrieved"
      exit 1
    fi
  '''

  // Load as Jenkins properties
  def props = readProperties file: 'secrets.properties'
  props.each { key, value ->
    env[key] = value
  }
}

pipeline {
  agent any

  environment {
    DOTENV_API_KEY = credentials('dotenv-api-key')
    DOTENV_PROJECT = 'my-app'
  }

  stages {
    stage('Setup') {
      steps {
        script {
          pullDotEnvSecrets(env.BRANCH_NAME == 'main' ? 'production' : 'staging')
        }
      }
    }

    stage('Build') {
      steps {
        sh 'npm run build'
      }
    }
  }
}
```

### GitLab CI

```yaml
# .gitlab-ci.yml
.dotenv_setup:
    before_script:
        - |
            # Install DotEnv CLI
            if ! command -v dotenv &> /dev/null; then
              curl -sSL https://dotenv.cloud/install.sh | sh
              export PATH="$HOME/.dotenv/bin:$PATH"
            fi

            # Pull secrets based on branch
            case "$CI_COMMIT_REF_NAME" in
              main) ENV="production" ;;
              staging) ENV="staging" ;;
              *) ENV="development" ;;
            esac

            # Export secrets
            dotenv pull \
              --project "$DOTENV_PROJECT" \
              --environment "$ENV" \
              --format shell > .env.sh

            # Source secrets
            set -a
            source .env.sh
            set +a

deploy:
    extends: .dotenv_setup
    script:
        - echo "Deploying with secrets..."
        - ./deploy.sh
```

## Advanced Scripting Patterns

### Parallel Processing

Process multiple projects in parallel:

```bash
#!/bin/bash
# parallel-sync.sh

# Function to process a single project
process_project() {
  local project=$1
  echo "[$project] Starting sync..."

  if dotenv pull --project "$project" --output ".env.$project"; then
    echo "[$project] Success"
    return 0
  else
    echo "[$project] Failed"
    return 1
  fi
}

export -f process_project

# Get all projects
PROJECTS=$(dotenv projects list --format json | jq -r '.[].name')

# Process in parallel (max 5 concurrent)
echo "$PROJECTS" | xargs -P 5 -I {} bash -c 'process_project "{}"'

# Combine results
cat .env.* > .env.combined
rm -f .env.*
```

### State Management

Maintain state between runs:

```bash
#!/bin/bash
# stateful-sync.sh

STATE_FILE=".dotenv-sync.state"

# Load previous state
if [ -f "$STATE_FILE" ]; then
  source "$STATE_FILE"
fi

# Get current timestamp
CURRENT_TIME=$(date +%s)
LAST_SYNC=${LAST_SYNC:-0}

# Check if sync needed (every hour)
if (( CURRENT_TIME - LAST_SYNC < 3600 )); then
  echo "Last sync was $(( (CURRENT_TIME - LAST_SYNC) / 60 )) minutes ago, skipping"
  exit 0
fi

# Perform sync
if dotenv pull; then
  # Update state
  cat > "$STATE_FILE" <<EOF
LAST_SYNC=$CURRENT_TIME
LAST_STATUS=success
SYNC_COUNT=$((SYNC_COUNT + 1))
EOF
  echo "Sync complete (count: $((SYNC_COUNT + 1)))"
else
  # Update failure state
  cat > "$STATE_FILE" <<EOF
LAST_SYNC=$LAST_SYNC
LAST_STATUS=failed
LAST_FAILURE=$CURRENT_TIME
FAILURE_COUNT=$((FAILURE_COUNT + 1))
EOF
  echo "Sync failed (failures: $((FAILURE_COUNT + 1)))"
  exit 1
fi
```

### Dynamic Configuration

Generate configuration from secrets:

```bash
#!/bin/bash
# generate-config.sh

# Template function
generate_nginx_config() {
  cat <<EOF
server {
    listen ${PORT:-80};
    server_name ${SERVER_NAME:-localhost};

    location / {
        proxy_pass ${BACKEND_URL:-http://localhost:3000};
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
    }

    ssl_certificate ${SSL_CERT_PATH:-/etc/ssl/cert.pem};
    ssl_certificate_key ${SSL_KEY_PATH:-/etc/ssl/key.pem};
}
EOF
}

# Pull secrets
dotenv pull --environment production

# Source secrets
set -a
source .env
set +a

# Generate configs
generate_nginx_config > /etc/nginx/sites-available/app

# Validate
nginx -t && systemctl reload nginx
```

## Testing Scripts

### Unit Testing

Test scripts with mock data:

```bash
#!/bin/bash
# test-deploy.sh

# Mock dotenv command
mock_dotenv() {
  case "$1" in
    "pull")
      cat > .env <<EOF
DATABASE_URL=mock://localhost/test
API_KEY=mock_key_123
EOF
      return 0
      ;;
    "validate")
      return 0
      ;;
    *)
      return 1
      ;;
  esac
}

# Override for testing
if [ "${MOCK_MODE:-}" = "true" ]; then
  alias dotenv=mock_dotenv
fi

# Run deployment script
source ./deploy.sh

# Assertions
assert_file_exists() {
  if [ ! -f "$1" ]; then
    echo "FAIL: File $1 does not exist"
    exit 1
  fi
}

assert_env_var() {
  if [ -z "${!1}" ]; then
    echo "FAIL: Environment variable $1 is not set"
    exit 1
  fi
}

# Test
assert_file_exists ".env"
assert_env_var "DATABASE_URL"
assert_env_var "API_KEY"

echo "All tests passed!"
```

### Integration Testing

Test real integrations:

```bash
#!/bin/bash
# integration-test.sh

# Test environment
TEST_PROJECT="test-$(date +%s)"
TEST_KEY="TEST_KEY_$(date +%s)"
TEST_VALUE="test_value_123"

# Cleanup function
cleanup() {
  dotenv projects delete "$TEST_PROJECT" --force 2>/dev/null || true
}
trap cleanup EXIT

# Test project creation
echo "Testing project creation..."
if ! dotenv projects create "$TEST_PROJECT"; then
  echo "FAIL: Could not create project"
  exit 1
fi

# Test secret operations
echo "Testing secret operations..."
dotenv use "$TEST_PROJECT"

# Set secret
if ! dotenv set "$TEST_KEY=$TEST_VALUE"; then
  echo "FAIL: Could not set secret"
  exit 1
fi

# Get secret
RETRIEVED=$(dotenv get "$TEST_KEY" --format raw)
if [ "$RETRIEVED" != "$TEST_VALUE" ]; then
  echo "FAIL: Retrieved value mismatch"
  exit 1
fi

# Pull test
dotenv pull --output .env.test
if ! grep -q "$TEST_KEY=$TEST_VALUE" .env.test; then
  echo "FAIL: Pull did not contain secret"
  exit 1
fi

echo "All integration tests passed!"
```

## Performance Scripts

### Benchmarking

Measure CLI performance:

```bash
#!/bin/bash
# benchmark.sh

# Function to measure time
benchmark() {
  local name=$1
  shift
  local start=$(date +%s.%N)
  "$@"
  local end=$(date +%s.%N)
  echo "$name: $(echo "$end - $start" | bc) seconds"
}

# Run benchmarks
echo "DotEnv CLI Benchmarks"
echo "===================="

benchmark "Login" dotenv whoami
benchmark "List Projects" dotenv projects list
benchmark "List Secrets" dotenv secrets list
benchmark "Get Single Secret" dotenv get API_KEY
benchmark "Pull All Secrets" dotenv pull --output /dev/null
benchmark "Validate" dotenv validate

# Bulk operations
echo -e "\nBulk Operations"
echo "==============="

# Create test data
for i in {1..100}; do
  echo "BULK_TEST_$i=value_$i"
done > bulk.env

benchmark "Set 100 Secrets" dotenv import bulk.env
benchmark "Export 100 Secrets" dotenv export --output /dev/null

rm -f bulk.env
```

### Optimization

Optimize script performance:

```bash
#!/bin/bash
# optimized-sync.sh

# Cache configuration
CACHE_DIR="${HOME}/.dotenv/script-cache"
CACHE_TTL=300  # 5 minutes
mkdir -p "$CACHE_DIR"

# Check cache
get_cached() {
  local key=$1
  local cache_file="$CACHE_DIR/$key"

  if [ -f "$cache_file" ]; then
    local age=$(( $(date +%s) - $(stat -c %Y "$cache_file" 2>/dev/null || stat -f %m "$cache_file") ))
    if [ $age -lt $CACHE_TTL ]; then
      cat "$cache_file"
      return 0
    fi
  fi
  return 1
}

# Set cache
set_cached() {
  local key=$1
  local value=$2
  echo "$value" > "$CACHE_DIR/$key"
}

# Optimized project list
get_projects() {
  local projects
  if projects=$(get_cached "projects"); then
    echo "$projects"
  else
    projects=$(dotenv projects list --format json)
    set_cached "projects" "$projects"
    echo "$projects"
  fi
}

# Use cached data
PROJECTS=$(get_projects | jq -r '.[].name')

# Process with minimal API calls
for project in $PROJECTS; do
  echo "Processing $project..."
  # Implementation
done
```

## Utility Scripts

### Secret Scanner

Scan for exposed secrets:

```bash
#!/bin/bash
# scan-secrets.sh

# Patterns to search for
PATTERNS=(
  'api[_-]?key'
  'secret[_-]?key'
  'password'
  'token'
  'private[_-]?key'
)

# Files to exclude
EXCLUDE_PATTERNS=(
  '*.env.example'
  '*.md'
  'package-lock.json'
)

# Build grep pattern
GREP_PATTERN=$(IFS='|'; echo "${PATTERNS[*]}")

# Build find excludes
FIND_EXCLUDES=""
for pattern in "${EXCLUDE_PATTERNS[@]}"; do
  FIND_EXCLUDES="$FIND_EXCLUDES -not -name '$pattern'"
done

# Scan files
echo "Scanning for potential secrets..."
eval "find . -type f $FIND_EXCLUDES" | while read -r file; do
  if grep -Ei "$GREP_PATTERN" "$file" | grep -Ev '^[[:space:]]*[#*//]' | grep -E '=.+|:.+'; then
    echo "WARNING: Potential secret in $file"
  fi
done

# Check git history
echo -e "\nChecking git history..."
git log -p | grep -Ei "$GREP_PATTERN" | head -20
```

### Environment Differ

Compare environments with context:

```bash
#!/bin/bash
# env-diff.sh

ENV1=${1:-development}
ENV2=${2:-production}

# Get secrets from both environments
echo "Comparing $ENV1 vs $ENV2..."

dotenv export --environment "$ENV1" --format json > env1.json
dotenv export --environment "$ENV2" --format json > env2.json

# Compare with jq
jq -r --slurpfile env2 env2.json '
  . as $env1 |
  $env2[0] as $env2 |
  {
    "only_in_env1": ($env1 | keys - ($env2 | keys)),
    "only_in_env2": (($env2 | keys) - ($env1 | keys)),
    "different_values": [
      $env1 | to_entries[] |
      select($env2[.key] != null and $env2[.key] != .value) |
      {
        key: .key,
        env1: .value,
        env2: $env2[.key]
      }
    ]
  }
' env1.json

# Cleanup
rm -f env1.json env2.json
```

## Next Steps

- [Troubleshooting](./troubleshooting) - Common issues and solutions
- [CI/CD Integration](./ci-cd-usage) - Platform-specific guides
- [Advanced Usage](./advanced-usage) - Power user features
- [Command Reference](./commands) - Complete command list
