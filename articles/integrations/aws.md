---
title: AWS Integration
slug: aws
order: 10
tags: [integrations, aws, cloud, deployment]
---

# AWS Integration

Integrate DotEnv with Amazon Web Services to manage secrets across your AWS infrastructure. This guide covers integration with various AWS services and deployment patterns.

## Overview

The DotEnv AWS integration enables:

- AWS Secrets Manager synchronization
- ECS/Fargate container secrets
- Lambda function configuration
- EC2 instance secret management
- CodeBuild/CodeDeploy integration
- Multi-region secret replication

## Installation

### 1. AWS Lambda Layer

Install as a Lambda Layer:

```bash
# Create layer package
mkdir dotenv-layer
cd dotenv-layer
curl -fsSL https://cli.dotenv.cloud/install.sh | DOTENV_INSTALL_DIR=. sh
zip -r dotenv-layer.zip .

# Publish layer
aws lambda publish-layer-version \
  --layer-name dotenv-cli \
  --description "DotEnv CLI for Lambda" \
  --zip-file fileb://dotenv-layer.zip \
  --compatible-runtimes nodejs18.x python3.10 python3.11
```

### 2. Container Image

Add to Dockerfile:

```dockerfile
FROM public.ecr.aws/lambda/nodejs:18

# Install DotEnv CLI
RUN curl -fsSL https://cli.dotenv.cloud/install.sh | sh
ENV PATH="/root/.dotenv/bin:$PATH"

# Copy function code
COPY . ${LAMBDA_TASK_ROOT}

# Load secrets at runtime
CMD ["index.handler"]
```

### 3. EC2 User Data

Install via User Data:

```bash
#!/bin/bash
# Install DotEnv CLI
curl -fsSL https://cli.dotenv.cloud/install.sh | sh
echo 'export PATH="$HOME/.dotenv/bin:$PATH"' >> /etc/profile

# Configure systemd service
cat > /etc/systemd/system/dotenv-sync.service <<EOF
[Unit]
Description=DotEnv Secret Sync
After=network.target

[Service]
Type=oneshot
ExecStart=/root/.dotenv/bin/dotenv sync-aws
Environment="DOTENV_API_KEY=${DOTENV_API_KEY}"
Environment="DOTENV_PROJECT=${DOTENV_PROJECT}"

[Install]
WantedBy=multi-user.target
EOF

systemctl enable dotenv-sync
systemctl start dotenv-sync
```

## AWS Secrets Manager Integration

### Sync to Secrets Manager

Create sync script:

```python
# scripts/sync-to-aws.py
import boto3
import subprocess
import json
import os

def sync_dotenv_to_aws():
    """Sync DotEnv secrets to AWS Secrets Manager."""

    # Initialize AWS client
    secrets_client = boto3.client('secretsmanager')

    # Get secrets from DotEnv
    project = os.environ.get('DOTENV_PROJECT', 'my-app')
    environment = os.environ.get('DOTENV_ENVIRONMENT', 'production')

    result = subprocess.run(
        ['dotenv', 'pull', environment, '--project', project, '--format', 'json'],
        capture_output=True,
        text=True
    )

    if result.returncode != 0:
        raise Exception(f"Failed to pull secrets: {result.stderr}")

    secrets = json.loads(result.stdout)

    # Create or update AWS secret
    secret_name = f"{project}/{environment}"

    try:
        # Try to update existing secret
        secrets_client.update_secret(
            SecretId=secret_name,
            SecretString=json.dumps(secrets)
        )
        print(f"Updated secret: {secret_name}")
    except secrets_client.exceptions.ResourceNotFoundException:
        # Create new secret
        secrets_client.create_secret(
            Name=secret_name,
            SecretString=json.dumps(secrets),
            Tags=[
                {'Key': 'ManagedBy', 'Value': 'DotEnv'},
                {'Key': 'Project', 'Value': project},
                {'Key': 'Environment', 'Value': environment}
            ]
        )
        print(f"Created secret: {secret_name}")

    # Enable rotation if needed
    if os.environ.get('ENABLE_ROTATION') == 'true':
        enable_rotation(secrets_client, secret_name)

def enable_rotation(client, secret_name):
    """Enable automatic rotation for the secret."""
    rotation_lambda_arn = os.environ.get('ROTATION_LAMBDA_ARN')

    client.rotate_secret(
        SecretId=secret_name,
        RotationLambdaARN=rotation_lambda_arn,
        RotationRules={
            'AutomaticallyAfterDays': 30
        }
    )

if __name__ == '__main__':
    sync_dotenv_to_aws()
```

