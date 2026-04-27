# AWS Billing & Cost Management

> **The #1 mistake beginners make with AWS is getting an unexpected bill.**
>
> Before you deploy ANYTHING, read this entire note.
> AWS charges by usage — if you leave something running, you pay for it.
> The good news: AWS has excellent free tier options and cost management tools.
> The bad news: It's YOUR responsibility to monitor costs.

---

## Table of Contents

- [[#AWS Free Tier]]
- [[#Setting Up Billing Alerts (DO THIS FIRST)]]
- [[#AWS Budgets]]
- [[#AWS Cost Explorer]]
- [[#AWS Pricing Calculator]]
- [[#Common Costs Breakdown]]
- [[#Cost Optimization Tips]]
- [[#AWS Trusted Advisor]]
- [[#Tagging Strategy]]
- [[#Avoiding Bill Shock]]

---

## AWS Free Tier

AWS offers **three types** of free usage:

### 1. Always Free (Forever)

These services are **always free** up to certain limits, no matter how long you've had your account:

| Service | Free Amount | Notes |
|---|---|---|
| **Lambda** | 1 million requests/month | + 400,000 GB-seconds of compute |
| **DynamoDB** | 25 GB storage | + 25 WCU + 25 RCU (provisioned) |
| **CloudWatch** | 10 custom metrics | + 10 alarms + 1 million API requests |
| **SNS** | 1 million publishes | + 100,000 HTTP deliveries |
| **SQS** | 1 million requests | Standard and FIFO queues |
| **CloudFormation** | Unlimited | You pay for the resources it creates |
| **IAM** | Unlimited | Users, roles, policies — always free |
| **VPC** | Free | VPCs, subnets, route tables, SGs |

### 2. 12 Months Free (New Accounts Only)

These are **free for the first 12 months** after you create your AWS account:

| Service | Free Amount | After Free Tier |
|---|---|---|
| **EC2** | 750 hrs/month t2.micro | ~$8.50/month (t3.micro) |
| **S3** | 5 GB storage | ~$0.023/GB-month |
| **RDS** | 750 hrs/month db.t2.micro | ~$12/month (db.t3.micro) |
| **EBS** | 30 GB storage (gp2/gp3) | ~$0.08/GB-month |
| **CloudFront** | 1 TB data transfer out | ~$0.085/GB |
| **Elastic Load Balancing** | 750 hours ALB | ~$16/month + data |
| **ElastiCache** | 750 hours cache.t2.micro | ~$12/month |

> [!warning] 12-Month Clock
> The 12-month clock starts when you **create your account**, not when you first use a service.
> If you created your account 6 months ago and never used EC2, you only have 6 months of free EC2 left.

### 3. Short-Term Trials

Some services offer **short free trials** (e.g., Amazon SageMaker, Amazon Redshift, Amazon Inspector).
These vary — check the [AWS Free Tier page](https://aws.amazon.com/free/) for the latest.

### How to Check Your Free Tier Usage

```bash
# Check free tier usage via CLI
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics "UsageQuantity" "BlendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE

# Or in the console:
# Billing → Free Tier → See usage vs limits for each service
```

> [!tip] Bookmark This Page
> **Console**: Go to **Billing** → **Free Tier** to see a dashboard of your usage vs. free tier limits.
> Check it weekly when you're starting out.

---

## Setting Up Billing Alerts (DO THIS FIRST)

> [!danger] CRITICAL
> **Set up billing alerts BEFORE you do ANYTHING else in AWS.**
> This is not optional. This is the single most important step as a beginner.

### Why?

Without alerts, you won't know you're being charged until you see the bill at the end of the month. By then, you might owe $50, $100, or even more.

### Option 1: AWS Budgets (Recommended)

```bash
# Create a monthly budget of $10 with email alerts at 50%, 80%, and 100%
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{
    "BudgetName": "MonthlyBudget",
    "BudgetLimit": {
      "Amount": "10",
      "Unit": "USD"
    },
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST"
  }' \
  --notifications-with-subscribers '[
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 50
      },
      "Subscribers": [
        {
          "SubscriptionType": "EMAIL",
          "Address": "you@email.com"
        }
      ]
    },
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 80
      },
      "Subscribers": [
        {
          "SubscriptionType": "EMAIL",
          "Address": "you@email.com"
        }
      ]
    },
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 100
      },
      "Subscribers": [
        {
          "SubscriptionType": "EMAIL",
          "Address": "you@email.com"
        }
      ]
    }
  ]'
```

### Option 2: Console Steps

1. Go to **Billing** → **Budgets** → **Create a budget**
2. Choose **Cost budget — Recommended**
3. Budget name: `MonthlyBudget`
4. Budget amount: `$10` (for learning)
5. Email recipients: your email
6. Alert thresholds: **50%**, **80%**, **100%**
7. Click **Create budget**

### Option 3: CloudWatch Billing Alarm (Legacy but still useful)

```bash
# First, enable billing alerts in account settings
# Console: Billing → Billing Preferences → Receive Billing Alerts ✓

# Then create a CloudWatch alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "BillingAlarm-10USD" \
  --alarm-description "Alarm when charges exceed $10" \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --statistic Maximum \
  --period 21600 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=Currency,Value=USD \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:BillingAlerts \
  --region us-east-1
```

> [!note] Billing data is ONLY in us-east-1
> Regardless of which region you use, billing metrics are only available in the **US East (N. Virginia)** region.

---

## AWS Budgets

### What Are AWS Budgets?

AWS Budgets let you set **spending limits** and get **alerts** when you approach or exceed them.

### Types of Budgets

| Type | What It Tracks |
|---|---|
| **Cost Budget** | Total dollar spend |
| **Usage Budget** | Usage of specific services (e.g., EC2 hours) |
| **Savings Plans Budget** | Utilization of Savings Plans |
| **Reservation Budget** | Utilization of Reserved Instances |

### Budget Best Practices

1. **Set a cost budget** for your total monthly spend ($10-$25 for learning)
2. **Set alerts at multiple thresholds**: 50%, 80%, 100%, and 150% (forecasted)
3. **Add a forecasted alert** — warns you before you actually hit the limit
4. Create **per-service budgets** for expensive services (EC2, RDS)
5. Review and adjust monthly

### List and Check Budgets

```bash
# List all budgets
aws budgets describe-budgets --account-id 123456789012

# Get budget details
aws budgets describe-budget \
  --account-id 123456789012 \
  --budget-name MonthlyBudget
```

---

## AWS Cost Explorer

### What is Cost Explorer?

Cost Explorer is a **visual tool** to analyze your AWS spending. It shows charts and tables of your costs over time.

### Enabling Cost Explorer

1. Go to **Billing** → **Cost Explorer**
2. Click **Enable Cost Explorer**
3. Wait **24 hours** — it needs time to collect and process data

### What You Can Do

- **View costs by service** — See which service costs the most
- **View costs by region** — Find out if you accidentally launched resources in expensive regions
- **View costs by tag** — See costs per project or environment (requires tagging)
- **Filter and group** — Drill down into specific time periods, services, or accounts
- **Forecast** — Predict next month's bill based on current usage trends

### CLI Examples

```bash
# Get cost breakdown by service for last month
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics "BlendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE

# Get daily costs for current month
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity DAILY \
  --metrics "UnblendedCost"

# Get cost forecast for next month
aws ce get-cost-forecast \
  --time-period Start=2024-02-01,End=2024-02-28 \
  --granularity MONTHLY \
  --metric UNBLENDED_COST
```

---

## AWS Pricing Calculator

### What is it?

The [AWS Pricing Calculator](https://calculator.aws) lets you **estimate costs before deploying** anything. It's free to use and doesn't require an AWS account.

### How to Use It

1. Go to [calculator.aws](https://calculator.aws)
2. Click **Create estimate**
3. Search for a service (e.g., "EC2")
4. Configure the service (instance type, region, storage, etc.)
5. Add more services as needed
6. Get a **monthly cost estimate**

### Example: Estimate a Simple Web App

| Component | Configuration | Estimated Monthly Cost |
|---|---|---|
| EC2 | 2x t3.small, On-Demand | ~$30 |
| RDS | 1x db.t3.small, Multi-AZ | ~$50 |
| ALB | 1 ALB, 10 LCU-hours | ~$25 |
| S3 | 50 GB storage | ~$1 |
| Data Transfer | 100 GB out | ~$9 |
| **Total** | | **~$115/month** |

> [!tip] Always estimate before deploying
> Run the calculator BEFORE creating resources. It prevents surprises.

---

## Common Costs Breakdown

### What Does Each Service Actually Charge For?

#### EC2 Costs

| Component | What You Pay | Notes |
|---|---|---|
| Instance hours | Per hour/second the instance runs | Stopped = no charge (but EBS still costs) |
| EBS storage | Per GB-month of allocated storage | Even when instance is stopped! |
| Elastic IPs | Free if attached to running instance | $0.005/hr if NOT attached |
| Data transfer | Outbound data costs money | Inbound is free |
| Snapshots | Per GB-month of snapshot data | Incremental (cheap) |

#### S3 Costs

| Component | What You Pay | Approximate Cost |
|---|---|---|
| Storage | Per GB stored per month | ~$0.023/GB (Standard) |
| PUT requests | Per 1,000 requests | ~$0.005 |
| GET requests | Per 1,000 requests | ~$0.0004 |
| Data transfer out | Per GB transferred out | ~$0.09/GB |
| Data transfer in | Free | $0 |

#### RDS Costs

| Component | What You Pay |
|---|---|
| Instance hours | Per hour the DB runs (Multi-AZ = 2x) |
| Storage | Per GB-month (gp3: ~$0.115/GB) |
| Backup storage | Free up to your DB size |
| Snapshots | Per GB-month beyond free tier |
| Data transfer | Outbound costs, inbound free |

#### Data Transfer Costs (IMPORTANT!)

| Direction | Cost |
|---|---|
| **Internet → AWS** (inbound) | **Free** |
| **AWS → Internet** (outbound) | ~$0.09/GB (first 10TB) |
| **Between AZs** (same region) | ~$0.01/GB each way |
| **Between Regions** | ~$0.02/GB |
| **Within same AZ** | Free (using private IPs) |

> [!warning] Data transfer between AZs adds up
> If you have heavy traffic between an EC2 in AZ-a and an RDS in AZ-b, the cross-AZ data transfer costs can surprise you. This is the cost of Multi-AZ high availability.

---

## Cost Optimization Tips

### For Learning / Development

1. **Use `t3.micro` instances** — They're free tier eligible for 12 months
2. **Stop EC2 instances when not using them** — You still pay for EBS, but not compute
   ```bash
   # Stop an instance (keeps EBS, stops compute charges)
   aws ec2 stop-instances --instance-ids i-xxx
   
   # Start it again when you need it
   aws ec2 start-instances --instance-ids i-xxx
   ```
3. **Terminate resources you don't need** — Stopping ≠ deleting. Terminate to stop ALL charges.
   ```bash
   # Terminate an instance (deletes it completely)
   aws ec2 terminate-instances --instance-ids i-xxx
   
   # Delete an RDS instance
   aws rds delete-db-instance --db-instance-identifier my-db --skip-final-snapshot
   
   # Delete NAT Gateway (saves ~$32/month!)
   aws ec2 delete-nat-gateway --nat-gateway-id nat-xxx
   
   # Release Elastic IP (charges when not attached)
   aws ec2 release-address --allocation-id eipalloc-xxx
   ```
4. **Delete NAT Gateways** when not in use — They cost ~$32/month
5. **Release unattached Elastic IPs** — AWS charges for unused EIPs
6. **Use Spot Instances** for testing — Up to 90% cheaper than On-Demand

### For Production

7. **Use Savings Plans** — Commit to 1 or 3 year usage for up to 72% savings
8. **Use Reserved Instances** — Similar to Savings Plans but instance-specific
9. **Use Spot Instances** for batch processing, CI/CD, data analysis
10. **Right-size instances** — Don't use a `t3.xlarge` when a `t3.small` would suffice
    ```bash
    # Check CloudWatch metrics to find underutilized instances
    aws cloudwatch get-metric-statistics \
      --namespace AWS/EC2 \
      --metric-name CPUUtilization \
      --dimensions Name=InstanceId,Value=i-xxx \
      --start-time 2024-01-01T00:00:00Z \
      --end-time 2024-01-31T23:59:59Z \
      --period 86400 \
      --statistics Average
    ```
11. **Set S3 Lifecycle Policies** — Move old data to cheaper storage classes
    ```bash
    aws s3api put-bucket-lifecycle-configuration \
      --bucket my-bucket \
      --lifecycle-configuration '{
        "Rules": [
          {
            "ID": "MoveToIA",
            "Status": "Enabled",
            "Filter": {"Prefix": ""},
            "Transitions": [
              {"Days": 30, "StorageClass": "STANDARD_IA"},
              {"Days": 90, "StorageClass": "GLACIER"}
            ]
          }
        ]
      }'
    ```
12. **Use auto-scaling** — Scale in during low traffic, scale out during peak

---

## AWS Trusted Advisor

### What is Trusted Advisor?

Trusted Advisor is an automated tool that inspects your AWS environment and gives **recommendations** across five categories:

| Category | What It Checks | Example |
|---|---|---|
| **Cost Optimization** | Underutilized resources | "You have 3 idle EC2 instances" |
| **Performance** | Overutilized resources | "Your RDS CPU is at 95%" |
| **Security** | Open ports, IAM issues | "Port 22 is open to 0.0.0.0/0" |
| **Fault Tolerance** | Single points of failure | "RDS is not Multi-AZ" |
| **Service Limits** | Approaching AWS limits | "85% of your VPC limit used" |

### Free vs Paid Checks

| Plan | Available Checks |
|---|---|
| **Basic (Free)** | 7 core checks (security + service limits) |
| **Business/Enterprise Support** | All checks (~50+) |

### Free Checks Include

1. S3 bucket permissions (public access)
2. Security group rules (unrestricted ports)
3. IAM use (root account usage)
4. MFA on root account
5. EBS public snapshots
6. RDS public snapshots
7. Service limits

```bash
# List all Trusted Advisor checks
aws support describe-trusted-advisor-checks --language en

# Get results for a specific check
aws support describe-trusted-advisor-check-result \
  --check-id "Pfx0RwqBli" \
  --language en

# Refresh a check
aws support refresh-trusted-advisor-check --check-id "Pfx0RwqBli"
```

> [!note] Trusted Advisor API requires Business or Enterprise Support plan.
> The console shows free checks for all accounts.

---

## Tagging Strategy

### Why Tag?

Tags are **key-value labels** you attach to AWS resources. They're essential for:
- **Cost allocation** — See costs per project, team, or environment
- **Automation** — Scripts that act on resources by tag (e.g., stop all "Dev" instances at night)
- **Organization** — Know what a resource is for and who owns it
- **Access control** — IAM policies can restrict actions based on tags

### Recommended Tags

| Tag Key | Example Value | Purpose |
|---|---|---|
| `Environment` | Dev, Staging, Prod | Identify environment |
| `Project` | MyApp, DataPipeline | Cost allocation by project |
| `Owner` | john@company.com | Know who to contact |
| `CostCenter` | Engineering, Marketing | Department billing |
| `ManagedBy` | Terraform, Manual | Track how it was created |
| `AutoStop` | true, false | Automated shutdown for dev |

### Tagging Hands-On

```bash
# Tag an EC2 instance
aws ec2 create-tags \
  --resources i-xxx \
  --tags Key=Environment,Value=Dev Key=Project,Value=MyApp Key=Owner,Value=john@company.com

# Tag an RDS instance
aws rds add-tags-to-resource \
  --resource-name arn:aws:rds:ap-south-1:123456789012:db:my-database \
  --tags Key=Environment,Value=Dev Key=Project,Value=MyApp

# Tag an S3 bucket
aws s3api put-bucket-tagging \
  --bucket my-bucket \
  --tagging 'TagSet=[{Key=Environment,Value=Dev},{Key=Project,Value=MyApp}]'

# Find all resources with a specific tag
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=Environment,Values=Dev \
  --query 'ResourceTagMappingList[*].ResourceARN'
```

> [!tip] Tag Everything
> Make tagging a habit. Tag every resource the moment you create it.
> Enable **Cost Allocation Tags** in the Billing console to see per-tag cost reports.

### Enable Cost Allocation Tags

1. Go to **Billing** → **Cost Allocation Tags**
2. Select your custom tags (e.g., Environment, Project)
3. Click **Activate**
4. Wait 24 hours — tags will appear in Cost Explorer

---

## Avoiding Bill Shock

### The 10 Commandments of AWS Billing

1. **Set up budgets and alerts FIRST** — Before creating any resource
2. **Check your billing dashboard weekly** — Console → Billing → Dashboard
3. **Know what's free tier eligible** — And track your usage against limits
4. **Terminate resources when done** — Stopping is NOT enough (EBS still costs money)
5. **Watch for hidden costs** — NAT Gateways ($32/mo), Elastic IPs, data transfer
6. **Don't launch in multiple regions accidentally** — Check your region dropdown!
7. **Be careful with "production" checkboxes** — Multi-AZ, provisioned IOPS = expensive
8. **Delete snapshots you don't need** — EBS and RDS snapshots accumulate over time
9. **Review auto-created resources** — Some services create EBS volumes, EIPs, etc. automatically
10. **Enable Cost Anomaly Detection** — AWS alerts you when spending patterns change

### Enable Cost Anomaly Detection

```bash
# Create a cost anomaly monitor for all services
aws ce create-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "AllServicesMonitor",
    "MonitorType": "DIMENSIONAL",
    "MonitorDimension": "SERVICE"
  }'

# Create an alert subscription
aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName": "CostAnomalyAlerts",
    "MonitorArnList": ["arn:aws:ce::123456789012:anomalymonitor/xxx"],
    "Subscribers": [
      {
        "Address": "you@email.com",
        "Type": "EMAIL"
      }
    ],
    "Frequency": "IMMEDIATE",
    "ThresholdExpression": {
      "Dimensions": {
        "Key": "ANOMALY_TOTAL_IMPACT_ABSOLUTE",
        "Values": ["5"],
        "MatchOptions": ["GREATER_THAN_OR_EQUAL"]
      }
    }
  }'
```

### Monthly Billing Checklist

- [ ] Check billing dashboard for unexpected charges
- [ ] Review Cost Explorer for cost trends
- [ ] Check free tier usage dashboard
- [ ] Look for idle/unused resources (stopped EC2, unattached EBS, unused EIPs)
- [ ] Review Trusted Advisor recommendations
- [ ] Delete resources from completed experiments
- [ ] Verify budget alerts are still configured

### AWS Organizations (Multi-Account)

If you have multiple AWS accounts (e.g., Dev, Staging, Prod):
- **Consolidated billing** — One bill for all accounts
- **Volume discounts** — Combined usage gets better pricing tiers
- **Service Control Policies (SCPs)** — Restrict what accounts can do
- **Cost allocation by account** — See per-account spending

---

## Summary

| What You Learned | Key Takeaway |
|---|---|
| Free Tier | Always Free + 12-Month Free + Trials |
| Billing Alerts | Set up BEFORE deploying anything |
| AWS Budgets | Set spending limits with email alerts |
| Cost Explorer | Visualize and analyze your spending |
| Pricing Calculator | Estimate costs before deploying |
| Common Costs | Instance hours + storage + data transfer |
| Cost Optimization | Stop/terminate unused resources |
| Trusted Advisor | Automated recommendations |
| Tagging | Tag everything for cost tracking |
| Anomaly Detection | Get alerted to unusual spending |

> [!success] Golden Rule
> **The best way to save money on AWS: delete what you're not using.**
> It sounds obvious, but forgotten resources are the #1 cause of unexpected bills.

---

## Cross-References

- [[Cost Management]] — General cloud cost management principles
- [[AWS Getting Started]] — Account setup and first steps
- [[Azure Billing]] — How Azure handles billing (for comparison)
- [[AWS Databases]] — Database pricing details (RDS, DynamoDB)
- [[AWS Networking]] — NAT Gateway and data transfer costs
- [[AWS Compute]] — EC2 pricing models (On-Demand, Spot, Reserved)
