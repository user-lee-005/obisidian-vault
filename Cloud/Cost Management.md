# Cost Management

> **Goal**: Understand how cloud billing works, avoid surprise bills, and optimize spending so you get the most value for every dollar.

---

## How Cloud Billing Works

The cloud uses a **pay-as-you-go** model — like a utility bill (electricity, water).

```
Traditional IT:                          Cloud:
┌──────────────────────────┐            ┌──────────────────────────┐
│ Buy servers upfront       │            │ No upfront cost           │
│ Pay whether you use       │            │ Pay only for what you     │
│   them or not             │            │   actually use            │
│ Capacity planning needed  │            │ Scale up/down anytime     │
│ $100,000+ capital expense │            │ $0.10/hour operating cost │
└──────────────────────────┘            └──────────────────────────┘
```

### Key Billing Concepts

- **Pay-as-you-go** — Use a VM for 3 hours? Pay for 3 hours. Turn it off? Stop paying.
- **Per-second / Per-hour billing** — Most compute resources bill per second or per hour.
- **Metered usage** — Storage, data transfer, API calls — all metered like electricity.
- **No termination fees** — Stop using a service anytime (except reserved instances).

> **Warning**: The cloud makes it VERY easy to spend money. A forgotten VM running 24/7 can cost hundreds per month. Monitoring your costs is just as important as monitoring your applications.

---

## Pricing Models

### 1. On-Demand Instances

**Pay per hour or per second**, with no commitment. Stop anytime.

```
Example: AWS t3.medium (2 vCPU, 4 GB RAM)
  → $0.0416 per hour
  → ~$30 per month (running 24/7)
  → ~$360 per year
```

**When to use**: Unpredictable workloads, short-term projects, development/testing.

---

### 2. Reserved Instances / Savings Plans

**Commit to using resources for 1-3 years** in exchange for a big discount.

```
On-Demand:  $0.0416/hour  ← Full price
1-Year RI:  $0.026/hour   ← 37% savings
3-Year RI:  $0.017/hour   ← 59% savings
```

**Options**:
- **All Upfront** — Pay everything now → biggest discount
- **Partial Upfront** — Pay some now, rest monthly → medium discount
- **No Upfront** — Pay monthly → smallest discount (but still cheaper than on-demand)

**When to use**: Steady, predictable workloads (databases, always-on web servers).

---

### 3. Spot / Preemptible Instances

Use the cloud provider's **spare/unused capacity** at a massive discount (60-90% off). But the provider can **take it back with 2 minutes notice**.

```
On-Demand:   $0.0416/hour
Spot Price:  $0.0125/hour  ← 70% cheaper!

BUT: AWS can reclaim your instance at any time ⚠️
```

**When to use**: Fault-tolerant workloads that can handle interruptions:
- Batch processing (data analysis, video encoding)
- CI/CD build jobs
- Testing and development
- Distributed computing (tasks can restart from a checkpoint)

**When NOT to use**: Databases, single-instance web servers, anything that can't tolerate sudden shutdown.

---

### 4. Free Tier

Both AWS and Azure offer generous free tiers for new accounts:

**AWS Free Tier** (12 months):
- 750 hours/month of t2.micro or t3.micro EC2
- 5 GB of S3 storage
- 750 hours/month of RDS (database)
- 1 million Lambda requests/month
- And much more...

**Azure Free Tier** (12 months):
- 750 hours/month of B1S VM
- 5 GB of Blob Storage (LRS)
- 250 GB of SQL Database
- 1 million Azure Functions requests/month
- And much more...

> **Important**: Some free tier services are **Always Free** (never expire), while others are **12 Months Free** (expire after a year). Read the fine print!

---

### Pricing Models Comparison

| Model | Discount | Commitment | Risk | Best For |
|-------|----------|------------|------|----------|
| **On-Demand** | 0% (full price) | None | None | Unpredictable workloads |
| **Reserved (1-yr)** | 30-40% | 1 year | Locked in | Steady workloads |
| **Reserved (3-yr)** | 50-70% | 3 years | Very locked in | Long-term, predictable |
| **Spot/Preemptible** | 60-90% | None | Instance can be terminated | Fault-tolerant batch jobs |
| **Free Tier** | 100% | New accounts | Limited resources | Learning, small projects |

---

## What Costs Money?

Understanding **what** you're being charged for is the first step to controlling costs.

### 1. Compute

