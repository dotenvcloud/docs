---
title: Azure Integration
slug: azure
order: 12
tags: [integrations, azure, cloud, microsoft]
---

# Azure Integration

Integrate DotEnv with Microsoft Azure to manage secrets across your Azure infrastructure. This guide covers integration with various Azure services and deployment patterns.

## Overview

The DotEnv Azure integration provides:

- Azure Key Vault synchronization
- App Service configuration
- Azure Functions integration
- AKS secret management
- Container Instances support
- Azure DevOps pipeline integration

## Installation

### 1. Azure Functions

Configure in `function.json`:

```json
{
    "bindings": [
        {
            "authLevel": "anonymous",
            "type": "httpTrigger",
            "direction": "in",
            "name": "req",
            "methods": ["get", "post"]
        },
        {
            "type": "http",
            "direction": "out",
            "name": "res"
        }
    ],
    "scriptFile": "../dist/HttpTrigger/index.js"
}
```

Install DotEnv:

```javascript
// HttpTrigger/index.js
const { execSync } = require("child_process");

// Install and load DotEnv at cold start
let secretsLoaded = false;

async function loadSecrets() {
    if (secretsLoaded) return;

    try {
        // Install CLI if not present
        if (!require("fs").existsSync("/home/.dotenv/bin/dotenv")) {
            execSync("curl -fsSL https://cli.dotenv.cloud/install.sh | sh");
        }

        // Load secrets
        process.env.PATH = `/home/.dotenv/bin:${process.env.PATH}`;
        execSync(
            `dotenv pull ${process.env.DOTENV_ENVIRONMENT} --project ${process.env.DOTENV_PROJECT}`,
            { stdio: "inherit" },
        );

        require("dotenv").config();
        secretsLoaded = true;
    } catch (error) {
        console.error("Failed to load secrets:", error);
    }
}

module.exports = async function (context, req) {
    await loadSecrets();

    context.res = {
        body: {
            message: "Function running",
            hasDatabase: !!process.env.DATABASE_URL,
        },
    };
};
```

### 2. App Service

Configure app settings:

```bash
# Set app settings
az webapp config appsettings set \
  --resource-group myResourceGroup \
  --name myapp \
  --settings \
    DOTENV_API_KEY="@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/dotenv-api-key/)" \
    DOTENV_PROJECT="my-app" \
    DOTENV_ENVIRONMENT="production"
```

Startup script:

```bash
#!/bin/bash
# startup.sh
curl -fsSL https://cli.dotenv.cloud/install.sh | sh
export PATH="$HOME/.dotenv/bin:$PATH"

# Load secrets
dotenv auth login --api-key $DOTENV_API_KEY
dotenv pull $DOTENV_ENVIRONMENT --project $DOTENV_PROJECT

# Start application
npm start
```

### 3. Container Instances

Deploy with environment variables:

```bash
az container create \
  --resource-group myResourceGroup \
  --name mycontainer \
  --image myapp:latest \
  --cpu 1 \
  --memory 1 \
  --environment-variables \
    DOTENV_PROJECT=my-app \
    DOTENV_ENVIRONMENT=production \
  --secure-environment-variables \
    DOTENV_API_KEY=$(az keyvault secret show \
      --vault-name myvault \
      --name dotenv-api-key \
      --query value -o tsv)
```

## Key Vault Integration

### Sync to Key Vault

```python
# scripts/sync_to_azure.py
from azure.keyvault.secrets import SecretClient
from azure.identity import DefaultAzureCredential
import subprocess
import json
import os

def sync_dotenv_to_keyvault():
    """Sync DotEnv secrets to Azure Key Vault."""

    # Initialize Key Vault client
    key_vault_name = os.environ["KEY_VAULT_NAME"]
    kv_uri = f"https://{key_vault_name}.vault.azure.net"
    credential = DefaultAzureCredential()
    client = SecretClient(vault_url=kv_uri, credential=credential)

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

    # Sync each secret
    for key, value in secrets.items():
        # Convert to Key Vault compatible name
        secret_name = f"dotenv-{project}-{key}".lower().replace('_', '-')

        try:
            # Set secret with metadata
            client.set_secret(
                secret_name,
                value,
                tags={
                    "ManagedBy": "DotEnv",
                    "Project": project,
                    "Environment": environment,
                    "OriginalKey": key
                }
            )
            print(f"Synced secret: {secret_name}")
        except Exception as e:
            print(f"Error syncing {secret_name}: {e}")

if __name__ == '__main__':
    sync_dotenv_to_keyvault()
```

