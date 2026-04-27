# Azure Billing & Cost Management

> **The #1 mistake beginners make in the cloud is forgetting about costs.**
> This note covers everything you need to know to avoid surprise bills,
> set up alerts, and optimize your Azure spending.

---

## Table of Contents

- [[#Azure Free Account]]
- [[#Azure Pricing Calculator]]
- [[#Setting Up Budget Alerts]]
- [[#Azure Cost Management + Billing]]
- [[#Common Azure Costs]]
- [[#Cost Optimization]]
- [[#Tagging for Cost Tracking]]
- [[#Azure Spending Limit]]
- [[#Common Cost Pitfalls]]
- [[#Azure Advisor]]

---

## Azure Free Account

When you sign up for Azure, you get a generous free tier:

### What You Get

| Benefit                    | Details                                               |
|----------------------------|-------------------------------------------------------|
| **$200 credit**            | Use on ANY Azure service for the first 30 days        |
| **12-month free services** | Popular services free for 12 months (see below)       |
| **Always-free services**   | Some services are free forever (see below)            |

### 12-Month Free Services (highlights)

| Service                    | Free Allowance                              |
|----------------------------|---------------------------------------------|
| **Linux B1S VM**           | 750 hours/month (≈ 1 VM running 24/7)       |
| **Windows B1S VM**         | 750 hours/month                             |
| **Blob Storage**           | 5 GB (LRS, Hot tier)                        |
| **Azure SQL Database**     | 250 GB (S0 tier)                            |
| **Cosmos DB**              | 25 GB storage + 1000 RU/s (shared throughput)|
| **Managed Disks**          | 2 × 64 GB (P6 SSD)                          |
| **Bandwidth**              | 15 GB outbound data transfer                |

### Always-Free Services

| Service                    | Free Allowance                              |
|----------------------------|---------------------------------------------|
| **Azure Functions**        | 1 million executions/month                  |
| **Azure DevOps**           | 5 users, unlimited private repos            |
| **Azure Active Directory** | Basic tier                                  |
| **App Service**            | 10 web apps (F1 Free tier — shared compute) |
| **Azure Advisor**          | Completely free                              |
| **Azure Monitor**          | Basic metrics and alerts                    |
| **Azure DevTest Labs**     | Free (you pay only for resources used)      |

### Free Account Timeline

```
Day 0 ────────── Day 30 ────────── Month 12 ────────── Beyond
  │                  │                  │                   │
  │  $200 credit     │  Credit expires  │  Free services    │  Pay-as-you-go
  │  + 12-mo free    │  12-mo free      │  expire           │  for everything
  │  + always free   │  continues       │  Always-free      │  Always-free
  │                  │  Always-free     │  continues        │  continues
  │                  │  continues       │                   │
```

### Important Rules

- **Credit expires after 30 days** — unused credit is lost.
- **After 30 days**: You must **upgrade to pay-as-you-go** to keep your resources.
  If you don't upgrade, all resources are **deleted after 90 days**.
- **12-month free services** have usage limits. Exceeding them incurs charges.
- **You need a credit card** to sign up (for identity verification), but you
  won't be charged unless you explicitly upgrade.

> 🎯 **Beginner tip**: Set up [[#Setting Up Budget Alerts|budget alerts]] on
> Day 1. Even with the free tier, it's good practice.

---

## Azure Pricing Calculator

> **URL**: https://azure.microsoft.com/pricing/calculator/

The Azure Pricing Calculator lets you **estimate costs before deploying** anything.

### How to Use It

1. Go to the Pricing Calculator website.
2. **Add services** — Click the services you plan to use (VM, SQL, Storage, etc.).
3. **Configure each service** — Choose region, tier, size, expected usage.
4. **See the monthly estimate** — The calculator shows estimated monthly cost.
5. **Export** — Download as Excel or share a link for approval.

### Example Estimate: Simple Web App

| Service                         | Configuration            | Monthly Cost (approx) |
|---------------------------------|--------------------------|----------------------|
| App Service Plan                | B1 (Basic, 1 core)      | ~$13                 |
| Azure SQL Database              | Basic (5 DTU)            | ~$5                  |
| Blob Storage                    | 10 GB, LRS, Hot          | ~$0.20               |
| Azure Cache for Redis           | Basic C0 (250 MB)        | ~$16                 |
| **Total**                       |                          | **~$34/month**       |

> Always check the calculator before deploying. A Premium SQL Database can cost
> $1,000+/month vs $5/month for Basic. Know what you're paying for!

---

## Setting Up Budget Alerts

> ⚠️ **This is the MOST IMPORTANT section in this note.**
> Set up budget alerts BEFORE you deploy anything.

### Via Azure Portal (Recommended for Beginners)

1. Go to **Cost Management + Billing** in the Azure Portal.
2. Click **Budgets** → **+ Add**.
3. Configure:
   - **Name**: `MonthlyBudget`
   - **Amount**: Your monthly limit (e.g., $50 for learning)
   - **Reset period**: Monthly
4. Set **Alert Conditions**:
   - Alert at **50%** of budget (you've spent $25)
   - Alert at **80%** of budget (you've spent $40)
   - Alert at **100%** of budget (you've hit $50)
5. Add **Alert Recipients**: Your email address.
6. (Optional) Add an **Action Group** for automated responses.

### Alert Thresholds — Recommended Setup

| Threshold | % of Budget | Purpose                                 |
|-----------|-------------|-----------------------------------------|
| Warning   | 50%         | Heads up — halfway through your budget  |
| Alert     | 80%         | Action needed — review and optimize     |
| Critical  | 100%        | Stop! Investigate immediately            |
| Forecast  | 110%        | Predicted to exceed — take action now    |

### Action Groups — Automate Responses

Action Groups define **what happens** when a budget alert fires:

| Action Type     | Description                                         |
|-----------------|-----------------------------------------------------|
| **Email**       | Send email notification to individuals or groups     |
| **SMS**         | Send text message alerts                            |
| **Webhook**     | Call an HTTP endpoint (trigger automation)            |
| **Azure Function** | Run a function (e.g., auto-deallocate VMs)       |
| **Logic App**   | Trigger a workflow (e.g., Slack notification)        |

```bash
# Create an Action Group via CLI
az monitor action-group create \
  --resource-group myResourceGroup \
  --name BudgetAlerts \
  --short-name Budget \
  --action email admin you@email.com
```

> **Pro tip**: Set up a budget alert that triggers an Azure Function to
> **automatically deallocate VMs** when you hit 90% of your budget.
> This prevents runaway costs while you're learning.

---

## Azure Cost Management + Billing

**Azure Cost Management** is the built-in tool for analyzing and controlling costs.

### Key Features

| Feature              | Description                                          |
|----------------------|------------------------------------------------------|
| **Cost Analysis**    | Interactive charts — filter by resource group, service, tag, time |
| **Budgets**          | Set spending limits with alerts                      |
| **Advisor**          | Cost optimization recommendations                    |
| **Exports**          | Schedule CSV exports for accounting/reporting         |
| **Anomaly Detection**| Alerts when spending patterns change unexpectedly    |

### Cost Analysis — How to Use

1. Go to **Cost Management** → **Cost Analysis** in the Azure Portal.
2. **Filter** by:
   - Resource group (e.g., "show me costs for my dev environment")
   - Service name (e.g., "how much am I spending on VMs?")
   - Tag (e.g., "show costs for Project=MyApp")
   - Time range (last 7 days, this month, last 3 months)
3. **Group by**: Resource, Service, Tag, Location
4. **View as**: Chart (area, bar, table) or download CSV

### Scheduled Exports

```bash
# Set up automatic cost data export to a storage account
# Portal: Cost Management → Exports → Add
# Choose: Daily/Weekly/Monthly export
# Format: CSV
# Destination: Azure Storage account
```

> This is useful for feeding cost data into your own dashboards or accounting tools.

---

## Common Azure Costs

Understanding what costs money helps you avoid surprises.

### Compute Costs

| Service              | Pricing Model                               | Key Detail                     |
|----------------------|---------------------------------------------|--------------------------------|
| **VMs**              | Per-hour (while running or stopped*)         | *Stopped ≠ Deallocated!        |
| **App Service**      | Per-hour (plan runs 24/7)                    | Even with zero traffic         |
| **Azure Functions**  | Per execution + compute time                 | Consumption plan: 1M free/month|
| **AKS**              | Free control plane, pay for node VMs         | Worker nodes are just VMs      |

> ⚠️ **VM Stopped vs Deallocated**: A "Stopped" VM still holds its compute
> allocation — **you still pay for compute**. A "Deallocated" VM releases
> compute — **no compute charges**. Always **deallocate**, don't just stop.

```bash
# STOP (still costs money for compute!)
az vm stop --resource-group myResourceGroup --name myVM

# DEALLOCATE (no compute charges)
az vm deallocate --resource-group myResourceGroup --name myVM

# Check VM power state
az vm get-instance-view \
  --resource-group myResourceGroup \
  --name myVM \
  --query instanceView.statuses[1].displayStatus
```

### Storage Costs

| Component          | Cost Driver                                    |
|--------------------|------------------------------------------------|
| **Data stored**    | Per GB per month (varies by tier: Hot/Cool/Archive) |
| **Transactions**   | Per 10,000 operations (read, write, list)      |
| **Data transfer**  | Outbound transfer costs money; inbound is free |
| **Snapshots**      | Per GB of snapshot data                         |

### Database Costs

| Service             | Cost Driver                                    |
|---------------------|------------------------------------------------|
| **Azure SQL**       | DTUs or vCores + storage                        |
| **Cosmos DB**       | RU/s + storage                                  |
| **MySQL/PostgreSQL**| vCores + storage + backup storage               |
| **Redis**           | Cache size × tier                               |

### Networking Costs

| Component              | Free?                                     |
|------------------------|-------------------------------------------|
| **Inbound data**       | ✅ Free                                   |
| **Outbound data**      | ❌ First 5 GB/month free, then per GB     |
| **VNet**               | ✅ Free                                   |
| **VNet Peering**       | ❌ Per GB transferred (both directions)   |
| **Public IP (unused)** | ❌ Costs money even when not attached!     |
| **Load Balancer rules**| ❌ Per rule + data processed               |
| **VPN Gateway**        | ❌ Per hour + data transfer                |
| **ExpressRoute**       | ❌ Monthly circuit fee + data transfer     |

---

## Cost Optimization

### 1. Right-Size VMs

Don't pay for resources you don't need.

```bash
# Check Azure Advisor recommendations
az advisor recommendation list --category cost --output table
```

- Use Azure Advisor to find underutilized VMs.
- Downsize VMs that consistently use < 40% CPU.
- Use **B-series (burstable)** VMs for variable workloads.

### 2. Deallocate VMs When Not in Use

```bash
# Deallocate a VM (stops billing for compute)
az vm deallocate --resource-group myResourceGroup --name myVM

# Start it again when needed
az vm start --resource-group myResourceGroup --name myVM
```

### 3. Schedule Auto-Shutdown for Dev/Test VMs

```bash
# Auto-shutdown at 7 PM every day
az vm auto-shutdown \
  --resource-group myResourceGroup \
  --name myVM \
  --time 1900 \
  --email you@email.com
```

> This prevents the classic mistake: forgetting to turn off your dev VM over
> the weekend and wasting 48+ hours of compute charges.

### 4. Azure Reserved Instances (RIs)

| Commitment | Savings vs Pay-as-you-go |
|------------|--------------------------|
| 1 year     | Up to 40%                |
| 3 years    | Up to 72%                |

- Best for **predictable, steady-state** workloads.
- You commit to a specific VM size and region.
- Payment: All upfront, monthly, or no upfront.

### 5. Azure Spot VMs

- **Up to 90% cheaper** than regular VMs.
- Azure can **evict** your VM at any time when it needs the capacity.
- Best for: Batch processing, CI/CD builds, dev/test, fault-tolerant workloads.
- **Not for**: Production web servers, databases, anything that must stay up.

```bash
# Create a Spot VM
az vm create \
  --resource-group myResourceGroup \
  --name mySpotVM \
  --image Ubuntu2204 \
  --size Standard_D2s_v3 \
  --priority Spot \
  --max-price 0.05 \
  --eviction-policy Deallocate
```

### 6. Use Consumption-Based Pricing

- **Azure Functions (Consumption Plan)**: Pay only when code runs.
  1 million executions free per month.
- **Cosmos DB Serverless**: Pay per RU consumed. No provisioned throughput.
- **Logic Apps (Consumption)**: Pay per action executed.

### 7. Delete Unused Resources

```bash
# Find all resources in a resource group
az resource list --resource-group myResourceGroup --output table

# Delete an entire resource group (everything inside it)
az group delete --name myResourceGroup --yes --no-wait
```

> **Best practice**: Use a dedicated resource group for each project/experiment.
> When done, delete the entire group — nothing is left behind.

### 8. Use Storage Tiers

| Tier        | Cost       | Access Frequency        | Use Case                 |
|-------------|------------|-------------------------|--------------------------|
| **Hot**     | Higher     | Frequent access         | Active data              |
| **Cool**    | Lower      | Infrequent (30+ days)   | Backups, older data      |
| **Cold**    | Even lower | Rare (90+ days)         | Compliance data          |
| **Archive** | Cheapest   | Rarely (180+ days)      | Long-term retention      |

---

## Tagging for Cost Tracking

**Tags** are key-value pairs you attach to Azure resources. They're essential
for tracking costs by project, team, or environment.

### Hands-on: Tag Your Resources

```bash
# Tag a resource group
az group update \
  --name myResourceGroup \
  --tags Environment=Dev Project=MyApp CostCenter=Engineering
```

```bash
# Tag a specific resource
az resource tag \
  --tags Environment=Dev Project=MyApp \
  --ids /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM
```

```bash
# List all resources with a specific tag
az resource list --tag Environment=Dev --output table
```

### Recommended Tag Strategy

| Tag Key          | Example Values           | Purpose                        |
|------------------|--------------------------|--------------------------------|
| `Environment`    | Dev, Staging, Prod       | Separate cost by environment   |
| `Project`        | MyApp, DataPipeline      | Track project-level spending   |
| `CostCenter`     | Engineering, Marketing   | Chargeback to departments      |
| `Owner`          | alice@company.com        | Know who to contact            |
| `AutoShutdown`   | True, False              | Mark VMs for auto-shutdown     |
| `Expiry`         | 2024-12-31               | Flag temporary resources       |

### Using Tags in Cost Analysis

1. Go to **Cost Management** → **Cost Analysis**.
2. **Group by** → **Tag** → Select your tag key (e.g., `Project`).
3. Now you can see cost breakdown per project.

> **Enforce tagging** with Azure Policy — require certain tags on all resources.
> This prevents untagged resources from being created.

---

## Azure Spending Limit

### How It Works

| Account Type       | Spending Limit                                     |
|--------------------|----------------------------------------------------|
| **Free Account**   | ✅ $200 limit (credit). Can't exceed it.           |
| **Pay-as-you-go**  | ❌ No limit by default. You MUST set budgets!      |
| **Enterprise**     | Configurable by the organization.                   |

> When you upgrade from Free to Pay-as-you-go, the spending limit is **removed**.
> This is why budget alerts are critical — there's no automatic safety net.

---

## Common Cost Pitfalls

> ⚠️ Learn from other people's mistakes. These are the most common Azure bill shocks.

### 1. Stopped vs Deallocated VMs
**Problem**: You "stop" a VM in the OS (or via `az vm stop`), but it's NOT
deallocated. You're still paying for the compute reservation.
**Fix**: Always use `az vm deallocate` or stop from the Azure Portal (which deallocates).

### 2. No Budget Alerts
**Problem**: You don't know you're overspending until the bill arrives.
**Fix**: Set up budget alerts on Day 1. See [[#Setting Up Budget Alerts]].

### 3. Premium When Basic Suffices
**Problem**: Deploying Premium-tier databases, Premium VMs, or Enterprise
Redis when Basic or Standard would work fine for dev/test.
**Fix**: Start with the cheapest tier. Scale up only when needed.

### 4. Unused Public IPs
**Problem**: You delete a VM but forget to delete its Public IP.
Unused Standard Public IPs cost ~$3.65/month each.
**Fix**: Always clean up associated resources when deleting VMs.

```bash
# List all public IPs and check if they're assigned
az network public-ip list --resource-group myResourceGroup --output table
```

### 5. Not Using Resource Groups for Cleanup
**Problem**: Resources scattered across random resource groups. Hard to find
and delete everything from a project.
**Fix**: Create one resource group per project/experiment. Delete the group when done.

### 6. Over-Provisioned Databases
**Problem**: Running a Premium-tier SQL Database (thousands of DTUs) when your
app only uses 5% of the capacity.
**Fix**: Check DTU usage in Azure Portal metrics. Downsize accordingly.

### 7. Leaving Dev/Test Resources Running 24/7
**Problem**: Dev VMs running nights and weekends when nobody's using them.
**Fix**: Auto-shutdown + auto-start schedules. Or use Azure DevTest Labs.

### 8. Ignoring Data Transfer Costs
**Problem**: Moving large amounts of data between regions or out to the internet.
**Fix**: Keep resources in the same region. Use CDN for content delivery.

---

## Azure Advisor

**Azure Advisor** is a **free, built-in service** that analyzes your resources and
provides recommendations across five categories:

| Category           | Examples                                                |
|--------------------|---------------------------------------------------------|
| **Cost**           | Resize underutilized VMs, buy reservations              |
| **Security**       | Enable MFA, update NSG rules                            |
| **Reliability**    | Enable backups, add redundancy                           |
| **Operational Excellence** | Set up alerts, use best practices             |
| **Performance**    | Optimize database queries, use caching                   |

### Hands-on: Check Advisor Recommendations

```bash
# List all recommendations
az advisor recommendation list --output table
```

```bash
# Filter by category
az advisor recommendation list --category cost --output table
az advisor recommendation list --category security --output table
```

```bash
# Get detailed recommendation
az advisor recommendation list --category cost --query "[].{Name:shortDescription.problem, Impact:impact, Category:category}" --output table
```

> **Check Advisor weekly.** It's free and almost always finds something useful.
> Even experienced Azure users find savings through Advisor.

---

## Monthly Cost Control Checklist

Use this checklist every month:

- [ ] Review **Cost Analysis** — any unexpected spikes?
- [ ] Check **Azure Advisor** cost recommendations
- [ ] Verify all dev/test VMs have **auto-shutdown** enabled
- [ ] Look for **unused resources** (public IPs, disks, NICs)
- [ ] Review **resource group** list — delete any abandoned projects
- [ ] Check if any resources can be **downsized** (VMs, databases)
- [ ] Verify **budget alerts** are active and thresholds are appropriate
- [ ] Review **tags** — ensure all resources are tagged for cost tracking

---

## Summary

| Concept                   | Key Takeaway                                          |
|---------------------------|-------------------------------------------------------|
| **Free Account**          | $200 credit (30 days) + 12 months free + always-free  |
| **Pricing Calculator**    | Estimate costs BEFORE deploying                       |
| **Budget Alerts**         | Set up on Day 1 — non-negotiable                      |
| **Cost Management**       | Built-in tool for analysis, budgets, exports          |
| **Cost Optimization**     | Right-size, deallocate, reserve, spot, delete unused   |
| **Tagging**               | Tag everything for cost tracking by project/team      |
| **Advisor**               | Free recommendations — check weekly                   |
| **Biggest Pitfall**       | Stopped ≠ Deallocated. Always deallocate VMs.         |

> 🎯 **Rule of thumb**: If you're learning Azure, set a **$50/month budget**
> with alerts at 50%, 80%, and 100%. Use the free tier wherever possible.
> Delete resource groups when you're done experimenting.

---

## Related Notes

- [[Cost Management]] — General cloud cost management strategies
- [[Azure Getting Started]] — Setting up your Azure account
- [[Azure Compute]] — Understanding VM costs and tiers
- [[Azure Databases]] — Database pricing models (DTU, RU/s, vCore)
- [[Azure Networking]] — Network egress costs
- [[AWS Billing]] — Compare with AWS billing and cost tools
