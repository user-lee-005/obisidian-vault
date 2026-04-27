# AWS Getting Started

> **Amazon Web Services (AWS)** is the world's largest and most widely adopted cloud platform. This note covers everything you need to get started — from creating your account to running your first CLI commands.

---

## What is AWS?

**Amazon Web Services (AWS)** is a cloud computing platform provided by Amazon. It offers over **200+ fully-featured services** from data centres around the world.

### Key Facts

| Detail                | Info                                                     |
| --------------------- | -------------------------------------------------------- |
| **Launched**          | 2006 (started with S3 and SQS)                          |
| **Market Share**      | ~31% of the global cloud market (largest provider)       |
| **Regions**           | 30+ geographic regions worldwide                         |
| **Availability Zones**| 90+ AZs (isolated data centres within regions)           |
| **Services**          | 200+ services across compute, storage, database, AI, etc.|

### Why AWS?

- **Pay-as-you-go** — Only pay for what you use, no upfront costs
- **Global infrastructure** — Deploy close to your users anywhere in the world
- **Scalability** — Scale up or down in minutes, not weeks
- **Security** — Built-in compliance and security features
- **Ecosystem** — Largest community, most documentation, most certifications
- **Innovation** — Constantly releasing new services and features

### What Can You Do With AWS?

- Host websites and web applications ([[AWS Compute]])
- Store files and data ([[AWS Storage]])
- Run databases (RDS, DynamoDB, Aurora)
- Build serverless applications (Lambda, API Gateway)
- Deploy machine learning models (SageMaker)
- Stream and process data (Kinesis, Kafka)
- Manage containers (ECS, EKS, Fargate)
- And hundreds more...

---

## Creating an AWS Account

> ⚠️ You'll need a **credit card** to create an account, but the **Free Tier** means you won't be charged for most beginner usage.

### Step-by-Step Guide

#### Step 1: Go to the AWS Website

- Open your browser and navigate to: **https://aws.amazon.com**
- Click the **"Create an AWS Account"** button (top-right corner)

#### Step 2: Enter Account Details

- **Email address** — Use a personal or work email you check regularly
- **Password** — Use a strong, unique password (12+ characters recommended)
- **AWS account name** — This is just a label (e.g., "MyLearningAccount")
- Click **"Verify email address"** — AWS will send you a verification code

#### Step 3: Contact Information

- Choose **"Personal"** for a learning account (or "Business" for work)
- Fill in your full name, phone number, and address
- Accept the AWS Customer Agreement

#### Step 4: Add Payment Method

- Enter a valid **credit or debit card**
- AWS may place a temporary $1 hold to verify the card (refunded immediately)
- 💡 **Tip**: You will NOT be charged if you stay within the Free Tier limits

#### Step 5: Verify Your Identity

- AWS will call or text your phone number
- Enter the verification code displayed on screen

#### Step 6: Choose a Support Plan

| Plan          | Cost     | Best For                  |
| ------------- | -------- | ------------------------- |
| **Basic**     | Free     | Learning and experimenting |
| Developer     | $29/month| Building on AWS           |
| Business      | $100/month+| Running production workloads|
| Enterprise    | $15,000/month+| Large organisations    |

- ✅ **Choose Basic (Free)** — It's more than enough for learning

#### Step 7: Sign In to the Console

- Go to: **https://console.aws.amazon.com**
- Sign in with the email and password you just created
- You're now using the **root account** — we'll secure this next

### ⚠️ CRITICAL: Secure Your Root Account Immediately

The root account has **unlimited access** to everything. If compromised, an attacker could spin up thousands of expensive instances.

**Enable MFA (Multi-Factor Authentication) RIGHT NOW:**

1. Click your account name (top-right) → **Security credentials**
2. Under "Multi-factor authentication (MFA)" → **Assign MFA device**
3. Choose **"Authenticator app"**
4. Scan the QR code with Google Authenticator or Authy
5. Enter two consecutive MFA codes to verify
6. Click **"Add MFA"**

> 🔐 After this, every login to the root account will require your password + a 6-digit code from your phone.

See [[AWS IAM]] for more on securing your account.

---

## The AWS Console

The **AWS Management Console** is the web-based interface for managing all your AWS services.

### Console Tour

#### 1. Dashboard / Home Page

When you first log in, you'll see:
- **Recently visited services** — Quick access to services you've used
- **AWS Health Dashboard** — Status of AWS services
- **Cost & Usage** — Quick view of your spending
- **Tutorials and resources** — Guided learning paths