### Load from Key Vault

```javascript
// utils/keyvault.js
const { SecretClient } = require("@azure/keyvault-secrets");
const { DefaultAzureCredential } = require("@azure/identity");

async function loadSecretsFromKeyVault() {
    const keyVaultName = process.env.KEY_VAULT_NAME;
    const url = `https://${keyVaultName}.vault.azure.net`;

    const credential = new DefaultAzureCredential();
    const client = new SecretClient(url, credential);

    // List all DotEnv managed secrets
    const project = process.env.DOTENV_PROJECT;
    const prefix = `dotenv-${project}-`;

    for await (const properties of client.listPropertiesOfSecrets()) {
        if (properties.name.startsWith(prefix)) {
            const secret = await client.getSecret(properties.name);

            // Convert back to original key name
            const originalKey =
                properties.tags?.OriginalKey ||
                properties.name
                    .replace(prefix, "")
                    .replace(/-/g, "_")
                    .toUpperCase();

            process.env[originalKey] = secret.value;
        }
    }

    console.log("Loaded secrets from Key Vault");
}

module.exports = { loadSecretsFromKeyVault };
```

## App Service

### Configuration

```json
// .azure/config
[defaults]
group = myResourceGroup
sku = P1V2
appserviceplan = myAppServicePlan
location = eastus
web = myapp

[deploy]
startup-file = startup.sh
```

### Deployment

```yaml
# azure-pipelines.yml
trigger:
    - main

pool:
    vmImage: "ubuntu-latest"

variables:
    azureSubscription: "MyAzureSubscription"
    webAppName: "myapp"
    resourceGroupName: "myResourceGroup"

steps:
    - task: NodeTool@0
      inputs:
          versionSpec: "18.x"
      displayName: "Install Node.js"

    - script: |
          curl -fsSL https://cli.dotenv.cloud/install.sh | sh
          echo "##vso[task.prependpath]$HOME/.dotenv/bin"
      displayName: "Install DotEnv CLI"

    - task: AzureKeyVault@2
      inputs:
          azureSubscription: "$(azureSubscription)"
          KeyVaultName: "myvault"
          SecretsFilter: "dotenv-api-key"
      displayName: "Get secrets from Key Vault"

    - script: |
          dotenv auth login --api-key $(dotenv-api-key)
          dotenv pull production --project $(Build.Repository.Name)
          source .env
          npm install
          npm run build
      displayName: "Build application"

    - task: AzureWebApp@1
      inputs:
          azureSubscription: "$(azureSubscription)"
          appType: "webAppLinux"
          appName: "$(webAppName)"
          package: "$(System.DefaultWorkingDirectory)"
          runtimeStack: "NODE|18-lts"
          startUpCommand: "npm start"
```

### Slot Configuration

```bash
# Create deployment slot
az webapp deployment slot create \
  --name myapp \
  --resource-group myResourceGroup \
  --slot staging

# Configure staging slot
az webapp config appsettings set \
  --resource-group myResourceGroup \
  --name myapp \
  --slot staging \
  --settings \
    DOTENV_ENVIRONMENT="staging" \
    DOTENV_PROJECT="my-app" \
    DOTENV_API_KEY="@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/dotenv-api-key-staging/)"

# Swap slots
az webapp deployment slot swap \
  --resource-group myResourceGroup \
  --name myapp \
  --slot staging \
  --target-slot production
```

## Azure Functions

### Function App Configuration

```typescript
// FunctionApp/index.ts
import {
    app,
    HttpRequest,
    HttpResponseInit,
    InvocationContext,
} from "@azure/functions";
import { loadDotEnvSecrets } from "./utils/secrets";

// Load secrets once at cold start
const secretsPromise = loadDotEnvSecrets();

export async function httpTrigger(
    request: HttpRequest,
    context: InvocationContext,
): Promise<HttpResponseInit> {
    await secretsPromise;

    return {
        body: JSON.stringify({
            message: "Function executed successfully",
            hasDatabase: !!process.env.DATABASE_URL,
            functionName: context.functionName,
            invocationId: context.invocationId,
        }),
    };
}

