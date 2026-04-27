# Azure Getting Started

> **Your first steps into Microsoft Azure — from zero to deploying resources.**
> If you're brand new to cloud, start with [[Cloud Fundamentals]] first, then come back here.

---

## Table of Contents

- [[#What is Azure?]]
- [[#Creating an Azure Account]]
- [[#The Azure Portal — A Guided Tour]]
- [[#Azure Free Tier — What You Get for Free]]
- [[#Key Azure Concepts]]
- [[#Installing the Azure CLI]]
- [[#Login and Configure Azure CLI]]
- [[#Your First CLI Commands]]
- [[#Azure vs AWS Quick Comparison]]

---

## What is Azure?

**Microsoft Azure** is Microsoft's cloud computing platform. Think of it as a massive collection of
computers, storage, and services that Microsoft owns and maintains in data centers around the world —
and you can rent pieces of it on-demand, paying only for what you use.

### Key Facts

- **Launched**: February 2010 (originally called "Windows Azure", renamed to "Microsoft Azure" in 2014)
- **Services**: 200+ cloud services covering compute, storage, databases, AI, networking, and more
- **Market Position**: Second largest cloud provider globally (behind AWS, ahead of Google Cloud)
- **Data Centers**: 60+ regions worldwide — more regions than any other cloud provider
- **Revenue**: One of the fastest-growing segments of Microsoft's business

### Why Azure?

1. **Enterprise Integration** — If your company already uses Microsoft 365, Active Directory, or
   Windows Server, Azure integrates seamlessly with all of them.
2. **Hybrid Cloud** — Azure has the strongest hybrid cloud story (connecting on-premises data
   centers with the cloud) via Azure Arc and Azure Stack.
3. **Developer Tools** — Deep integration with Visual Studio, VS Code, GitHub, and Azure DevOps.
4. **Compliance** — More compliance certifications than any other cloud provider (important for
   regulated industries like healthcare, finance, government).
5. **Global Reach** — 60+ regions means you can deploy close to your users anywhere in the world.

> 💡 **Beginner Tip**: You don't need to know all 200+ services. Most projects use only 5-10
> services. Start with the basics (VMs, Storage, App Service) and expand from there.

---

## Creating an Azure Account

Follow these steps to create your **free** Azure account:

### Step 1: Go to the Azure Website

Open your browser and navigate to:

```
https://azure.microsoft.com/en-us/free/
```

Click the big **"Start Free"** button (or "Try Azure for Free").

### Step 2: Sign In with a Microsoft Account

- If you already have a Microsoft account (Outlook, Hotmail, Xbox, etc.), sign in with it.
- If you don't have one, click **"Create one!"** and follow the prompts to make a new Microsoft
  account. You'll need an email address and a password.

### Step 3: Identity Verification — Phone

- Azure will ask you to verify your identity via phone.
- Enter your phone number.
- Choose either **Text me** or **Call me** for verification.
- Enter the verification code you receive.

### Step 4: Identity Verification — Credit Card

- You **must** provide a credit card. This is for identity verification only.
- ⚠️ **You will NOT be charged** unless you explicitly upgrade to a pay-as-you-go subscription.
- The free tier has a spending limit that prevents accidental charges.
- Supported: Visa, Mastercard, American Express (no debit cards in some regions).

### Step 5: Agreement and Sign Up

- Check the agreement box.
- Click **"Sign up"**.
- You'll be redirected to the Azure Portal — congratulations, you're in! 🎉

### What You Get

| Benefit | Details |
|---------|---------|
| **$200 Credit** | Use on any Azure service for the first 30 days |
| **12 Months Free** | Popular services free for 12 months (within limits) |
| **Always Free** | 55+ services that are always free (within limits) |

---

## The Azure Portal — A Guided Tour

The **Azure Portal** is the web-based interface where you manage everything in Azure.

```
URL: https://portal.azure.com
```

When you first log in, here's what you'll see:

### Dashboard (Home Page)

- This is your customizable home page.
- You can pin frequently used resources, charts, and monitoring dashboards.
- Click **"+ New dashboard"** to create custom views.
- Think of it like a customizable desktop for your cloud resources.

### The Top Navigation Bar

| Element | What It Does |
|---------|-------------|
| **Search bar** (🔍) | Search for ANY service, resource, or documentation. Use this constantly! |
| **Cloud Shell** (>_ icon) | Opens a terminal directly in your browser — no local setup needed! |
| **Notifications** (🔔) | Shows deployment progress, errors, and alerts |
| **Settings** (⚙️) | Theme, language, portal settings |
| **Directory + Subscription** | Switch between tenants and subscriptions |

### Left Sidebar

- **+ Create a resource** — Start here to create anything new
- **Home** — Back to dashboard
- **All services** — Browse every Azure service by category
- **All resources** — See everything you've created across all resource groups
- **Resource groups** — View and manage your resource groups
- **Subscriptions** — Manage billing and subscription settings
- **Microsoft Entra ID** — Manage users, groups, and identity (see [[Azure IAM]])

### Cloud Shell — Your Browser-Based Terminal

This is one of Azure's best features for beginners:

1. Click the **>_** icon in the top navigation bar.
2. First time: Choose **Bash** or **PowerShell** (choose Bash if unsure).
3. It will ask you to create a storage account (this is for persisting your files — say yes).
4. A terminal opens at the bottom of your browser!
5. Azure CLI (`az`) and many other tools are pre-installed.

> 💡 **Beginner Tip**: Cloud Shell means you can manage Azure from any computer with a browser —
> even a Chromebook or a tablet. No software to install!

### Resource Groups — A Concept Unique to Azure

This is one of the **most important concepts** in Azure and is different from AWS:

- A **Resource Group** is a logical container that holds related Azure resources.
- **Every single resource** in Azure must belong to a resource group. No exceptions.
- Think of it like a folder on your computer — you organize related files into folders.
- **Delete the resource group = delete EVERYTHING inside it** (this is a feature, not a bug!).

Example: For a web application project, you might create a resource group called `webapp-rg` and
put the VM, database, storage account, and network all inside it.

### Subscriptions — The Billing Boundary

- A **Subscription** is how Azure organizes billing.
- Every resource belongs to a resource group, and every resource group belongs to a subscription.
- You can have multiple subscriptions (e.g., one for development, one for production).
- Your free account comes with one subscription called "Azure subscription 1".

---

## Azure Free Tier — What You Get for Free

### Tier 1: $200 Credit (First 30 Days)

- Use on **any** Azure service — no restrictions.
- Great for experimenting and learning.
- Once it runs out or 30 days pass (whichever comes first), unused credit expires.

### Tier 2: 12 Months Free (Popular Services)

These services are free for 12 months within specified limits:

| Service | Free Amount |
|---------|------------|
| **Linux Virtual Machine** (B1S) | 750 hours/month |
| **Windows Virtual Machine** (B1S) | 750 hours/month |
| **Blob Storage** | 5 GB LRS hot |
| **Azure SQL Database** | 250 GB (S0 instance) |
| **Cosmos DB** | 25 GB storage + 1000 RU/s |
| **Bandwidth** | 15 GB outbound data transfer |
| **Azure Files** | 5 GB LRS file storage |

### Tier 3: Always Free (No Time Limit)

These are free forever, within limits:

| Service | Free Amount |
|---------|------------|
| **Azure Functions** | 1 million executions/month |
| **Azure DevOps** | 5 users + unlimited private repos |
| **Azure Cosmos DB** | 1000 RU/s throughput + 25 GB |
| **App Service** | 10 web/mobile/API apps (F1 tier) |
| **Azure Kubernetes Service** | Free cluster management |
| **Azure Active Directory** | Free tier (50,000 objects) |
| **Visual Studio Code** | Free forever (not technically Azure, but useful!) |

### ⚠️ Avoiding Surprise Bills

This is the #1 fear of cloud beginners. Here's how to protect yourself:

1. **Set Budget Alerts**:
   - Go to **Subscriptions** → your subscription → **Budgets**
   - Create a budget (e.g., $10/month)
   - Set alerts at 50%, 80%, 100% of your budget
   - You'll get email notifications

2. **Check Cost Analysis Regularly**:
   - Go to **Subscriptions** → **Cost analysis**
   - See exactly what's costing money

3. **Delete What You Don't Need**:
   - The easiest way: delete the entire resource group when done experimenting
   - Stopped VMs still cost money for storage! Use `az vm deallocate` to truly stop costs.

4. **Use the Spending Limit**:
   - Free accounts have a spending limit by default
   - Don't remove it unless you understand the implications

---

## Key Azure Concepts

### The Azure Resource Hierarchy

This is how Azure organizes everything — understand this and everything else makes sense:

```
Management Groups          ← Organize multiple subscriptions
    └── Subscriptions      ← Billing boundary
        └── Resource Groups   ← Logical container for resources
            └── Resources     ← The actual services (VMs, databases, etc.)
```

### Subscriptions

- A **Subscription** is the billing unit in Azure.
- Every charge goes to a subscription.
- You can have multiple subscriptions for different environments or departments:
  - `Dev-Subscription` — for development (maybe with spending limits)
  - `Prod-Subscription` — for production workloads
- Each subscription has its own set of resource groups and resources.
- **Azure equivalent of AWS Accounts**.

### Resource Groups

- A **logical container** that holds related Azure resources.
- Every Azure resource MUST be in exactly one resource group.
- Resources in a group can be in **different regions** (the group itself has a location, but it's
  just metadata — the resources inside can be anywhere).
- **Lifecycle management**: Delete the group = delete everything inside it.
- **Access control**: You can assign permissions at the resource group level (see [[Azure IAM]]).

**Best Practice**: One resource group per project or per environment:
- `webapp-dev-rg` — Development web app resources
- `webapp-prod-rg` — Production web app resources

### Regions

- Azure has **60+ regions** worldwide.
- A **region** is a set of data centers in a geographic area.
- Each region contains one or more **Availability Zones** (physically separate data centers).
- Choose your region based on:
  - **Latency** — Pick the region closest to your users
  - **Compliance** — Some data must stay in specific countries
  - **Service Availability** — Not all services are available in all regions
  - **Pricing** — Prices vary by region

**Common Regions**:
| Region Name | Location |
|-------------|----------|
| `eastus` | Virginia, USA |
| `westeurope` | Netherlands |
| `centralindia` | Pune, India |
| `southeastasia` | Singapore |
| `uksouth` | London, UK |

### Management Groups

- Used to organize **multiple subscriptions** into a hierarchy.
- Apply policies and access control across many subscriptions at once.
- Useful for large organizations.
- Most beginners won't need this right away, but good to know it exists.

---

## Installing the Azure CLI

The **Azure CLI** (`az`) is the command-line tool for managing Azure resources. You'll use it
constantly alongside the portal.

### Windows

**Option 1: Using winget (recommended)**
```powershell
winget install -e --id Microsoft.AzureCLI
```

**Option 2: Download MSI installer**
1. Go to: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-windows
2. Download the MSI installer.
3. Run the installer, accept defaults.
4. Restart your terminal.

### macOS

```bash
brew install azure-cli
```

If you don't have Homebrew, install it first from https://brew.sh

### Linux (Ubuntu/Debian)

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

### Verify Installation

After installing, open a **new** terminal and run:

```bash
az --version
```

You should see output like:

```
azure-cli    2.xx.x
core         2.xx.x
telemetry    1.x.x
...
```

> 💡 **Tip**: Keep Azure CLI updated. Run `az upgrade` periodically.

---

## Login and Configure Azure CLI

### Login to Azure

```bash
# Login — this opens your default browser for authentication
az login
```

After running this command:
1. Your browser opens automatically.
2. Sign in with your Microsoft account.
3. Close the browser tab when prompted.
4. Back in the terminal, you'll see your subscription information.

### List Your Subscriptions

```bash
# See all subscriptions linked to your account
az account list --output table
```

Output looks like:
```
Name                  CloudName    SubscriptionId                        TenantId                              State    IsDefault
--------------------  -----------  ------------------------------------  ------------------------------------  -------  -----------
Azure subscription 1  AzureCloud   xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  Enabled  True
```

### Set Active Subscription

If you have multiple subscriptions, set the one you want to work with:

```bash
az account set --subscription "your-subscription-id"
```

### Set Default Region

Set a default region so you don't have to specify `--location` every time:

```bash
az configure --defaults location=centralindia
```

### Useful Configuration

```bash
# Set default resource group (saves typing --resource-group every time)
az configure --defaults group=myResourceGroup

# Set default output format (table is easiest to read)
az configure --defaults output=table

# View current defaults
az configure --list-defaults
```

---

## Your First CLI Commands

Let's run some commands to get comfortable with the CLI.

### Check Who You Are

```bash
# Show details of the currently logged-in account
az account show
```

This shows your subscription name, ID, tenant ID, and other details.

### Create a Resource Group

This is always your **first step** for any project. Everything needs a resource group.

```bash
# Create a resource group in Central India
az group create --name myFirstResourceGroup --location centralindia
```

Output:
```json
{
  "id": "/subscriptions/.../resourceGroups/myFirstResourceGroup",
  "location": "centralindia",
  "name": "myFirstResourceGroup",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "type": "Microsoft.Resources/resourceGroups"
}
```

### List Resource Groups

```bash
# List all resource groups in your subscription
az group list --output table
```

### Explore Available Regions

```bash
# See all Azure regions available to you
az account list-locations --output table
```

### Check What Resources Exist

```bash
# List ALL resources in your subscription
az resource list --output table

# List resources in a specific resource group
az resource list --resource-group myFirstResourceGroup --output table
```

### Delete a Resource Group

⚠️ **This deletes EVERYTHING inside the group.** Use with caution!

```bash
# Delete a resource group and all its resources (--yes skips confirmation)
az group delete --name myFirstResourceGroup --yes
```

### Get Help

```bash
# General help
az --help

# Help for a specific command group
az vm --help

# Help for a specific command
az vm create --help

# Find commands (fuzzy search)
az find "create virtual machine"
```

> 💡 **Beginner Tip**: The `az find` command is incredibly useful when you don't know the exact
> command. Just describe what you want to do in plain English!

---

## Azure vs AWS Quick Comparison

If you're also learning AWS, here's a quick mapping of equivalent concepts:

| Concept | Azure | AWS |
|---------|-------|-----|
| **Web Console** | Azure Portal | AWS Console |
| **CLI Tool** | `az` CLI | `aws` CLI |
| **Identity Service** | Microsoft Entra ID | AWS IAM |
| **Resource Organization** | Resource Groups | Tags (no direct equivalent) |
| **Billing Unit** | Subscriptions | AWS Accounts |
| **Subscription Organization** | Management Groups | AWS Organizations |
| **Regions** | 60+ regions | 30+ regions |
| **CLI Login** | `az login` | `aws configure` |
| **Free Tier Credit** | $200 / 30 days | None (just free tier limits) |

> For a full service-by-service mapping, see [[AWS vs Azure Comparison]].

### Key Differences to Remember

1. **Resource Groups are mandatory in Azure** — In AWS, you just create resources wherever. In
   Azure, every resource must belong to a resource group. This is actually a better organizational
   model.

2. **Entra ID is separate from RBAC** — In AWS, IAM handles both identity and permissions. In
   Azure, Entra ID handles identity and Azure RBAC handles permissions. See [[Azure IAM]].

3. **Azure has better hybrid cloud** — If you need to connect on-premises infrastructure to the
   cloud, Azure generally has better tooling (Azure Arc, Azure Stack).

4. **Azure Portal is arguably easier** — The Azure Portal is considered more user-friendly than
   the AWS Console, especially for beginners.

---

## What's Next?

Now that you have an Azure account and know the basics, here's your learning path:

1. ✅ **Azure Getting Started** — You are here!
2. 🔐 [[Azure IAM]] — Learn about users, permissions, and security
3. 💻 [[Azure Compute]] — Create VMs, deploy web apps, run containers
4. 📦 [[Azure Storage]] — Store files, blobs, and data
5. 🌐 [[Azure Networking]] — Virtual networks, load balancers, DNS
6. 🗄️ [[Azure Databases]] — SQL, Cosmos DB, and other databases
7. 💰 [[Azure Billing]] — Understand and optimize your costs

---

## Quick Reference Card

```bash
# ===== ESSENTIAL AZURE CLI COMMANDS =====

# Login
az login

# Show current account
az account show

# List subscriptions
az account list --output table

# Set subscription
az account set --subscription "<subscription-id>"

# Create resource group
az group create --name <name> --location <region>

# List resource groups
az group list --output table

# Delete resource group (and everything in it!)
az group delete --name <name> --yes

# List all resources
az resource list --output table

# Get help
az <command> --help

# Find a command
az find "<description>"

# Check CLI version
az --version

# Upgrade CLI
az upgrade
```

---

**Tags**: #azure #cloud #getting-started #beginner #cli
**Related**: [[Cloud Fundamentals]] | [[Azure IAM]] | [[Azure Billing]] | [[AWS Getting Started]]
