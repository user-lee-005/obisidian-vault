# Azure Compute

> **Run your code in the cloud — from full VMs to serverless functions.**
> Prerequisites: [[Azure Getting Started]] (you need a resource group first!)

---

## Table of Contents

- [[#Azure Compute Overview]]
- [[#Azure Virtual Machines]]
- [[#VM Sizes and Series]]
- [[#Hands-On — Create a Linux VM]]
- [[#Managing VMs with CLI]]
- [[#Network Security Groups (NSGs)]]
- [[#Availability and Scaling]]
- [[#Azure App Service]]
- [[#Hands-On — Deploy a Web App]]
- [[#Deployment Slots]]
- [[#Azure Functions — Serverless]]
- [[#Hands-On — Create an Azure Function]]
- [[#Azure Container Instances (ACI)]]
- [[#Azure Kubernetes Service (AKS)]]
- [[#Compute Comparison Table]]

---

## Azure Compute Overview

Azure offers multiple ways to run your code, ranging from full control (you manage everything)
to fully managed (Azure handles everything). Here's the spectrum:

```
Most Control                                                    Least Control
(You manage more)                                          (Azure manages more)

┌──────────────┐  ┌─────────────────┐  ┌───────┐  ┌─────────────┐  ┌───────────┐
│   Virtual    │  │    Container    │  │  AKS  │  │ App Service │  │ Functions │
│  Machines    │  │   Instances     │  │       │  │             │  │           │
│              │  │                 │  │       │  │             │  │           │
│  IaaS        │  │  Containers    │  │ K8s   │  │  PaaS       │  │ Serverless│
│  Full OS     │  │  No orchestr.  │  │ Mgd   │  │  Web apps   │  │ Per exec. │
└──────────────┘  └─────────────────┘  └───────┘  └─────────────┘  └───────────┘
```

### When to Use What?

| Service | Best For | Think of It As... |
|---------|----------|-------------------|
| **Virtual Machines** | Full OS control, legacy apps, custom software | Renting a computer in the cloud |
| **Container Instances** | Quick container runs, batch jobs, simple containers | `docker run` in the cloud |
| **AKS** | Microservices, complex container orchestration | Managed Kubernetes cluster |
| **App Service** | Web apps, APIs, mobile backends | Managed web hosting (like Heroku) |
| **Functions** | Event-driven, lightweight tasks, APIs | Code that runs only when triggered |

---

## Azure Virtual Machines

### What is a VM?

An Azure Virtual Machine (VM) is a **full computer running in the cloud**. You get:
- An operating system (Windows or Linux)
- CPU, RAM, and storage
- A network interface with an IP address
- Full admin/root access

It's the most flexible compute option — you can install anything, configure everything. But
it's also the most work to manage (you're responsible for OS updates, security patches, etc.).

> 💡 **Analogy**: A VM is like renting a fully furnished apartment. You have full control of the
> space, but you're responsible for cleaning, maintenance, and utilities.

### What Gets Created with a VM?

When you create a VM in Azure, several related resources are automatically created:

```
Virtual Machine
    ├── OS Disk (Managed Disk)
    ├── Network Interface (NIC)
    │   ├── Public IP Address (optional)
    │   └── Network Security Group (NSG)
    └── Virtual Network + Subnet (if not existing)
```

This is important to know because **deleting the VM doesn't delete all these resources**.
You may need to clean them up separately, or use a resource group to delete everything at once.

---

## VM Sizes and Series

Azure VMs come in different **sizes** (configurations of CPU, RAM, storage, etc.) organized
into **series** based on workload type.

### Most Common Series

| Series | Type | Use Case | AWS Equivalent |
|--------|------|----------|----------------|
| **B-series** | Burstable | Dev/test, low traffic websites, small databases | t3, t4g |
| **D-series** | General Purpose | Most production workloads, web servers, app servers | m5, m6i |
| **F-series** | Compute Optimized | Batch processing, gaming, high-performance computing | c5, c6i |
| **E-series** | Memory Optimized | Large databases, in-memory caching, data analytics | r5, r6i |
| **L-series** | Storage Optimized | Big data, data warehousing, large transactional databases | i3, i4i |
| **N-series** | GPU | Machine learning, graphics rendering, video encoding | p3, g4dn |

### Understanding VM Size Names

Azure VM sizes follow a naming convention:

```
Standard_B1s
│        ││
│        │└─ Size modifier (s = supports premium storage)
│        └── Size number (1 = smallest in series)
└── Series (B = Burstable)
```

More examples:
- `Standard_B1s` — Burstable, 1 vCPU, 1 GB RAM (cheapest, great for learning!)
- `Standard_B2s` — Burstable, 2 vCPU, 4 GB RAM
- `Standard_D2s_v5` — General purpose, 2 vCPU, 8 GB RAM, version 5
- `Standard_E4s_v5` — Memory optimized, 4 vCPU, 32 GB RAM, version 5

### List Available VM Sizes via CLI

```bash
# List all VM sizes available in a region
az vm list-sizes --location centralindia --output table

# Filter for B-series (burstable — cheapest)
az vm list-sizes --location centralindia --output table --query "[?starts_with(name, 'Standard_B')]"
```

### Free Tier VM

The Azure free tier includes **750 hours/month** of B1S VM (both Linux and Windows). That's
enough to run one B1S VM 24/7 for a month. Perfect for learning!

---

## Hands-On — Create a Linux VM

Let's create your first virtual machine! We'll use the Azure CLI.

### Step 1: Make Sure You Have a Resource Group

```bash
# Create a resource group if you don't have one
az group create --name myResourceGroup --location centralindia
```

### Step 2: Create the VM

```bash
# Create a Linux VM with SSH key authentication
az vm create \
  --resource-group myResourceGroup \
  --name myFirstVM \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --size Standard_B1s \
  --public-ip-sku Standard
```

Let's break down each parameter:

| Parameter | Meaning |
|-----------|---------|
| `--resource-group` | Which resource group to put the VM in |
| `--name` | Name for your VM |
| `--image` | The OS image. `Ubuntu2204` = Ubuntu 22.04 LTS |
| `--admin-username` | The username for SSH login (don't use "admin" or "root") |
| `--generate-ssh-keys` | Auto-generate SSH keys for secure login |
| `--size` | The VM size (B1s = free tier eligible!) |
| `--public-ip-sku` | Assigns a public IP so you can access it from the internet |

**Output** (this takes 1-3 minutes):
```json
{
  "publicIpAddress": "20.xxx.xxx.xxx",
  "resourceGroup": "myResourceGroup",
  "powerState": "VM running",
  "fqdns": ""
}
```

### Step 3: Connect to Your VM

```bash
# SSH into the VM using the public IP from the output
ssh azureuser@<public-ip-address>
```

Replace `<public-ip-address>` with the actual IP from the output above.

You're now inside a Linux machine running in the cloud! 🎉

```bash
# Once connected, try some commands:
hostname                    # Shows the VM name
uname -a                   # Shows Linux version
free -h                     # Shows available memory
df -h                       # Shows disk space
cat /proc/cpuinfo | head    # Shows CPU info
```

Type `exit` to disconnect from the VM.

### Step 4: Get VM Details

```bash
# Show VM details
az vm show \
  --resource-group myResourceGroup \
  --name myFirstVM \
  --output table

# Get just the public IP address
az vm show \
  --resource-group myResourceGroup \
  --name myFirstVM \
  --show-details \
  --query publicIps \
  --output tsv
```

### Common Images

```bash
# List popular VM images
az vm image list --output table

# Search for specific images
az vm image list --offer Ubuntu --all --output table | head -20
az vm image list --offer WindowsServer --all --output table | head -20
```

| Image Alias | OS |
|-------------|-----|
| `Ubuntu2204` | Ubuntu 22.04 LTS |
| `Ubuntu2404` | Ubuntu 24.04 LTS |
| `Debian11` | Debian 11 |
| `CentOS85` | CentOS 8.5 |
| `Win2022Datacenter` | Windows Server 2022 |
| `Win2019Datacenter` | Windows Server 2019 |

---

## Managing VMs with CLI

### Power Management

Understanding VM states is important for cost management:

```
Running        → You're paying for compute + storage + network
Stopped        → You're STILL paying for compute reservation + storage
Deallocated    → You're paying for storage only (compute freed)
Deleted        → Check if disks/IPs still exist!
```

```bash
# Stop a VM (WARNING: still incurs compute charges!)
az vm stop \
  --resource-group myResourceGroup \
  --name myFirstVM

# Deallocate a VM (stops ALL compute charges — only storage cost remains)
az vm deallocate \
  --resource-group myResourceGroup \
  --name myFirstVM

# Start a VM
az vm start \
  --resource-group myResourceGroup \
  --name myFirstVM

# Restart a VM
az vm restart \
  --resource-group myResourceGroup \
  --name myFirstVM
```

> ⚠️ **Critical Cost Tip**: Always use `az vm deallocate` instead of `az vm stop` when you're
> done working. `stop` keeps the compute reservation (you still pay!). `deallocate` releases it.

### Resizing a VM

```bash
# List available sizes for resizing (depends on current hardware cluster)
az vm list-vm-resize-options \
  --resource-group myResourceGroup \
  --name myFirstVM \
  --output table

# Resize a VM (VM must be deallocated for some size changes)
az vm resize \
  --resource-group myResourceGroup \
  --name myFirstVM \
  --size Standard_B2s
```

### Deleting a VM

```bash
# Delete the VM (this does NOT delete the disk, NIC, public IP, etc.!)
az vm delete \
  --resource-group myResourceGroup \
  --name myFirstVM \
  --yes

# To clean up EVERYTHING, delete the entire resource group
az group delete --name myResourceGroup --yes
```

### Running Commands on a VM (Without SSH)

```bash
# Run a command on a VM remotely
az vm run-command invoke \
  --resource-group myResourceGroup \
  --name myFirstVM \
  --command-id RunShellScript \
  --scripts "apt update && apt upgrade -y"
```

### Install a Web Server on Your VM

```bash
# SSH into the VM first, then:
sudo apt update
sudo apt install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx

# Verify it's running
curl localhost
```

Now you need to open port 80 so the internet can reach it (see NSGs below).

---

## Network Security Groups (NSGs)

### What is an NSG?

A **Network Security Group** is a firewall for your Azure resources. It contains rules that
allow or deny network traffic.

By default, when you create a VM:
- **SSH (port 22)** is open (for Linux VMs)
- **RDP (port 3389)** is open (for Windows VMs)
- **Everything else is blocked**

### Open a Port

```bash
# Open port 80 (HTTP) to the internet
az vm open-port \
  --resource-group myResourceGroup \
  --name myFirstVM \
  --port 80

# Open port 443 (HTTPS)
az vm open-port \
  --resource-group myResourceGroup \
  --name myFirstVM \
  --port 443 \
  --priority 900
```

The `--priority` must be unique per rule (lower number = higher priority). Default is 900.

After opening port 80 and installing nginx, browse to `http://<your-vm-public-ip>` and you'll
see the nginx welcome page!

### List NSG Rules

```bash
# List all NSG rules for a VM's network
az network nsg rule list \
  --resource-group myResourceGroup \
  --nsg-name myFirstVMNSG \
  --output table
```

### Create Custom NSG Rules

```bash
# Allow traffic on port 8080 from a specific IP
az network nsg rule create \
  --resource-group myResourceGroup \
  --nsg-name myFirstVMNSG \
  --name AllowPort8080 \
  --priority 1000 \
  --destination-port-ranges 8080 \
  --source-address-prefixes 203.0.113.0/24 \
  --access Allow \
  --protocol Tcp

# Deny all inbound traffic (use high priority number = low priority)
az network nsg rule create \
  --resource-group myResourceGroup \
  --nsg-name myFirstVMNSG \
  --name DenyAllInbound \
  --priority 4096 \
  --destination-port-ranges '*' \
  --access Deny \
  --protocol '*'
```

> 💡 **Security Tip**: Never open ports unnecessarily. Only open what your application needs.
> For SSH, consider using Azure Bastion instead of exposing port 22 to the internet.

---

## Availability and Scaling

### Availability Sets

- Protect against hardware failures within a single data center.
- VMs in an availability set are spread across **Fault Domains** (separate racks) and
  **Update Domains** (separate maintenance windows).
- Gives you a **99.95% SLA**.

### Availability Zones

- Protect against entire data center failures.
- VMs are spread across **physically separate data centers** within a region.
- Gives you a **99.99% SLA**.
- Not all regions support Availability Zones.

```bash
# Create a VM in a specific availability zone
az vm create \
  --resource-group myResourceGroup \
  --name myZonalVM \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --size Standard_B1s \
  --zone 1
```

### Virtual Machine Scale Sets (VMSS)

- A group of **identical VMs** that can **auto-scale** based on demand.
- Similar to AWS Auto Scaling Groups.
- Integrates with Azure Load Balancer for distributing traffic.

```bash
# Create a scale set with 2 initial instances
az vmss create \
  --resource-group myResourceGroup \
  --name myScaleSet \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --instance-count 2 \
  --vm-sku Standard_B1s \
  --upgrade-policy-mode automatic

# Enable auto-scaling
az monitor autoscale create \
  --resource-group myResourceGroup \
  --resource myScaleSet \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name autoscale-config \
  --min-count 1 \
  --max-count 5 \
  --count 2

# Add a scale-out rule (add VM when CPU > 70%)
az monitor autoscale rule create \
  --resource-group myResourceGroup \
  --autoscale-name autoscale-config \
  --condition "Percentage CPU > 70 avg 5m" \
  --scale out 1
```

---

## Azure App Service

### What is App Service?

**Azure App Service** is a **PaaS (Platform as a Service)** for hosting web applications, APIs,
and mobile backends. You just deploy your code — Azure handles the infrastructure.

> 💡 **Analogy**: If a VM is renting an apartment (you manage everything), App Service is like
> staying at a hotel (the hotel manages the building, you just use your room).

### Why Use App Service?

- **No server management** — No OS updates, no patching, no infrastructure.
- **Multiple languages** — .NET, Java, Node.js, Python, PHP, Ruby, Go.
- **Built-in CI/CD** — Deploy from GitHub, Azure DevOps, Docker Hub, or local git.
- **Auto-scaling** — Scale out automatically based on traffic.
- **SSL/Custom domains** — Built-in HTTPS with free managed certificates.
- **Deployment slots** — Staging environments with zero-downtime swaps.

### App Service Plans

Your App Service runs on an **App Service Plan**, which defines the compute resources:

| Tier | Plan | vCPU | RAM | Cost | Features |
|------|------|------|-----|------|----------|
| **Free** | F1 | Shared | 1 GB | $0 | 60 min/day CPU, no custom domain SSL |
| **Basic** | B1 | 1 | 1.75 GB | ~$13/mo | Custom domains, SSL, manual scale |
| **Standard** | S1 | 1 | 1.75 GB | ~$70/mo | Auto-scale, staging slots, backups |
| **Premium** | P1v3 | 1 | 8 GB | ~$100/mo | Better performance, more slots, VNet |

> 💡 **For learning**: Start with **F1 (Free)** tier. It's enough for testing and experimenting.

---

## Hands-On — Deploy a Web App

### Step 1: Create an App Service Plan

```bash
# Create a Linux App Service Plan on the Free tier
az appservice plan create \
  --name myAppServicePlan \
  --resource-group myResourceGroup \
  --sku F1 \
  --is-linux
```

### Step 2: Create a Web App

```bash
# Create a Python web app
# NOTE: The name must be globally unique (it becomes <name>.azurewebsites.net)
az webapp create \
  --name myuniquewebapp123 \
  --resource-group myResourceGroup \
  --plan myAppServicePlan \
  --runtime "PYTHON:3.12"
```

Your app is now live at: `https://myuniquewebapp123.azurewebsites.net`
(It shows a default placeholder page until you deploy your code.)

### Step 3: Deploy Your Code

**Option A: Deploy from GitHub**
```bash
az webapp deployment source config \
  --name myuniquewebapp123 \
  --resource-group myResourceGroup \
  --repo-url https://github.com/your-username/your-repo \
  --branch main \
  --manual-integration
```

**Option B: Deploy from a ZIP file**
```bash
# Package your app as a ZIP, then deploy
az webapp deployment source config-zip \
  --resource-group myResourceGroup \
  --name myuniquewebapp123 \
  --src ./app.zip
```

**Option C: Deploy from local Git**
```bash
# Configure local git deployment
az webapp deployment source config-local-git \
  --name myuniquewebapp123 \
  --resource-group myResourceGroup

# The output gives you a git remote URL. Add it and push:
git remote add azure <deployment-url-from-output>
git push azure main
```

### Step 4: View and Manage

```bash
# Open the app in your browser
az webapp browse --name myuniquewebapp123 --resource-group myResourceGroup

# View application logs (live streaming)
az webapp log tail --name myuniquewebapp123 --resource-group myResourceGroup

# View app settings (environment variables)
az webapp config appsettings list --name myuniquewebapp123 --resource-group myResourceGroup

# Set an environment variable
az webapp config appsettings set \
  --name myuniquewebapp123 \
  --resource-group myResourceGroup \
  --settings DATABASE_URL="your-connection-string"

# Restart the app
az webapp restart --name myuniquewebapp123 --resource-group myResourceGroup
```

### Available Runtimes

```bash
# List all available runtimes
az webapp list-runtimes
```

Common runtimes: `PYTHON:3.12`, `NODE:20-lts`, `JAVA:17-java17`, `DOTNETCORE:8.0`,
`PHP:8.3`, `RUBY:3.2`

---

## Deployment Slots

Deployment slots are a killer feature of App Service (available in Standard tier and above).

### What are Deployment Slots?

- A **slot** is a live instance of your app with its own URL.
- The default slot is called `production`.
- You can create additional slots like `staging`, `testing`, `canary`.
- Each slot has its own configuration and can have different code deployed.
- You can **swap** slots to perform zero-downtime deployments.

### How It Works

```
1. Deploy new code to "staging" slot
2. Test it at staging URL
3. Swap staging ↔ production
4. Users instantly see the new version
5. If something's wrong, swap back!
```

```bash
# Create a staging slot
az webapp deployment slot create \
  --name myuniquewebapp123 \
  --resource-group myResourceGroup \
  --slot staging

# Deploy to the staging slot
az webapp deployment source config-zip \
  --resource-group myResourceGroup \
  --name myuniquewebapp123 \
  --slot staging \
  --src ./new-version.zip

# Test at: https://myuniquewebapp123-staging.azurewebsites.net

# Swap staging and production
az webapp deployment slot swap \
  --name myuniquewebapp123 \
  --resource-group myResourceGroup \
  --slot staging \
  --target-slot production
```

---

## Azure Functions — Serverless

### What are Azure Functions?

**Azure Functions** is a **serverless compute service**. You write small pieces of code
(functions) that run in response to events. You don't manage any servers.

- **Pay only when your code runs** (Consumption plan).
- Scales automatically — from zero to thousands of executions.
- Supports: C#, JavaScript, Python, Java, PowerShell, TypeScript.

> 💡 **Analogy**: If App Service is a hotel room (always there, you pay monthly), Azure Functions
> is like an Uber (shows up when you need it, you pay per ride).

### Triggers and Bindings

**Triggers** define WHAT starts the function:

| Trigger | Description |
|---------|------------|
| **HTTP** | Runs when it receives an HTTP request (great for APIs) |
| **Timer** | Runs on a schedule (like a cron job) |
| **Blob Storage** | Runs when a file is added/modified in blob storage |
| **Queue** | Runs when a message arrives in a queue |
| **Cosmos DB** | Runs when a document changes in Cosmos DB |
| **Event Hub** | Runs when an event is received |

**Bindings** make it easy to connect to other services (input/output data) without writing
boilerplate code.

### Hosting Plans

| Plan | Pricing | Scale | Max Execution | Best For |
|------|---------|-------|---------------|----------|
| **Consumption** | Pay per execution | 0 to 200 instances | 5 min (default) | Sporadic workloads |
| **Premium** | Pre-warmed instances | 1 to 100+ | 30 min+ | Low latency needs |
| **Dedicated** | App Service Plan | Manual or auto-scale | Unlimited | Already have App Service |

---

## Hands-On — Create an Azure Function

### Local Development Setup

```bash
# Install Azure Functions Core Tools (requires Node.js)
npm install -g azure-functions-core-tools@4 --unsafe-perm true
```

### Create a Function Project

```bash
# Create a new function project (Python example)
func init MyFunctionProject --python
cd MyFunctionProject

# Create a new HTTP-triggered function
func new --name HttpTrigger --template "HTTP trigger" --authlevel anonymous
```

### Run Locally

```bash
# Start the local function runtime
func start
```

Output:
```
Functions:
    HttpTrigger: [GET,POST] http://localhost:7071/api/HttpTrigger
```

Test it: Open `http://localhost:7071/api/HttpTrigger?name=Azure` in your browser.

### Deploy to Azure

First, create the required Azure resources:

```bash
# 1. Create a storage account (required for Functions)
az storage account create \
  --name myfuncstorageacct \
  --resource-group myResourceGroup \
  --location centralindia \
  --sku Standard_LRS

# 2. Create the Function App in Azure
az functionapp create \
  --resource-group myResourceGroup \
  --consumption-plan-location centralindia \
  --runtime python \
  --runtime-version 3.12 \
  --functions-version 4 \
  --name myfuncapp123 \
  --storage-account myfuncstorageacct \
  --os-type Linux
```

```bash
# 3. Deploy your function code
func azure functionapp publish myfuncapp123
```

Your function is now live at: `https://myfuncapp123.azurewebsites.net/api/HttpTrigger`

### Managing Functions

```bash
# List function apps
az functionapp list --resource-group myResourceGroup --output table

# View function app settings
az functionapp config appsettings list --name myfuncapp123 --resource-group myResourceGroup

# Set an app setting
az functionapp config appsettings set \
  --name myfuncapp123 \
  --resource-group myResourceGroup \
  --settings "MY_SETTING=my_value"

# View function logs
az functionapp log show --name myfuncapp123 --resource-group myResourceGroup

# Delete a function app
az functionapp delete --name myfuncapp123 --resource-group myResourceGroup
```

---

## Azure Container Instances (ACI)

### What is ACI?

**Azure Container Instances** is the simplest way to run a Docker container in Azure.
No cluster management, no orchestration — just run a container.

> 💡 **Analogy**: ACI is like `docker run` in the cloud. Quick, simple, no fuss.

### When to Use ACI?

- Quick demos and prototypes
- Batch processing jobs
- Build agents for CI/CD
- Running sidecar containers
- When you don't need Kubernetes but want containers

### Hands-On: Run a Container

```bash
# Run a simple web container
az container create \
  --resource-group myResourceGroup \
  --name mycontainer \
  --image mcr.microsoft.com/azuredocs/aci-helloworld \
  --dns-name-label mycontainerdemo123 \
  --ports 80 \
  --cpu 1 \
  --memory 1.5
```

Parameters:
- `--image` — Docker image to run (from Docker Hub, ACR, or Microsoft Container Registry)
- `--dns-name-label` — Creates a public URL (must be unique in the region)
- `--ports` — Ports to expose
- `--cpu` — Number of CPU cores (default: 1)
- `--memory` — GB of memory (default: 1.5)

```bash
# Check the status
az container show \
  --resource-group myResourceGroup \
  --name mycontainer \
  --query instanceView.state \
  --output tsv

# Get the full URL (FQDN)
az container show \
  --resource-group myResourceGroup \
  --name mycontainer \
  --query ipAddress.fqdn \
  --output tsv

# View container logs
az container logs \
  --resource-group myResourceGroup \
  --name mycontainer

# Attach to container output (live streaming)
az container attach \
  --resource-group myResourceGroup \
  --name mycontainer

# Execute a command inside the container
az container exec \
  --resource-group myResourceGroup \
  --name mycontainer \
  --exec-command "/bin/sh"

# Delete the container
az container delete \
  --resource-group myResourceGroup \
  --name mycontainer \
  --yes
```

### Run a Custom Docker Image

```bash
# Run your own image from Docker Hub
az container create \
  --resource-group myResourceGroup \
  --name mycustomapp \
  --image yourdockerhubuser/yourimage:latest \
  --dns-name-label mycustomapp123 \
  --ports 8080 \
  --environment-variables DATABASE_URL=your-db-url API_KEY=your-key
```

---

## Azure Kubernetes Service (AKS)

### What is AKS?

**Azure Kubernetes Service** is a **managed Kubernetes cluster**. Azure manages the control
plane (master nodes) — you just manage your worker nodes and deploy your workloads.

> 💡 **When to Use AKS**: When you have multiple microservices that need orchestration, scaling,
> service discovery, and rolling updates. If you just need to run one or two containers, use ACI.

### Hands-On: Create an AKS Cluster

```bash
# Create an AKS cluster with 1 node
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --node-count 1 \
  --node-vm-size Standard_B2s \
  --generate-ssh-keys \
  --enable-managed-identity
```

This takes 5-10 minutes. Azure is provisioning the Kubernetes control plane and worker nodes.

```bash
# Install kubectl (if not already installed)
az aks install-cli

# Get credentials to connect to your cluster
az aks get-credentials \
  --resource-group myResourceGroup \
  --name myAKSCluster

# Verify connection
kubectl get nodes
```

Output:
```
NAME                                STATUS   ROLES    AGE   VERSION
aks-nodepool1-12345678-vmss000000   Ready    <none>   5m    v1.28.x
```

### Basic kubectl Commands

```bash
# View all pods across all namespaces
kubectl get pods --all-namespaces

# Deploy a sample application
kubectl create deployment hello-aks --image=mcr.microsoft.com/azuredocs/aks-helloworld:v1

# Expose it via a load balancer
kubectl expose deployment hello-aks --type=LoadBalancer --port=80

# Check the external IP (may take a minute)
kubectl get service hello-aks --watch

# Scale the deployment
kubectl scale deployment hello-aks --replicas=3

# View pods
kubectl get pods

# Delete the deployment
kubectl delete deployment hello-aks
kubectl delete service hello-aks
```

### Manage AKS Cluster

```bash
# Scale the node pool
az aks scale \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --node-count 3

# Upgrade the cluster
az aks get-upgrades --resource-group myResourceGroup --name myAKSCluster --output table
az aks upgrade --resource-group myResourceGroup --name myAKSCluster --kubernetes-version <version>

# Delete the cluster
az aks delete --resource-group myResourceGroup --name myAKSCluster --yes
```

---

## Compute Comparison Table

| Feature | **VMs** | **App Service** | **Functions** | **ACI** | **AKS** |
|---------|---------|----------------|---------------|---------|---------|
| **Type** | IaaS | PaaS | Serverless | Containers | Managed K8s |
| **Control** | Full OS | App-level | Code-level | Container | K8s cluster |
| **Languages** | Any | .NET, Java, Python, Node, PHP, Ruby, Go | C#, JS, Python, Java, PowerShell | Any (Docker) | Any (Docker) |
| **Scaling** | Manual or VMSS | Built-in auto-scale | Automatic (0→∞) | Manual | HPA, KEDA |
| **Min Cost** | ~$3.80/mo (B1ls) | Free (F1) | Free (1M exec) | ~$0.0025/sec | Free control plane |
| **Cold Start** | None (always on) | None (always on) | Yes (Consumption) | ~5 sec | None |
| **Best For** | Legacy apps, custom OS | Web apps, APIs | Event-driven tasks | Simple containers | Microservices |
| **Complexity** | Medium | Low | Low | Low | High |
| **OS Patches** | You manage | Azure manages | Azure manages | Azure manages | You manage nodes |
| **AWS Equiv.** | EC2 | Elastic Beanstalk | Lambda | Fargate (basic) | EKS |

### Decision Flowchart

```
Need to run code in Azure?
│
├── Need full OS control? ──────────────────────── → Virtual Machine
│
├── Building a web app/API?
│   ├── Simple web app? ────────────────────────── → App Service
│   └── Microservices with complex orchestration? ─ → AKS
│
├── Running a container?
│   ├── Just one or two containers? ───────────── → ACI
│   └── Many containers with orchestration? ────── → AKS
│
└── Event-driven / small task? ────────────────── → Azure Functions
```

---

## Cost-Saving Tips

1. **Use B1S VMs** for learning — eligible for free tier.
2. **Always deallocate VMs** when not in use (`az vm deallocate`, not `az vm stop`).
3. **Use Reserved Instances** for production VMs (save up to 72%).
4. **Use Spot VMs** for fault-tolerant workloads (save up to 90%).
5. **Use Azure Functions Consumption plan** for sporadic workloads — pay nothing when idle.
6. **Use App Service Free (F1) tier** for learning and testing.
7. **Delete resource groups** when done experimenting — catches all leftover resources.

---

## What's Next?

- 📦 [[Azure Storage]] — Store your application data
- 🌐 [[Azure Networking]] — Connect your compute resources
- 🗄️ [[Azure Databases]] — Add a database to your applications
- 🚀 [[Azure Deployment]] — CI/CD pipelines for automated deployment
- 🔐 [[Azure IAM]] — Control who can access your compute resources

---

## Quick Reference Card

```bash
# ===== VIRTUAL MACHINES =====
az vm create -g myRG -n myVM --image Ubuntu2204 --admin-username azureuser --generate-ssh-keys --size Standard_B1s
az vm show -g myRG -n myVM --show-details --query publicIps -o tsv
az vm deallocate -g myRG -n myVM          # Stop paying for compute
az vm start -g myRG -n myVM
az vm delete -g myRG -n myVM --yes
az vm open-port -g myRG -n myVM --port 80

# ===== APP SERVICE =====
az appservice plan create -n myPlan -g myRG --sku F1 --is-linux
az webapp create -n myapp -g myRG --plan myPlan --runtime "PYTHON:3.12"
az webapp deployment source config-zip -g myRG -n myapp --src app.zip
az webapp browse -n myapp -g myRG
az webapp log tail -n myapp -g myRG

# ===== FUNCTIONS =====
func init MyProject --python && cd MyProject
func new --name HttpTrigger --template "HTTP trigger"
func start                                # Run locally
func azure functionapp publish myfuncapp  # Deploy to Azure

# ===== CONTAINER INSTANCES =====
az container create -g myRG -n mycontainer --image nginx --dns-name-label myapp --ports 80
az container logs -g myRG -n mycontainer
az container delete -g myRG -n mycontainer --yes

# ===== AKS =====
az aks create -g myRG -n myCluster --node-count 1 --generate-ssh-keys
az aks get-credentials -g myRG -n myCluster
kubectl get nodes
```

---

**Tags**: #azure #compute #vm #app-service #functions #containers #aks #beginner
**Related**: [[Cloud Fundamentals]] | [[Azure Getting Started]] | [[Azure Networking]] | [[Azure Deployment]] | [[AWS Compute]]