app.http("httpTrigger", {
    methods: ["GET", "POST"],
    authLevel: "anonymous",
    handler: httpTrigger,
});
```

### Durable Functions

```typescript
// DurableFunction/index.ts
import * as df from "durable-functions";
import { loadDotEnvSecrets } from "./utils/secrets";

const orchestrator = df.orchestrator(function* (context) {
    // Load secrets for orchestrator
    yield context.df.callActivity("LoadSecrets");

    const outputs = [];
    outputs.push(yield context.df.callActivity("ProcessTask1"));
    outputs.push(yield context.df.callActivity("ProcessTask2"));
    outputs.push(yield context.df.callActivity("ProcessTask3"));

    return outputs;
});

const loadSecrets = df.activity(async () => {
    await loadDotEnvSecrets();
    return "Secrets loaded";
});

df.app.orchestration("DurableOrchestrator", orchestrator);
df.app.activity("LoadSecrets", { handler: loadSecrets });
```

## Azure Kubernetes Service (AKS)

### Secret Provider Class

```yaml
# secret-provider-class.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
    name: azure-dotenv-secrets
spec:
    provider: azure
    parameters:
        usePodIdentity: "false"
        useVMManagedIdentity: "true"
        userAssignedIdentityID: "<identity-client-id>"
        keyvaultName: "myvault"
        cloudName: "AzurePublicCloud"
        objects: |
            array:
              - |
                objectName: dotenv-api-key
                objectType: secret
                objectAlias: DOTENV_API_KEY
        tenantId: "<tenant-id>"
```

### Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: my-app
spec:
    replicas: 3
    selector:
        matchLabels:
            app: my-app
    template:
        metadata:
            labels:
                app: my-app
                aadpodidbinding: my-app-identity
        spec:
            containers:
                - name: app
                  image: myacr.azurecr.io/my-app:latest
                  env:
                      - name: DOTENV_PROJECT
                        value: "my-app"
                      - name: DOTENV_ENVIRONMENT
                        value: "production"
                  volumeMounts:
                      - name: secrets-store
                        mountPath: "/mnt/secrets-store"
                        readOnly: true
                  lifecycle:
                      postStart:
                          exec:
                              command:
                                  - "/bin/sh"
                                  - "-c"
                                  - |
                                      export DOTENV_API_KEY=$(cat /mnt/secrets-store/DOTENV_API_KEY)
                                      dotenv pull $DOTENV_ENVIRONMENT --project $DOTENV_PROJECT
            volumes:
                - name: secrets-store
                  csi:
                      driver: secrets-store.csi.k8s.io
                      readOnly: true
                      volumeAttributes:
                          secretProviderClass: azure-dotenv-secrets
```

## Azure DevOps

### Pipeline Integration

```yaml
# azure-pipelines.yml
trigger:
    branches:
        include:
            - main
            - develop

variables:
    - group: dotenv-secrets

stages:
    - stage: Build
      jobs:
          - job: BuildJob
            pool:
                vmImage: "ubuntu-latest"
            steps:
                - task: UseDotNet@2
                  inputs:
                      packageType: "sdk"
                      version: "7.x"

                - task: Bash@3
                  displayName: "Install DotEnv CLI"
                  inputs:
                      targetType: "inline"
                      script: |
                          curl -fsSL https://cli.dotenv.cloud/install.sh | sh
                          echo "##vso[task.prependpath]$HOME/.dotenv/bin"

                - task: Bash@3
                  displayName: "Load DotEnv Secrets"
                  inputs:
                      targetType: "inline"
                      script: |
                          dotenv auth login --api-key $(dotenv-api-key)
                          dotenv pull $(Build.SourceBranchName) --project $(Build.Repository.Name)

                          # Export to pipeline variables
                          while IFS='=' read -r key value; do
                            echo "##vso[task.setvariable variable=$key]$value"
                          done < .env

                - task: DotNetCoreCLI@2
                  displayName: "Build"
                  inputs:
                      command: "build"
                      projects: "**/*.csproj"

                - task: DotNetCoreCLI@2
                  displayName: "Test"
                  inputs:
                      command: "test"
                      projects: "**/*Tests.csproj"

    - stage: Deploy
      dependsOn: Build
      condition: succeeded()
      jobs:
          - deployment: DeployJob
            pool:
                vmImage: "ubuntu-latest"
            environment: "production"
            strategy:
                runOnce:
                    deploy:
                        steps:
                            - task: AzureWebApp@1
                              inputs:
                                  azureSubscription: "MyAzureSubscription"
                                  appType: "webApp"
                                  appName: "myapp"
                                  package: "$(Pipeline.Workspace)/drop"
```