### Load from Secrets Manager

```javascript
// utils/secrets.js
const AWS = require("aws-sdk");
const secretsManager = new AWS.SecretsManager();

async function loadSecretsFromAWS() {
    const secretName = `${process.env.APP_NAME}/${process.env.ENVIRONMENT}`;

    try {
        const data = await secretsManager
            .getSecretValue({ SecretId: secretName })
            .promise();
        const secrets = JSON.parse(data.SecretString);

        // Set environment variables
        Object.entries(secrets).forEach(([key, value]) => {
            process.env[key] = value;
        });

        console.log(`Loaded ${Object.keys(secrets).length} secrets from AWS`);
    } catch (error) {
        console.error("Failed to load secrets from AWS:", error);
        throw error;
    }
}

module.exports = { loadSecretsFromAWS };
```

## Lambda Functions

### Environment Variables

Configure Lambda with DotEnv:

```javascript
// index.js
const { execSync } = require("child_process");

// Load secrets at cold start
let secretsLoaded = false;

async function loadSecrets() {
    if (secretsLoaded) return;

    try {
        // DotEnv CLI is available via Lambda Layer
        execSync(
            `dotenv pull ${process.env.ENVIRONMENT} --project ${process.env.PROJECT_NAME}`,
            { stdio: "inherit" },
        );

        // Load .env file
        require("dotenv").config();
        secretsLoaded = true;
    } catch (error) {
        console.error("Failed to load secrets:", error);
        // Fall back to Lambda environment variables
    }
}

exports.handler = async (event, context) => {
    await loadSecrets();

    // Your function logic
    return {
        statusCode: 200,
        body: JSON.stringify({
            message: "Success",
            hasDatabase: !!process.env.DATABASE_URL,
        }),
    };
};
```

### Terraform Configuration

Deploy with Terraform:

```hcl
# lambda.tf
resource "aws_lambda_function" "app" {
  function_name = "my-app"
  runtime       = "nodejs18.x"
  handler       = "index.handler"

  environment {
    variables = {
      DOTENV_API_KEY    = var.dotenv_api_key
      DOTENV_PROJECT    = var.project_name
      DOTENV_ENVIRONMENT = var.environment
    }
  }

  layers = [aws_lambda_layer_version.dotenv.arn]

  # IAM role with Secrets Manager permissions
  role = aws_iam_role.lambda.arn
}

resource "aws_lambda_layer_version" "dotenv" {
  filename   = "dotenv-layer.zip"
  layer_name = "dotenv-cli"

  compatible_runtimes = ["nodejs18.x", "python3.10", "python3.11"]
}

resource "aws_iam_role_policy" "lambda_secrets" {
  name = "lambda-secrets-policy"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ]
        Resource = "arn:aws:secretsmanager:*:*:secret:${var.project_name}/*"
      }
    ]
  })
}
```

## ECS/Fargate

### Task Definition

Configure ECS tasks:

```json
{
    "family": "my-app",
    "taskRoleArn": "arn:aws:iam::123456789012:role/ecsTaskRole",
    "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
    "networkMode": "awsvpc",
    "cpu": "256",
    "memory": "512",
    "containerDefinitions": [
        {
            "name": "app",
            "image": "my-app:latest",
            "essential": true,
            "environment": [
                {
                    "name": "DOTENV_PROJECT",
                    "value": "my-app"
                },
                {
                    "name": "DOTENV_ENVIRONMENT",
                    "value": "production"
                }
            ],
            "secrets": [
                {
                    "name": "DOTENV_API_KEY",
                    "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:dotenv/api-key"
                }
            ],
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/my-app",
                    "awslogs-region": "us-east-1",
                    "awslogs-stream-prefix": "ecs"
                }
            }
        }
    ]
}
```