#### 2. Service Search Bar (Top-Left)

- The **fastest way** to find any service
- Click the search bar or press **Alt + S**
- Type the service name: "EC2", "S3", "Lambda"
- It also searches features, documentation, and marketplace

#### 3. Region Selector (Top-Right)

> ⚠️ **ALWAYS check your region before creating resources!**

- AWS has **30+ regions** around the world
- Each region is a separate geographic area with its own resources
- Resources created in `us-east-1` are NOT visible in `ap-south-1`
- Click the region name (e.g., "N. Virginia") to switch

**Common Regions:**

| Region Code      | Location             | Notes                        |
| ---------------- | -------------------- | ---------------------------- |
| `us-east-1`      | N. Virginia          | Oldest region, most services |
| `us-west-2`      | Oregon               | Popular for dev/test         |
| `eu-west-1`      | Ireland              | Popular for European users   |
| `ap-south-1`     | Mumbai               | Popular for Indian users     |
| `ap-southeast-1` | Singapore            | Southeast Asia               |

💡 **Tip**: Pick the region closest to your users for lowest latency.

#### 4. Account Menu (Top-Right)

Click your account name to access:
- **Account** — Account settings, alternate contacts
- **Billing and Cost Management** — View charges, set budgets
- **Security credentials** — MFA, access keys (for root)
- **Switch role** — Assume different IAM roles
- **Sign out**

#### 5. CloudShell (Bottom-Left Icon)

- A **browser-based terminal** with AWS CLI pre-installed
- No need to install anything locally
- Free to use (1GB persistent storage)
- Great for quick CLI commands

---

## AWS Free Tier

AWS offers a generous free tier so you can learn and experiment without charges.

### Three Types of Free Tier

#### 1. Always Free ✅

These never expire — available to all AWS customers forever:

| Service          | Free Allowance                          |
| ---------------- | --------------------------------------- |
| **Lambda**       | 1 million requests/month                |
| **DynamoDB**     | 25 GB storage, 25 read/write capacity   |
| **SNS**          | 1 million publishes/month               |
| **SQS**          | 1 million requests/month                |
| **CloudWatch**   | 10 custom metrics, 10 alarms            |
| **CloudFormation**| No additional charge (pay for resources)|
| **IAM**          | Always free                             |

#### 2. 12 Months Free 📅

Free for the first 12 months after account creation:

| Service          | Free Allowance                          |
| ---------------- | --------------------------------------- |
| **EC2**          | 750 hours/month of t2.micro (or t3.micro in some regions) |
| **S3**           | 5 GB standard storage                  |
| **RDS**          | 750 hours/month of db.t2.micro         |
| **ELB**          | 750 hours/month + 15 GB data           |
| **EBS**          | 30 GB of General Purpose SSD           |
| **Data Transfer**| 15 GB outbound per month               |

#### 3. Short-Term Trials 🔬

Limited-time trials for specific services:

| Service          | Trial Period                            |
| ---------------- | --------------------------------------- |
| **SageMaker**    | 2 months free (250 hours t2.medium)     |
| **Redshift**     | 2 months free (750 hours DC2.Large)     |
| **GuardDuty**    | 30-day free trial                       |
| **Inspector**    | 15-day free trial                       |

### ⚠️ Avoiding Surprise Charges

Even with the Free Tier, you can accidentally incur charges. Protect yourself:

#### Set Up Billing Alerts

1. Go to **Billing and Cost Management** → **Budgets**
2. Click **"Create a budget"**
3. Choose **"Zero spend budget"** — alerts you at the first dollar
4. Set your email for notifications
5. Click **Create**

#### Additional Tips

- **Stop/terminate EC2 instances** when not using them (stopped instances still pay for EBS)
- **Delete unused EBS volumes** — they cost money even when unattached
- **Check Elastic IPs** — unattached Elastic IPs cost money
- **Monitor S3 usage** — data transfer out costs money
- **Set up AWS Cost Explorer** — visualize where your money goes

See [[AWS Billing]] for a deep dive on cost management.

---

## Installing AWS CLI

The **AWS Command Line Interface (CLI)** lets you manage AWS services from your terminal. It's essential for automation and faster workflows.

### Why Use the CLI?