### Library Variable Groups

```bash
# Create variable group from DotEnv
dotenv pull production --project my-app --format json > secrets.json

# Import to Azure DevOps
az pipelines variable-group create \
  --name dotenv-production \
  --variables @secrets.json \
  --organization https://dev.azure.com/myorg \
  --project myproject
```

## Security Best Practices

### 1. Managed Identity

Configure managed identity:

```bash
# Enable system-assigned identity
az webapp identity assign \
  --resource-group myResourceGroup \
  --name myapp

# Grant Key Vault access
az keyvault set-policy \
  --name myvault \
  --object-id <identity-object-id> \
  --secret-permissions get list
```

### 2. Private Endpoints

```bash
# Create private endpoint for Key Vault
az network private-endpoint create \
  --resource-group myResourceGroup \
  --name myKeyvaultEndpoint \
  --vnet-name myVNet \
  --subnet mySubnet \
  --private-connection-resource-id $(az keyvault show \
    --resource-group myResourceGroup \
    --name myvault \
    --query id -o tsv) \
  --connection-name myConnection \
  --group-id vault
```

### 3. Azure Policy

```json
{
    "properties": {
        "displayName": "Require DotEnv tag on Key Vault secrets",
        "policyType": "Custom",
        "mode": "All",
        "parameters": {},
        "policyRule": {
            "if": {
                "allOf": [
                    {
                        "field": "type",
                        "equals": "Microsoft.KeyVault/vaults/secrets"
                    },
                    {
                        "field": "tags['ManagedBy']",
                        "notEquals": "DotEnv"
                    }
                ]
            },
            "then": {
                "effect": "deny"
            }
        }
    }
}
```

### 4. Diagnostic Logging

```bash
# Enable diagnostic logging
az monitor diagnostic-settings create \
  --resource $(az keyvault show --name myvault --query id -o tsv) \
  --name myDiagSetting \
  --logs '[{"category": "AuditEvent", "enabled": true}]' \
  --workspace $(az monitor log-analytics workspace show \
    --resource-group myResourceGroup \
    --workspace-name myWorkspace \
    --query id -o tsv)
```

## Monitoring

### Application Insights

```javascript
// utils/monitoring.js
const appInsights = require("applicationinsights");

// Initialize Application Insights
appInsights
    .setup(process.env.APPLICATIONINSIGHTS_CONNECTION_STRING)
    .setAutoDependencyCorrelation(true)
    .setAutoCollectRequests(true)
    .setAutoCollectPerformance(true, true)
    .setAutoCollectExceptions(true)
    .setAutoCollectDependencies(true)
    .setAutoCollectConsole(true)
    .setUseDiskRetryCaching(true)
    .setSendLiveMetrics(false)
    .setDistributedTracingMode(appInsights.DistributedTracingModes.AI_AND_W3C);

appInsights.defaultClient.setAuthenticationHandler((envelope) => {
    // Add custom properties
    envelope.tags["ai.cloud.role"] = process.env.APP_NAME;
    envelope.tags["ai.cloud.roleInstance"] = process.env.WEBSITE_INSTANCE_ID;
});

appInsights.start();

// Track custom events
function trackSecretLoad(success, duration) {
    appInsights.defaultClient.trackEvent({
        name: "SecretLoad",
        properties: {
            success: success,
            project: process.env.DOTENV_PROJECT,
            environment: process.env.DOTENV_ENVIRONMENT,
        },
        measurements: {
            duration: duration,
        },
    });
}

module.exports = { trackSecretLoad };
```

### Azure Monitor Alerts