### Container Entrypoint

```bash
#!/bin/sh
# entrypoint.sh
set -e

# Load secrets from DotEnv
if [ -n "$DOTENV_API_KEY" ]; then
  echo "Loading secrets from DotEnv..."
  dotenv auth login --api-key $DOTENV_API_KEY
  dotenv pull $DOTENV_ENVIRONMENT --project $DOTENV_PROJECT

  # Export to environment
  set -a
  source .env
  set +a
fi

# Start application
exec "$@"
```

## CodeBuild/CodeDeploy

### CodeBuild Integration

`buildspec.yml`:

```yaml
version: 0.2

env:
    parameter-store:
        DOTENV_API_KEY: /dotenv/api-key

phases:
    install:
        runtime-versions:
            nodejs: 18
        commands:
            - echo "Installing DotEnv CLI..."
            - curl -fsSL https://cli.dotenv.cloud/install.sh | sh
            - export PATH="$HOME/.dotenv/bin:$PATH"

    pre_build:
        commands:
            - echo "Loading secrets..."
            - dotenv auth login --api-key $DOTENV_API_KEY
            - dotenv pull $ENVIRONMENT --project $PROJECT_NAME
            - source .env

    build:
        commands:
            - echo "Building application..."
            - npm install
            - npm run build

    post_build:
        commands:
            - echo "Build completed"
            - aws s3 cp dist/ s3://$BUCKET_NAME/ --recursive

artifacts:
    files:
        - "**/*"
    name: build-output

cache:
    paths:
        - node_modules/**/*
        - ~/.dotenv/**/*
```

### CodeDeploy Hooks

`appspec.yml`:

```yaml
version: 0.0
os: linux

files:
    - source: /
      destination: /var/app

hooks:
    BeforeInstall:
        - location: scripts/install-dotenv.sh
          timeout: 300
          runas: root

    ApplicationStart:
        - location: scripts/load-secrets.sh
          timeout: 300
          runas: app

    ValidateService:
        - location: scripts/validate.sh
          timeout: 300
```

`scripts/load-secrets.sh`:

```bash
#!/bin/bash
cd /var/app

# Get API key from Parameter Store
DOTENV_API_KEY=$(aws ssm get-parameter \
  --name /dotenv/api-key \
  --with-decryption \
  --query 'Parameter.Value' \
  --output text)

# Load secrets
export PATH="$HOME/.dotenv/bin:$PATH"
dotenv auth login --api-key $DOTENV_API_KEY
dotenv pull production --project my-app

# Restart service with new secrets
sudo systemctl restart my-app
```

## EC2 Instances

### Instance Profile Setup

Configure IAM role:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:GetParameter",
                "ssm:GetParameters",
                "ssm:GetParametersByPath"
            ],
            "Resource": "arn:aws:ssm:*:*:parameter/dotenv/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": "arn:aws:secretsmanager:*:*:secret:dotenv/*"
        }
    ]
}
```

### Auto Scaling Configuration

User data script:

```bash
#!/bin/bash
# Load instance metadata
INSTANCE_ID=$(ec2-metadata --instance-id | cut -d " " -f 2)
REGION=$(ec2-metadata --availability-zone | cut -d " " -f 2 | sed 's/[a-z]$//')

# Install DotEnv
curl -fsSL https://cli.dotenv.cloud/install.sh | sh
export PATH="/root/.dotenv/bin:$PATH"

# Get API key from Parameter Store
DOTENV_API_KEY=$(aws ssm get-parameter \
  --name /dotenv/api-key \
  --region $REGION \
  --with-decryption \
  --query 'Parameter.Value' \
  --output text)