Running virtual machines, containers, or functions.

| Service | Billing Unit | Example Cost |
|---------|-------------|--------------|
| EC2 / Azure VMs | Per hour/second the VM runs | $0.04/hr for a small VM |
| Containers (ECS, AKS) | Per vCPU + memory per second | $0.04/vCPU/hr |
| Serverless (Lambda, Functions) | Per request + execution time | $0.20 per 1M requests |

> **Tip**: Compute is often the #1 cost. Turn off dev/test VMs when not in use!

### 2. Storage

Storing data — charged per GB per month, plus per-operation fees.

| Service | Billing Unit | Example Cost |
|---------|-------------|--------------|
| S3 / Blob Storage | Per GB/month + per request | $0.023/GB/month (standard) |
| EBS / Managed Disks | Per GB/month (provisioned) | $0.10/GB/month (SSD) |
| Snapshots/Backups | Per GB/month stored | $0.05/GB/month |

> **Tip**: Old snapshots and unused volumes are silent cost killers. Clean them up regularly.

### 3. Data Transfer (The Hidden Cost!)

```
               ┌──────────────┐
  Data IN      │              │     Data OUT
  ────────→    │   AWS/Azure  │    ────────→
  FREE! ✅     │   Cloud      │    COSTS $$ ❌
               │              │
               └──────────────┘

Data between regions: COSTS $$
Data between AZs:     Costs $ (small)
Data within same AZ:  FREE ✅
```

| Transfer Type | Cost |
|--------------|------|
| **Data IN** (to cloud) | Usually **FREE** |
| **Data OUT** (to internet) | ~$0.09/GB (AWS) |
| **Between regions** | ~$0.02/GB |
| **Between AZs** | ~$0.01/GB |

> **This catches beginners off guard!** If your app serves 1 TB of data per month to users, that's ~$90/month just in data transfer.

### 4. Other Services That Cost Money

| Service | How It's Billed |
|---------|----------------|
| **Databases (RDS, Azure SQL)** | Instance size + storage + I/O |
| **Load Balancers** | Per hour + per GB processed |
| **DNS (Route 53, Azure DNS)** | Per hosted zone + per million queries |
| **CDN (CloudFront, Azure CDN)** | Per GB delivered + per request |
| **NAT Gateway** | Per hour + per GB processed |
| **Elastic IP (unused)** | Charged if allocated but NOT attached! |

---

## Cost Optimization Strategies

### 1. Right-Sizing

**Don't over-provision.** If your app uses 1 GB of RAM, don't run it on a 32 GB instance.

```
❌ Over-provisioned:  m5.2xlarge (8 vCPU, 32 GB) → $0.384/hr → $280/month
   Actual usage:      1 vCPU, 2 GB RAM

✅ Right-sized:       t3.small (2 vCPU, 2 GB) → $0.0208/hr → $15/month
   Savings:           $265/month (94% less!)
```

**How to right-size:**
- Use monitoring tools to check actual CPU/memory usage over 2 weeks
- AWS: Use **Compute Optimizer** for recommendations
- Azure: Use **Azure Advisor** for recommendations
- Start small, scale up if needed

### 2. Auto-Scaling

Don't pay for capacity you don't need. Scale up during peak hours, scale down during off-hours.

```
Traffic Pattern:
           ┌─────┐
           │     │
       ┌───┘     └───┐
   ────┘              └────────

Without Auto-scaling: ████████████████████████  (pay for peak 24/7)
With Auto-scaling:    ██  ████████  ████  ██    (pay only for what you use)
```

### 3. Use Reserved Instances for Steady Workloads

If a database server runs 24/7/365, don't pay on-demand prices.

```
Database server (m5.large), running 24/7:
  On-Demand:        $70/month
  1-Year Reserved:  $44/month  (save $312/year)
  3-Year Reserved:  $29/month  (save $492/year)
```

### 4. Use Spot Instances for Fault-Tolerant Work

```
CI/CD build jobs:
  On-Demand:  $0.192/hr × 1000 builds = $192
  Spot:       $0.058/hr × 1000 builds = $58  (save 70%!)
```

### 5. Delete Unused Resources

Common resource waste:

| Forgotten Resource | Monthly Cost |
|-------------------|--------------|
| Idle EC2 instance (m5.large) | ~$70 |
| Orphaned EBS volume (500 GB) | ~$50 |
| Old snapshots (1 TB total) | ~$50 |
| Unused Elastic IP | ~$3.60 |
| Idle load balancer | ~$16 |
| **Total waste** | **~$190/month** |