```bash
# Create metric alert
az monitor metrics alert create \
  --name dotenv-secret-load-failure \
  --resource-group myResourceGroup \
  --scopes $(az webapp show --name myapp --resource-group myResourceGroup --query id -o tsv) \
  --condition "count customMetrics/SecretLoadFailure > 5" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action-group myActionGroup
```

## Troubleshooting

### Common Issues

#### Key Vault Access Denied

```bash
# Check managed identity
az webapp identity show \
  --resource-group myResourceGroup \
  --name myapp

# Verify Key Vault access policy
az keyvault show \
  --name myvault \
  --query properties.accessPolicies
```

#### Function App Cold Start

```javascript
// Optimize cold start
const { DefaultAzureCredential } = require("@azure/identity");

// Initialize credential outside handler
const credential = new DefaultAzureCredential();

// Pre-warm connections
const preWarm = async () => {
    // Pre-authenticate
    await credential.getToken("https://vault.azure.net/.default");
};

preWarm().catch(console.error);
```

## Examples

### Complete .NET Application

```csharp
// Program.cs
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

var builder = WebApplication.CreateBuilder(args);

// Add Key Vault
if (!string.IsNullOrEmpty(builder.Configuration["KeyVaultName"]))
{
    var keyVaultName = builder.Configuration["KeyVaultName"];
    var kvUri = $"https://{keyVaultName}.vault.azure.net";

    builder.Configuration.AddAzureKeyVault(
        new Uri(kvUri),
        new DefaultAzureCredential());
}

// Load DotEnv secrets
var dotEnvLoader = new DotEnvLoader(
    builder.Configuration["DotEnv:ApiKey"],
    builder.Configuration["DotEnv:Project"],
    builder.Configuration["DotEnv:Environment"]
);

await dotEnvLoader.LoadSecretsAsync();

builder.Services.AddControllers();

var app = builder.Build();

app.MapControllers();
app.Run();
```

### Bicep Template

```bicep
// main.bicep
param location string = resourceGroup().location
param appName string
param dotEnvApiKey string

resource keyVault 'Microsoft.KeyVault/vaults@2022-07-01' = {
  name: '${appName}-kv'
  location: location
  properties: {
    sku: {
      name: 'standard'
      family: 'A'
    }
    tenantId: subscription().tenantId
    accessPolicies: []
  }
}

resource dotEnvApiKeySecret 'Microsoft.KeyVault/vaults/secrets@2022-07-01' = {
  parent: keyVault
  name: 'dotenv-api-key'
  properties: {
    value: dotEnvApiKey
    attributes: {
      enabled: true
    }
    tags: {
      ManagedBy: 'DotEnv'
    }
  }
}

resource appServicePlan 'Microsoft.Web/serverfarms@2022-03-01' = {
  name: '${appName}-plan'
  location: location
  sku: {
    name: 'P1V2'
    tier: 'PremiumV2'
  }
}

resource webApp 'Microsoft.Web/sites@2022-03-01' = {
  name: appName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      appSettings: [
        {
          name: 'DOTENV_PROJECT'
          value: appName
        }
        {
          name: 'DOTENV_ENVIRONMENT'
          value: 'production'
        }
        {
          name: 'DOTENV_API_KEY'
          value: '@Microsoft.KeyVault(VaultName=${keyVault.name};SecretName=dotenv-api-key)'
        }
      ]
    }
  }
}

// Grant Key Vault access to App Service
resource keyVaultAccessPolicy 'Microsoft.KeyVault/vaults/accessPolicies@2022-07-01' = {
  parent: keyVault
  name: 'add'
  properties: {
    accessPolicies: [
      {
        tenantId: subscription().tenantId
        objectId: webApp.identity.principalId
        permissions: {
          secrets: ['get', 'list']
        }
      }
    ]
  }
}
```

## Resources

- [Azure Documentation](https://docs.microsoft.com/azure/)
- [Key Vault Documentation](https://docs.microsoft.com/azure/key-vault/)
- [App Service Documentation](https://docs.microsoft.com/azure/app-service/)
- [DotEnv CLI Reference](/documentation/v1/cli/commands)

## Next Steps

- [Configure Key Vault sync](#key-vault-integration)
- [Deploy to App Service](#app-service)
- [Set up Azure Functions](#azure-functions)
- [Enable monitoring](#monitoring)