# Determine environment based on tags
ENVIRONMENT=$(aws ec2 describe-tags \
  --region $REGION \
  --filters "Name=resource-id,Values=$INSTANCE_ID" "Name=key,Values=Environment" \
  --query 'Tags[0].Value' \
  --output text)

# Load secrets
dotenv auth login --api-key $DOTENV_API_KEY
dotenv pull $ENVIRONMENT --project my-app --output /etc/environment

# Start application
systemctl start my-app
```

## Security Best Practices

### 1. IAM Policies

Least privilege access:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DotEnvSecretAccess",
            "Effect": "Allow",
            "Action": ["secretsmanager:GetSecretValue"],
            "Resource": "arn:aws:secretsmanager:*:*:secret:dotenv/${Environment}/*",
            "Condition": {
                "StringEquals": {
                    "secretsmanager:ResourceTag/Project": "${aws:PrincipalTag/Project}"
                }
            }
        }
    ]
}
```

### 2. KMS Encryption

Encrypt secrets at rest:

```python
# utils/encryption.py
import boto3
from base64 import b64encode, b64decode

kms = boto3.client('kms')
KEY_ID = 'arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012'

def encrypt_secret(plaintext):
    """Encrypt secret using KMS."""
    response = kms.encrypt(
        KeyId=KEY_ID,
        Plaintext=plaintext,
        EncryptionContext={
            'purpose': 'dotenv-secrets',
            'environment': os.environ.get('ENVIRONMENT', 'production')
        }
    )
    return b64encode(response['CiphertextBlob']).decode('utf-8')

def decrypt_secret(ciphertext):
    """Decrypt secret using KMS."""
    response = kms.decrypt(
        CiphertextBlob=b64decode(ciphertext),
        EncryptionContext={
            'purpose': 'dotenv-secrets',
            'environment': os.environ.get('ENVIRONMENT', 'production')
        }
    )
    return response['Plaintext'].decode('utf-8')
```

### 3. VPC Endpoints

Keep traffic private:

```hcl
# vpc-endpoints.tf
resource "aws_vpc_endpoint" "secrets_manager" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.secretsmanager"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]

  private_dns_enabled = true
}

resource "aws_vpc_endpoint" "ssm" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.ssm"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]

  private_dns_enabled = true
}
```

### 4. Audit Logging

Enable CloudTrail:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "LogSecretAccess",
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:*:*:log-group:/aws/lambda/dotenv-*"
        }
    ]
}
```

## Multi-Region Deployment

### Cross-Region Replication

```python
# scripts/replicate-secrets.py
import boto3
import json

def replicate_secrets(source_region, target_regions):
    """Replicate secrets across regions."""

    source_client = boto3.client('secretsmanager', region_name=source_region)

    # List all DotEnv secrets
    paginator = source_client.get_paginator('list_secrets')
    secrets = []

    for page in paginator.paginate(
        Filters=[
            {
                'Key': 'tag-key',
                'Values': ['ManagedBy']
            },
            {
                'Key': 'tag-value',
                'Values': ['DotEnv']
            }
        ]
    ):
        secrets.extend(page['SecretList'])

    # Replicate to target regions
    for secret in secrets:
        secret_value = source_client.get_secret_value(
            SecretId=secret['ARN']
        )

        for region in target_regions:
            target_client = boto3.client('secretsmanager', region_name=region)

            try:
                # Create replica
                target_client.create_secret(
                    Name=secret['Name'],
                    SecretString=secret_value['SecretString'],
                    Tags=secret.get('Tags', [])
                )
                print(f"Replicated {secret['Name']} to {region}")
            except target_client.exceptions.ResourceExistsException:
                # Update existing
                target_client.update_secret(
                    SecretId=secret['Name'],
                    SecretString=secret_value['SecretString']
                )
                print(f"Updated {secret['Name']} in {region}")