> **Schedule regular cleanup!** Many teams waste 20-30% of their cloud spend on unused resources.

### 6. Use Serverless for Sporadic Workloads

If a function runs a few times per hour, serverless is dramatically cheaper than a VM:

```
Function that runs 1000 times/day, 200ms each:
  EC2 (t3.micro, 24/7):     $7.60/month
  Lambda (pay-per-use):      $0.03/month   ← 250x cheaper!
```

### 7. Choose Cheaper Regions

Cloud prices vary by region:

```
AWS EC2 m5.large:
  US East (Virginia):     $0.096/hr
  Asia Pacific (Tokyo):   $0.124/hr   (29% more expensive)
  South America (Brazil): $0.168/hr   (75% more expensive!)
```

> **Tip**: Choose the cheapest region that still meets your latency and compliance requirements.

### 8. Use Storage Tiers

Store data in the cheapest tier appropriate for access patterns:

| Tier | Access | Cost (per GB/month) | Use For |
|------|--------|---------------------|---------|
| **Hot / Standard** | Frequent access | $0.023 | Active application data |
| **Cool / Infrequent** | Occasional access | $0.0125 | Logs older than 30 days |
| **Archive / Glacier** | Rare access | $0.004 | Compliance data, old backups |

```
1 TB of logs:
  All in Standard:   $23.50/month
  Tiered strategy:   $6.80/month   (70% savings)
    - Last 30 days: Standard (100 GB)
    - 30-90 days: Cool (200 GB)
    - 90+ days: Archive (700 GB)
```

---

## Budgets and Alerts

### Setting Budgets

Always set a budget so you get warned before costs spiral out of control.

```
Budget Example:
  Monthly budget:  $500
  Alert 1:         At 50% ($250) — "You're on track"
  Alert 2:         At 80% ($400) — "Watch your spending"
  Alert 3:         At 100% ($500) — "Budget reached! Investigate!"
  Alert 4:         At 120% ($600) — "Over budget! Take action!"
```

### AWS Budget Setup

```
AWS Console → Billing → Budgets → Create Budget
  → Monthly cost budget
  → Amount: $500
  → Alert: 80% threshold → Email: team@company.com
```

### Azure Budget Setup

```
Azure Portal → Cost Management → Budgets → Add
  → Monthly budget
  → Amount: $500
  → Alert: 80% threshold → Email: team@company.com
```

> **Pro tip**: Set up budgets on DAY ONE of using the cloud. Don't wait until you get a surprise bill.

---

## Cost Management Tools

### AWS Cost Tools

| Tool | Purpose | Learn More |
|------|---------|------------|
| **AWS Cost Explorer** | Visualize and analyze spending over time | [[AWS Billing]] |
| **AWS Budgets** | Set budgets and alerts | [[AWS Billing]] |
| **AWS Trusted Advisor** | Recommendations for cost savings | [[AWS Billing]] |
| **AWS Compute Optimizer** | Right-sizing recommendations for EC2 | [[AWS Billing]] |
| **AWS Cost Anomaly Detection** | AI-powered unusual spend detection | [[AWS Billing]] |

### Azure Cost Tools

| Tool | Purpose | Learn More |
|------|---------|------------|
| **Azure Cost Management** | Visualize and analyze spending | [[Azure Billing]] |
| **Azure Advisor** | Cost optimization recommendations | [[Azure Billing]] |
| **Azure Budgets** | Set budgets and alerts | [[Azure Billing]] |
| **Azure Pricing Calculator** | Estimate costs before deploying | [[Azure Billing]] |

---

## Tagging for Cost Tracking

**Tags** are key-value labels you attach to cloud resources. They're essential for understanding WHERE your money goes.

### Example Tags

| Tag Key | Tag Value | Purpose |
|---------|-----------|---------|
| `Environment` | `production` / `staging` / `dev` | Track cost per environment |
| `Team` | `backend` / `frontend` / `data` | Track cost per team |
| `Project` | `order-service` / `search-api` | Track cost per project |
| `Owner` | `john@company.com` | Know who to contact |
| `CostCenter` | `CC-1234` | Map to accounting codes |

### Why Tags Matter

