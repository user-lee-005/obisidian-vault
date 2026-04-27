# Azure Deployment

> **Beginner-friendly guide** to deploying applications on Azure — from the simplest "push code and go" to full enterprise CI/CD pipelines with Infrastructure as Code.

---

## Table of Contents

- [[#Azure Deployment Options Overview]]
- [[#Deploying to Azure App Service]]
- [[#Azure Container Registry (ACR)]]
- [[#Azure DevOps Pipelines]]
- [[#ARM Templates (Azure Resource Manager)]]
- [[#Bicep — Simpler Infrastructure as Code]]
- [[#End-to-End Deployment Walkthrough]]
- [[#Deployment Best Practices]]

---

## Azure Deployment Options Overview

"Deployment" means getting your code from your laptop (or a repository) into Azure so users can access it. Azure gives you many options, ranging from **simplest** to **most control**:

| Option | Type | You Manage | Azure Manages | Best For |
|--------|------|-----------|---------------|----------|
| **App Service** | PaaS | Code only | OS, Runtime, Scaling | Web apps, APIs |
| **Container Instances (ACI)** | CaaS | Container | Infrastructure | Quick container runs |
| **Azure Kubernetes (AKS)** | CaaS | Containers, Orchestration | Master nodes | Microservices at scale |
| **Virtual Machines** | IaaS | Everything | Hardware | Full control, legacy apps |
| **ARM Templates / Bicep** | IaC | Template files | Provisioning | Repeatable infrastructure |
| **Azure DevOps Pipelines** | CI/CD | Pipeline config | Build agents | Automated deployments |

### Think of it like opening a restaurant:

- **App Service** → You cook the food, Azure owns the building, tables, and staff
- **Containers (ACI/AKS)** → You bring a food truck (container), Azure provides the parking lot
- **VMs** → You rent an empty building and set up everything yourself
- **ARM/Bicep** → You write a blueprint, Azure builds exactly what you described
- **Azure DevOps** → You set up a conveyor belt that automatically cooks and serves food whenever a new recipe arrives

> 💡 **Key Insight**: Most beginners should start with **App Service**. It's the fastest way to get an app running on Azure.

---

## Deploying to Azure App Service

Azure App Service is a **Platform as a Service (PaaS)** — you provide code, Azure handles servers, OS patching, scaling, and more. See [[Azure Compute]] for how App Service fits into Azure's compute options.

### Prerequisites

Before deploying, you need:

1. **Azure CLI installed** — see [[Azure Getting Started]] for setup
2. **An Azure subscription** — free tier works
3. **A resource group** — logical container for your resources

```bash
# Login to Azure (if not already)
az login

# Create a resource group (if you don't have one)
az resource group create --name myResourceGroup --location centralindia
```

### Method 1: Quick Deploy with `az webapp up` (Easiest!)

This is the **fastest** way to deploy. One command does everything — creates the App Service Plan, creates the web app, and deploys your code.

```bash
# Navigate to your project folder first
cd /path/to/your/project

# Deploy! Azure auto-detects your language
az webapp up --name myuniqueapp123 --resource-group myResourceGroup --runtime "PYTHON:3.12" --sku F1
```

**What this does behind the scenes:**
1. Creates an App Service Plan (the server) with SKU `F1` (free tier)
2. Creates a Web App named `myuniqueapp123`
3. Zips your current directory
4. Uploads and deploys the zip to Azure
5. Your app is live at: `https://myuniqueapp123.azurewebsites.net`

> ⚠️ **Important**: The app name must be **globally unique** across ALL of Azure because it becomes part of the URL.

**Common runtimes:**
- `"PYTHON:3.12"` — Python
- `"NODE:20-lts"` — Node.js
- `"JAVA:17-java17"` — Java
- `"DOTNETCORE:8.0"` — .NET

### Method 2: ZIP Deploy (Manual Control)

If you want more control over what gets deployed:

```bash
# Step 1: Create the zip file (exclude unnecessary files)
zip -r app.zip . -x '.git/*' 'node_modules/*' '__pycache__/*' '.env'

# Step 2: Deploy the zip
az webapp deployment source config-zip \
  --resource-group myResourceGroup \
  --name myuniqueapp123 \
  --src app.zip
```

**Why use ZIP deploy over `az webapp up`?**
- You can control exactly what files are included
- You can zip once and deploy to multiple environments
- Better for CI/CD pipelines where you want to build artifacts

### Method 3: Deploy from GitHub (Continuous Deployment)

Connect your GitHub repository so Azure **automatically deploys** whenever you push to a branch:

```bash
# Connect GitHub repo to your web app
az webapp deployment source config \
  --name myuniqueapp123 \
  --resource-group myResourceGroup \
  --repo-url https://github.com/user/repo \
  --branch main \
  --manual-integration
```

**What happens:**
- Azure pulls code from the specified branch
- Every time you push to `main`, Azure re-deploys
- `--manual-integration` means no webhook (you trigger deploys manually)
- Remove `--manual-integration` for automatic deployment on every push

```bash
# Trigger a manual sync (pull latest code)
az webapp deployment source sync --name myuniqueapp123 --resource-group myResourceGroup

# Check deployment status
az webapp deployment source show --name myuniqueapp123 --resource-group myResourceGroup
```

### Method 4: Deployment Slots (Zero-Downtime Deployments)

**The Problem:** If you deploy directly to production and something breaks, users see errors.

**The Solution:** Deployment Slots — deploy to a **staging** copy, test it, then **swap** it to production instantly.

```
Production Slot (live users)    ←── swap ──→    Staging Slot (your testing)
https://myapp.azurewebsites.net                 https://myapp-staging.azurewebsites.net
```

```bash
# Step 1: Create a staging slot
az webapp deployment slot create \
  --name myuniqueapp123 \
  --resource-group myResourceGroup \
  --slot staging

# Step 2: Deploy your new version to staging
az webapp deployment source config-zip \
  --resource-group myResourceGroup \
  --name myuniqueapp123 \
  --slot staging \
  --src app.zip

# Step 3: Test staging
# Visit: https://myuniqueapp123-staging.azurewebsites.net
# Run your tests against the staging URL
curl https://myuniqueapp123-staging.azurewebsites.net/health

# Step 4: Swap staging to production (zero downtime!)
az webapp deployment slot swap \
  --name myuniqueapp123 \
  --resource-group myResourceGroup \
  --slot staging \
  --target-slot production
```

**Why this is amazing:**
- Users never see downtime — the swap is instant
- If something goes wrong, swap back! (production becomes staging)
- You can test with production settings before going live

> 💡 **Note:** Deployment slots require **Standard tier or higher** (not available on Free/Basic).

### Method 5: Docker Container to App Service

If your app is packaged as a Docker container:

```bash
# Create an App Service running a Docker container
az webapp create \
  --resource-group myResourceGroup \
  --plan myPlan \
  --name myuniqueapp123 \
  --deployment-container-image-name nginx:latest

# Update to a different image later
az webapp config container set \
  --name myuniqueapp123 \
  --resource-group myResourceGroup \
  --docker-custom-image-name myacr123.azurecr.io/myapp:v2
```

This pulls an image from Docker Hub (or Azure Container Registry) and runs it on App Service. See the [[#Azure Container Registry (ACR)]] section below for using private images.

---

## Azure Container Registry (ACR)

ACR is Azure's **private Docker registry** — like Docker Hub, but private and integrated with Azure services.

### Why use ACR instead of Docker Hub?

- **Private** — your images aren't public
- **Integrated** — works seamlessly with App Service, AKS, ACI
- **Geo-replication** — store images close to your deployment regions
- **Security** — vulnerability scanning, signed images

### Creating and Using ACR

```bash
# Step 1: Create a Container Registry
az acr create \
  --resource-group myResourceGroup \
  --name myacr123 \
  --sku Basic

# Step 2: Login to ACR (authenticates Docker to push/pull)
az acr login --name myacr123

# Step 3: Build your Docker image locally
docker build -t myapp:latest .

# Step 4: Tag the image for ACR
# Format: <acr-name>.azurecr.io/<image-name>:<tag>
docker tag myapp:latest myacr123.azurecr.io/myapp:v1

# Step 5: Push the image to ACR
docker push myacr123.azurecr.io/myapp:v1

# Step 6: Verify the image is in ACR
az acr repository list --name myacr123 --output table

# List tags for a specific image
az acr repository show-tags --name myacr123 --repository myapp --output table
```

### ACR Build (Build in the Cloud)

You don't even need Docker installed locally! ACR can build images for you:

```bash
# Build directly in Azure (sends your Dockerfile + context to ACR)
az acr build --registry myacr123 --image myapp:v1 .
```

### Deploy ACR Image to App Service

```bash
# Create app service with ACR image
az webapp create \
  --resource-group myResourceGroup \
  --plan myPlan \
  --name myuniqueapp123 \
  --deployment-container-image-name myacr123.azurecr.io/myapp:v1

# Grant App Service permission to pull from ACR
az webapp config container set \
  --name myuniqueapp123 \
  --resource-group myResourceGroup \
  --docker-custom-image-name myacr123.azurecr.io/myapp:v1 \
  --docker-registry-server-url https://myacr123.azurecr.io
```

---

## Azure DevOps Pipelines

Azure DevOps is Microsoft's **full DevOps platform**. Think of it as GitHub + Jenkins + Jira all in one.

### Azure DevOps Services

| Service | Purpose | Similar To |
|---------|---------|------------|
| **Azure Boards** | Work tracking, sprints | Jira, Trello |
| **Azure Repos** | Git repositories | GitHub, GitLab |
| **Azure Pipelines** | CI/CD automation | Jenkins, GitHub Actions |
| **Azure Test Plans** | Manual/automated testing | TestRail |
| **Azure Artifacts** | Package management | npm, Maven registries |

### What is CI/CD?

- **CI (Continuous Integration)** — Automatically build and test code whenever someone pushes changes
- **CD (Continuous Deployment)** — Automatically deploy tested code to production

```
Developer pushes code → Build → Test → Deploy to Staging → Deploy to Production
         ↑                            ↑                              ↑
    Triggered by push          Automated tests              Manual approval gate
```

### Azure Pipelines YAML Example

Create a file called `azure-pipelines.yml` in your repo root:

```yaml
# azure-pipelines.yml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

steps:
  - task: UsePythonVersion@0
    inputs:
      versionSpec: '3.12'
  
  - script: |
      python -m pip install --upgrade pip
      pip install -r requirements.txt
    displayName: 'Install dependencies'
  
  - script: |
      python -m pytest tests/
    displayName: 'Run tests'
  
  - task: AzureWebApp@1
    inputs:
      azureSubscription: 'my-service-connection'
      appName: 'myuniqueapp123'
      package: '$(System.DefaultWorkingDirectory)'
```

**How this works:**
1. **Trigger** — Pipeline runs when code is pushed to `main`
2. **Pool** — Uses a Microsoft-hosted Ubuntu VM to run the build
3. **Steps:**
   - Install Python 3.12
   - Install project dependencies
   - Run tests (pipeline FAILS if tests fail — code doesn't get deployed)
   - If tests pass, deploy to Azure App Service

### GitHub Actions Alternative

If you use GitHub (not Azure Repos), you can use GitHub Actions to deploy to Azure:

```yaml
# .github/workflows/deploy.yml
name: Deploy to Azure

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      
      - uses: azure/webapps-deploy@v2
        with:
          app-name: 'myuniqueapp123'
          package: '.'
```

**Setup:**
1. Create a Service Principal for GitHub to authenticate with Azure:
   ```bash
   az ad sp create-for-rbac --name "github-deploy" --role contributor \
     --scopes /subscriptions/<sub-id>/resourceGroups/myResourceGroup \
     --sdk-auth
   ```
2. Copy the JSON output and add it as a GitHub secret named `AZURE_CREDENTIALS`
3. Push the workflow file to your repo — it deploys on every push to `main`

---

## ARM Templates (Azure Resource Manager)

ARM Templates are **Infrastructure as Code (IaC)** — you describe your infrastructure in a JSON file, and Azure creates it exactly as described.

### Why Infrastructure as Code?

Without IaC (clicking in the portal):
- "I set it up in dev, but how do I recreate it in prod?" 😰
- "Who changed that setting?" 🤷
- "It works in my environment..." 🫠

With IaC:
- **Repeatable** — Same template → same infrastructure, every time
- **Version controlled** — Track changes in Git
- **Reviewable** — Team can review infrastructure changes like code
- **Automated** — Deploy infrastructure with a single command

### ARM Template Structure

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "vmName": {
      "type": "string",
      "defaultValue": "myVM"
    }
  },
  "variables": {
    "vmSize": "Standard_B1s"
  },
  "resources": [
    {
      "type": "Microsoft.Compute/virtualMachines",
      "apiVersion": "2023-07-01",
      "name": "[parameters('vmName')]",
      "location": "[resourceGroup().location]",
      "properties": {
        "hardwareProfile": {
          "vmSize": "[variables('vmSize')]"
        }
      }
    }
  ],
  "outputs": {
    "vmId": {
      "type": "string",
      "value": "[resourceId('Microsoft.Compute/virtualMachines', parameters('vmName'))]"
    }
  }
}
```

**Sections explained:**
- `parameters` — Values you can change at deploy time (like function arguments)
- `variables` — Internal values computed from parameters
- `resources` — The actual Azure resources to create
- `outputs` — Values to display after deployment (like resource IDs, URLs)

### Deploying an ARM Template

```bash
# Deploy the template
az deployment group create \
  --resource-group myResourceGroup \
  --template-file template.json \
  --parameters vmName=myNewVM

# Validate without deploying (dry run)
az deployment group validate \
  --resource-group myResourceGroup \
  --template-file template.json

# See what would change (what-if)
az deployment group what-if \
  --resource-group myResourceGroup \
  --template-file template.json
```

> 💡 **Declarative vs Imperative**: ARM templates are **declarative** — you say WHAT you want, not HOW to do it. Azure figures out the order of operations, handles dependencies, and creates resources in parallel when possible.

---

## Bicep — Simpler Infrastructure as Code

Bicep is Microsoft's **newer IaC language** that compiles to ARM JSON. It's much cleaner and easier to read.

### ARM JSON vs Bicep — Same Thing, Better Syntax

**ARM Template (JSON) — 25 lines for a storage account:**
```json
{
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2022-09-01",
      "name": "mystorage123",
      "location": "[resourceGroup().location]",
      "sku": { "name": "Standard_LRS" },
      "kind": "StorageV2"
    }
  ]
}
```

**Bicep — 7 lines for the same storage account:**
```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2022-09-01' = {
  name: 'mystorage123'
  location: resourceGroup().location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}
```

### Full Bicep Example — Web App with App Service Plan

Create a file called `main.bicep`:

```bicep
@description('The name of the web app')
param appName string = 'mywebapp${uniqueString(resourceGroup().id)}'
param location string = resourceGroup().location
param sku string = 'F1'

resource appServicePlan 'Microsoft.Web/serverfarms@2022-03-01' = {
  name: '${appName}-plan'
  location: location
  sku: {
    name: sku
  }
  kind: 'linux'
  properties: {
    reserved: true
  }
}

resource webApp 'Microsoft.Web/sites@2022-03-01' = {
  name: appName
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'PYTHON|3.12'
    }
  }
}

output webAppUrl string = 'https://${webApp.properties.defaultHostName}'
```

**Deploy:**
```bash
# Deploy the Bicep file (Azure compiles it to ARM automatically)
az deployment group create \
  --resource-group myResourceGroup \
  --template-file main.bicep

# Deploy with custom parameters
az deployment group create \
  --resource-group myResourceGroup \
  --template-file main.bicep \
  --parameters appName=myprodapp sku=S1

# Preview changes before deploying
az deployment group what-if \
  --resource-group myResourceGroup \
  --template-file main.bicep
```

### ARM vs Bicep vs Terraform Comparison

| Feature | ARM Templates | Bicep | Terraform |
|---------|-------------|-------|-----------|
| **Language** | JSON | Domain-specific | HCL |
| **Readability** | ❌ Verbose | ✅ Clean | ✅ Clean |
| **Multi-cloud** | ❌ Azure only | ❌ Azure only | ✅ Any cloud |
| **State file** | ❌ No (stateless) | ❌ No (stateless) | ✅ Yes (tracks state) |
| **Learning curve** | Medium | Low | Medium |
| **Microsoft support** | ✅ First-party | ✅ First-party | Third-party |
| **Modules/reuse** | Limited | ✅ Good | ✅ Excellent |

> 💡 **Recommendation for beginners:**
> - Azure only? → Use **Bicep**
> - Multi-cloud? → Use **Terraform**
> - Don't use raw ARM JSON unless you have to

---

## End-to-End Deployment Walkthrough

### Option A: Quick Deploy — `az webapp up` (5 Minutes)

**Best for:** Learning, demos, personal projects.

```bash
# 1. Login
az login

# 2. Create resource group
az group create --name quickdeploy-rg --location centralindia

# 3. Navigate to your project and deploy
cd /path/to/your/python-app
az webapp up --name myquickapp123 --resource-group quickdeploy-rg --runtime "PYTHON:3.12" --sku F1

# 4. Open your app
az webapp browse --name myquickapp123 --resource-group quickdeploy-rg

# 5. Clean up when done
az group delete --name quickdeploy-rg --yes --no-wait
```

### Option B: Container Deployment — Docker → ACR → App Service

**Best for:** Apps with specific dependencies, microservices.

```bash
# 1. Create resources
az group create --name container-rg --location centralindia
az acr create --resource-group container-rg --name mycontaineracr --sku Basic
az appservice plan create --name mycontainerplan --resource-group container-rg --is-linux --sku B1

# 2. Build and push Docker image
az acr login --name mycontaineracr
az acr build --registry mycontaineracr --image myapp:v1 .

# 3. Deploy container to App Service
az webapp create --resource-group container-rg --plan mycontainerplan \
  --name mycontainerapp123 \
  --deployment-container-image-name mycontaineracr.azurecr.io/myapp:v1

# 4. Enable continuous deployment from ACR
az webapp deployment container config \
  --name mycontainerapp123 \
  --resource-group container-rg \
  --enable-cd true

# 5. Clean up
az group delete --name container-rg --yes --no-wait
```

### Option C: Enterprise — Bicep + Azure DevOps Pipeline

**Best for:** Production applications, team environments.

1. Define infrastructure in `main.bicep`
2. Define deployment pipeline in `azure-pipelines.yml`
3. Code changes trigger the pipeline automatically
4. Pipeline builds, tests, deploys to staging, then swaps to production

```
Developer pushes → Azure DevOps Pipeline triggers
  ├── Stage 1: Build & Test
  ├── Stage 2: Deploy Bicep (create/update infrastructure)
  ├── Stage 3: Deploy app to Staging slot
  ├── Stage 4: Run smoke tests on staging
  └── Stage 5: Swap staging → production (manual approval)
```

---

## Deployment Best Practices

### 1. Use Infrastructure as Code (IaC) for Everything

```bash
# ✅ Good: Define infrastructure in code
az deployment group create --template-file main.bicep

# ❌ Bad: Click around the portal to create resources
# (Not repeatable, not trackable, error-prone)
```

**Prefer Bicep** over raw ARM templates — it's cleaner and Microsoft's recommended approach.

### 2. Always Use CI/CD Pipelines

Never deploy to production by running manual commands. Set up a pipeline that:
- Runs tests automatically
- Deploys to staging first
- Requires approval before production
- Can roll back automatically

### 3. Use Deployment Slots for Zero Downtime

```bash
# Deploy to staging, test, then swap
az webapp deployment slot swap --name myapp --slot staging --target-slot production
```

### 4. Separate Environments

Use **different resource groups** for each environment:

```bash
az group create --name myapp-dev --location centralindia
az group create --name myapp-staging --location centralindia
az group create --name myapp-prod --location centralindia
```

### 5. Tag All Resources

Tags help you identify, organize, and track costs:

```bash
az resource tag --tags Environment=Production Team=Backend Project=MyApp \
  --resource-group myResourceGroup --name myuniqueapp123 \
  --resource-type Microsoft.Web/sites
```

### 6. Clean Up Resources When Done

Azure charges by the hour/minute. Always delete resources you're not using:

```bash
# Delete everything in a resource group (nuclear option)
az group delete --name myResourceGroup --yes --no-wait

# Or delete individual resources
az webapp delete --name myuniqueapp123 --resource-group myResourceGroup
```

> ⚠️ **Critical for beginners:** If you're learning and using a free trial, ALWAYS delete resource groups when you're done experimenting. It's the easiest way to avoid unexpected charges.

### 7. Naming Conventions

Use consistent naming for resources:

```
<project>-<environment>-<resource-type>-<region>
Examples:
  myapp-prod-rg-centralindia        (Resource Group)
  myapp-prod-plan-centralindia      (App Service Plan)
  myapp-prod-web-centralindia       (Web App)
  myapp-prod-acr                    (Container Registry)
```

---

## Quick Reference — Deployment Commands Cheat Sheet

```bash
# === App Service ===
az webapp up --name <app> --resource-group <rg> --runtime "PYTHON:3.12" --sku F1
az webapp deployment source config-zip --name <app> -g <rg> --src app.zip
az webapp browse --name <app> --resource-group <rg>
az webapp log tail --name <app> --resource-group <rg>

# === Deployment Slots ===
az webapp deployment slot create --name <app> -g <rg> --slot staging
az webapp deployment slot swap --name <app> -g <rg> --slot staging

# === Container Registry ===
az acr create -g <rg> --name <acr> --sku Basic
az acr login --name <acr>
az acr build --registry <acr> --image myapp:v1 .

# === Infrastructure as Code ===
az deployment group create -g <rg> --template-file main.bicep
az deployment group what-if -g <rg> --template-file main.bicep
az deployment group validate -g <rg> --template-file main.bicep

# === Cleanup ===
az group delete --name <rg> --yes --no-wait
```

---

## Related Notes

- [[Cloud Deployment Strategies]] — General deployment concepts across all clouds
- [[Azure Getting Started]] — Setting up Azure CLI, subscriptions, and resource groups
- [[Azure Compute]] — Deep dive into App Service, VMs, AKS, and other compute options
- [[AWS Deployment]] — Compare Azure deployment with AWS (Elastic Beanstalk, ECS, CloudFormation)
- [[Azure Monitoring]] — Monitor your deployments after they're live
- [[Azure Security]] — Secure your deployment pipeline and resources

---

> **Next Steps:** Once you've deployed an app, learn how to monitor it → [[Azure Monitoring]]