if __name__ == '__main__':
    replicate_secrets('us-east-1', ['us-west-2', 'eu-west-1', 'ap-southeast-1'])
```

## Monitoring

### CloudWatch Metrics

Track secret usage:

```javascript
// utils/metrics.js
const AWS = require("aws-sdk");
const cloudwatch = new AWS.CloudWatch();

async function trackSecretAccess(secretName, success) {
    const params = {
        Namespace: "DotEnv",
        MetricData: [
            {
                MetricName: "SecretAccess",
                Dimensions: [
                    {
                        Name: "SecretName",
                        Value: secretName,
                    },
                    {
                        Name: "Result",
                        Value: success ? "Success" : "Failure",
                    },
                ],
                Value: 1,
                Unit: "Count",
                Timestamp: new Date(),
            },
        ],
    };

    await cloudwatch.putMetricData(params).promise();
}

// Create alarm
async function createSecretAccessAlarm(secretName) {
    const params = {
        AlarmName: `DotEnv-${secretName}-AccessFailure`,
        ComparisonOperator: "GreaterThanThreshold",
        EvaluationPeriods: 1,
        MetricName: "SecretAccess",
        Namespace: "DotEnv",
        Period: 300,
        Statistic: "Sum",
        Threshold: 5,
        ActionsEnabled: true,
        AlarmActions: [process.env.SNS_TOPIC_ARN],
        AlarmDescription: "Alert on secret access failures",
        Dimensions: [
            {
                Name: "SecretName",
                Value: secretName,
            },
            {
                Name: "Result",
                Value: "Failure",
            },
        ],
    };

    await cloudwatch.putMetricAlarm(params).promise();
}

module.exports = { trackSecretAccess, createSecretAccessAlarm };
```

## Troubleshooting

### Common Issues

#### Lambda Timeout

```javascript
// Optimize cold start
const secretsPromise = loadSecrets(); // Start loading immediately

exports.handler = async (event) => {
    await secretsPromise; // Wait only when needed

    // Handler logic
};
```

#### IAM Permission Errors

Debug script:

```bash
#!/bin/bash
# Check IAM permissions
aws sts get-caller-identity
aws iam simulate-principal-policy \
  --policy-source-arn $(aws sts get-caller-identity --query Arn --output text) \
  --action-names secretsmanager:GetSecretValue \
  --resource-arns "arn:aws:secretsmanager:*:*:secret:dotenv/*"
```

## Examples

### Complete Serverless Application

`serverless.yml`:

```yaml
service: my-serverless-app

provider:
    name: aws
    runtime: nodejs18.x
    environment:
        DOTENV_PROJECT: ${self:service}
        DOTENV_ENVIRONMENT: ${opt:stage, 'dev'}

    iamRoleStatements:
        - Effect: Allow
          Action:
              - secretsmanager:GetSecretValue
          Resource:
              - "arn:aws:secretsmanager:*:*:secret:dotenv/*"

layers:
    dotenv:
        path: layers/dotenv
        name: ${self:service}-dotenv
        description: DotEnv CLI Layer

functions:
    api:
        handler: src/index.handler
        layers:
            - { Ref: DotenvLambdaLayer }
        events:
            - http:
                  path: /
                  method: any
            - http:
                  path: /{proxy+}
                  method: any
        environment:
            DOTENV_API_KEY: ${ssm:/dotenv/api-key~true}

plugins:
    - serverless-webpack
    - serverless-offline

custom:
    webpack:
        webpackConfig: ./webpack.config.js
        includeModules: true
```

## Resources

- [AWS SDK Documentation](https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/)
- [Secrets Manager](https://docs.aws.amazon.com/secretsmanager/)
- [Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [DotEnv CLI Reference](/documentation/v1/cli/commands)

## Next Steps

- [Configure Secrets Manager sync](#aws-secrets-manager-integration)
- [Set up Lambda functions](#lambda-functions)
- [Deploy to ECS/Fargate](#ecsfargate)
- [Enable monitoring](#monitoring)