- **Faster** than clicking through the console
- **Scriptable** — automate repetitive tasks
- **Reproducible** — save commands and re-run them
- **Required** for CI/CD pipelines and Infrastructure as Code

### Installation by Operating System

#### Windows

```powershell
# Option 1: MSI Installer (recommended)
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi

# Option 2: Download the installer manually
# Go to: https://awscli.amazonaws.com/AWSCLIV2.msi
# Double-click and follow the wizard
```

After installation, **restart your terminal** and verify:

```powershell
aws --version
# Expected output: aws-cli/2.x.x Python/3.x.x Windows/10 exe/AMD64
```

#### macOS

```bash
# Option 1: Homebrew (easiest)
brew install awscli

# Option 2: Official installer
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
```

Verify:

```bash
aws --version
# Expected output: aws-cli/2.x.x Python/3.x.x Darwin/x.x.x
```

#### Linux

```bash
# Download and install
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify
aws --version
# Expected output: aws-cli/2.x.x Python/3.x.x Linux/x.x.x
```

#### Verify Installation (All Platforms)

```bash
aws --version
```

If you see a version number, you're good to go! 🎉

---

## Configuring AWS CLI

Before you can use the CLI, you need to tell it **who you are** (your credentials) and **where to work** (your default region).

### Step 1: Create Access Keys

> ⚠️ **Do NOT create access keys for the root account.** Create an IAM user first. See [[AWS IAM]].

1. Go to the **IAM Console** → **Users** → Select your user
2. Click the **"Security credentials"** tab
3. Under "Access keys" → Click **"Create access key"**
4. Choose **"Command Line Interface (CLI)"**
5. Acknowledge the recommendation and click **Next**
6. Click **"Create access key"**
7. **Copy both the Access Key ID and Secret Access Key** — you won't see the secret again!

### Step 2: Run `aws configure`

```bash
aws configure
```

You'll be prompted for 4 values:

```
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: ap-south-1
Default output format [None]: json
```

| Setting                | What It Does                              | Recommended Value |
| ---------------------- | ----------------------------------------- | ----------------- |
| **Access Key ID**      | Identifies your IAM user                  | Your key          |
| **Secret Access Key**  | Authenticates your IAM user               | Your secret       |
| **Default region**     | Region for all commands (unless overridden)| `ap-south-1`     |
| **Output format**      | How results are displayed                 | `json`            |

### Named Profiles

You can configure **multiple profiles** for different accounts or environments:

```bash
# Create a "dev" profile
aws configure --profile dev

# Create a "prod" profile
aws configure --profile prod

# Use a specific profile
aws s3 ls --profile dev

# Set default profile via environment variable
export AWS_PROFILE=dev         # Linux/Mac
$env:AWS_PROFILE = "dev"       # PowerShell
```

### Where Credentials Are Stored

The CLI stores your configuration in two files:

```
~/.aws/credentials   ← Access keys (NEVER share this file!)
~/.aws/config        ← Region, output format, profile settings
```

---

## First CLI Commands

Now that you're set up, let's run some commands to verify everything works.

### Identity Check

```bash
# Who am I? — Returns your IAM user/role info
aws sts get-caller-identity
```

Expected output:

```json
{
    "UserId": "AIDACKCEVSQ6C2EXAMPLE",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/developer1"
}
```

### Explore Your Environment

```bash
# List all S3 buckets in your account
aws s3 ls

# List all available AWS regions
aws ec2 describe-regions --output table

# List Availability Zones in your current region
aws ec2 describe-availability-zones --output table

# List all EC2 instances
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,State.Name,InstanceType]' --output table

# List IAM users
aws iam list-users --output table
```

### Helpful CLI Flags

```bash
# Override region for a single command
aws s3 ls --region us-east-1

# Change output format for a single command
aws ec2 describe-regions --output table

# Get help for any command
aws ec2 help
aws s3 cp help

# Use --query to filter JSON output (JMESPath syntax)
aws ec2 describe-instances --query 'Reservations[*].Instances[*].InstanceId'

# Dry run — check permissions without actually making changes
aws ec2 run-instances --dry-run --image-id ami-12345 --instance-type t2.micro
```

---

## AWS Resource Naming — ARNs

Every resource in AWS has an **Amazon Resource Name (ARN)** — a unique identifier.

### ARN Format

```
arn:aws:service:region:account-id:resource-type/resource-id
```

### ARN Examples