```
Monthly bill: $5,000

Without tags:
  "We spent $5,000... on something. Not sure what."

With tags:
  Production:     $3,000 (60%)
  Staging:        $1,200 (24%)
  Development:    $800   (16%)  ← Why is dev so expensive?

  Backend team:   $2,500
  Frontend team:  $1,500
  Data team:      $1,000

  Order service:  $2,000  ← Our most expensive service
  Search API:     $1,500
  User service:   $1,000
  Other:          $500
```

### Tagging Best Practices

- Define a **tagging policy** before you start (which tags are required?)
- **Automate tagging** — use IaC templates that include tags by default
- **Enforce tags** — use policies to prevent creating untagged resources
- Review untagged resources monthly

---

## Common Cost Pitfalls

### 1. Leaving Dev/Test Resources Running Over Weekends

```
5 developers × 1 VM each × weekend (48 hours):
  → 5 × $0.096/hr × 48 hrs = $23 wasted per weekend
  → $23 × 52 weeks = $1,196 wasted per year

Fix: Auto-shutdown dev VMs at 7 PM, auto-start at 8 AM.
     Don't run dev resources on weekends.
```

### 2. Not Setting Budgets

```
Month 1: Spent $200 — "That seems fine"
Month 2: Spent $350 — Didn't notice
Month 3: Spent $800 — Still didn't notice
Month 4: Spent $2,500 — "WHAT HAPPENED?"

Answer: A forgotten load test VM was running with 10x the needed capacity.

Fix: Set a $300 budget with alerts on Day 1.
```

### 3. Over-Provisioning "Just In Case"

```
Developer: "I'll use a large instance just in case we get traffic."
Reality:   CPU usage never exceeds 5%
Cost:      $140/month instead of $15/month

Fix: Start small. Use auto-scaling for spikes.
     Monitor actual usage before sizing up.
```

### 4. Ignoring Data Transfer Costs

```
Architecture: App in US-East, Database in EU-West
Monthly data transfer between regions: 500 GB
Cost: 500 × $0.02 = $10/month

Better: Put app and database in the same region → $0/month
```

### 5. Not Using Auto-Scaling

```
Your app gets heavy traffic Mon-Fri, 9 AM - 5 PM.
Without auto-scaling: 10 servers running 24/7 = $720/month
With auto-scaling:    10 servers peak, 2 servers off-peak = $340/month
Savings:              $380/month (53%!)
```

### 6. Accumulating Old Snapshots

```
Daily snapshots × 365 days × 100 GB each = 36.5 TB
Cost: 36.5 TB × $0.05/GB = $1,825/month

Fix: Set a retention policy — keep only last 30 days of snapshots.
     New cost: 3 TB × $0.05/GB = $150/month
     Savings: $1,675/month
```

---

## Cost Optimization Checklist

Use this checklist monthly to keep costs under control:

```
□ Review the monthly bill — any surprises?
□ Check for unused resources (idle VMs, unattached volumes)
□ Verify auto-scaling is configured for variable workloads
□ Review right-sizing recommendations (Compute Optimizer / Azure Advisor)
□ Check reserved instance coverage for steady workloads
□ Verify dev/test environments auto-shutdown after hours
□ Review data transfer costs — any cross-region traffic?
□ Check storage tiers — any data that should be archived?
□ Verify all resources are properly tagged
□ Review budget alerts — are thresholds still appropriate?
```

---

## Key Takeaways

1. **Cloud billing is pay-as-you-go** — you pay for what you use, nothing more
2. **Choose the right pricing model**: On-Demand for flexibility, Reserved for steady workloads, Spot for fault-tolerant batch jobs
3. **Data transfer OUT costs money** — data IN is usually free
4. **Right-size everything** — don't over-provision
5. **Set budgets and alerts on Day 1** — never get surprised by a bill
6. **Tag all resources** — know where your money goes
7. **Regular cleanup** — delete unused resources monthly
8. **Use storage tiers** — don't pay hot-tier prices for cold data

---

## Related Notes

- [[AWS Billing]] — AWS-specific cost tools (Cost Explorer, Budgets, Trusted Advisor)
- [[Azure Billing]] — Azure-specific cost tools (Cost Management, Advisor)
- [[Cloud Fundamentals]] — Core cloud concepts including regions (which affect pricing)
- [[Monitoring and Observability]] — Monitor costs alongside application health
- [[Cloud Deployment Strategies]] — Deployment choices impact costs (Blue-Green = 2x infra)