| Resource       | ARN Example                                                      |
| -------------- | ---------------------------------------------------------------- |
| **S3 Bucket**  | `arn:aws:s3:::my-bucket`                                         |
| **S3 Object**  | `arn:aws:s3:::my-bucket/my-file.txt`                             |
| **EC2 Instance** | `arn:aws:ec2:ap-south-1:123456789012:instance/i-0abcdef1234567890` |
| **IAM User**   | `arn:aws:iam::123456789012:user/developer1`                      |
| **IAM Role**   | `arn:aws:iam::123456789012:role/EC2-S3-Access`                   |
| **Lambda**     | `arn:aws:lambda:ap-south-1:123456789012:function:HelloFunction`  |
| **DynamoDB**   | `arn:aws:dynamodb:ap-south-1:123456789012:table/Users`           |

> 💡 Note: **IAM** and **S3** ARNs don't have a region — they're global services.

---

## Key Concepts

### 1. Everything is an API Call

Every action in AWS — whether you click a button in the console, run a CLI command, or use an SDK — it's all **API calls** under the hood.

```
Console click → API call → AWS processes → Response
CLI command   → API call → AWS processes → Response
SDK function  → API call → AWS processes → Response
```

This means:
- You can automate **anything** the console does
- Every action is logged in **CloudTrail** (audit trail)
- You can control access with **IAM policies** at the API level

### 2. Everything is Regional

Most AWS services are **regional** — resources in one region are completely separate from another.

**Global Services (exceptions):**
- **IAM** — Users, groups, roles, policies ([[AWS IAM]])
- **Route 53** — DNS service
- **CloudFront** — CDN (Content Delivery Network)
- **WAF** — Web Application Firewall
- **S3** — Bucket names are global, but data is stored in a specific region

**Why this matters:**
- If you create an EC2 instance in `ap-south-1`, you won't see it if you switch to `us-east-1`
- Always check your region selector in the console!
- Choose a region close to your users for better performance

### 3. Always Tag Your Resources

Tags are **key-value pairs** you attach to resources for organisation and cost tracking.

```bash
# Add tags when creating a resource
aws ec2 run-instances ... --tag-specifications \
  'ResourceType=instance,Tags=[{Key=Environment,Value=Development},{Key=Project,Value=MyApp},{Key=Owner,Value=developer1}]'

# Add tags to an existing resource
aws ec2 create-tags --resources i-0abcdef1234567890 --tags Key=Name,Value=WebServer
```

**Recommended Tags:**
- `Name` — Human-readable name
- `Environment` — dev, staging, prod
- `Project` — Which project this belongs to
- `Owner` — Who created/manages this
- `CostCenter` — For billing allocation

---

## What's Next?

Now that you have your account set up and CLI configured, here's your learning path:

1. ✅ **AWS Getting Started** — You are here!
2. 🔐 [[AWS IAM]] — Secure your account with proper identity management
3. 💻 [[AWS Compute]] — Launch your first EC2 instance and Lambda function
4. 📦 [[AWS Storage]] — Store data with S3, EBS, and EFS
5. 💰 [[AWS Billing]] — Understand costs and set up budget alerts
6. 🌐 [[AWS Networking]] — VPCs, subnets, and load balancers
7. 🗄️ [[AWS Databases]] — RDS, DynamoDB, and more
8. 🚀 [[AWS Deployment]] — Deploy applications with CI/CD

---

## Quick Reference Card

```bash
# === IDENTITY ===
aws sts get-caller-identity

# === S3 ===
aws s3 ls
aws s3 mb s3://bucket-name
aws s3 cp file.txt s3://bucket-name/

# === EC2 ===
aws ec2 describe-instances
aws ec2 describe-regions --output table
aws ec2 describe-availability-zones --output table

# === IAM ===
aws iam list-users
aws iam list-groups
aws iam list-roles

# === GENERAL ===
aws <service> help            # Get help
aws <service> <command> help  # Get help for specific command
--region <region>             # Override region
--output table|json|text      # Change output format
--profile <name>              # Use named profile
--query '<jmespath>'          # Filter output
```

---

## Related Notes

- [[Cloud Fundamentals]] — What is cloud computing? IaaS, PaaS, SaaS explained
- [[AWS IAM]] — Identity and Access Management deep dive
- [[AWS Billing]] — Cost management, budgets, and alerts
- [[AWS Compute]] — EC2, Lambda, containers
- [[AWS Storage]] — S3, EBS, EFS
