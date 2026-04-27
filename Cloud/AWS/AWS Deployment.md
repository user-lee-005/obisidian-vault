# AWS Deployment

> **Summary**: Getting your application from your laptop to running on AWS — from the simplest "just upload your code" approach to full infrastructure-as-code pipelines. This note covers every deployment option, when to use each one, and hands-on commands to do it yourself.

---

## Table of Contents

- [[#Deployment Options Overview]]
- [[#AWS Elastic Beanstalk]]
- [[#AWS CodePipeline CI/CD]]
- [[#AWS CloudFormation]]
- [[#End-to-End Deployment Walkthrough]]
- [[#Deployment Best Practices]]

---

## Deployment Options Overview

> **The Big Question**: "I have code on my laptop. How do I get it running on AWS so the world can use it?"

There are multiple ways, ranging from **simplest** to **most control**:

| Option | Type | Effort | Control | Best For |
|--------|------|--------|---------|----------|
| **Elastic Beanstalk** | PaaS | ⭐ Lowest | Low | Quick deploys, small apps |
| **ECS / Fargate** | Containers | ⭐⭐ Medium | Medium | Docker-based microservices |
| **EC2 + CodeDeploy** | IaaS + CI/CD | ⭐⭐⭐ High | High | Full custom infrastructure |
| **CloudFormation** | IaC | ⭐⭐⭐ High | Highest | Repeatable, version-controlled infra |

### Think of it Like This

- **Elastic Beanstalk** = Valet parking. Hand over your keys (code), they handle everything.
- **ECS/Fargate** = Renting a parking spot with an electric charger. You bring the container, AWS manages the rest.
- **EC2 + CodeDeploy** = Owning a garage. You control everything — security, layout, maintenance.
- **CloudFormation** = The **blueprint** for building any of the above. You describe what you want, AWS builds it.

> 💡 **Beginner tip**: Start with Elastic Beanstalk. As your needs grow, move to ECS or CloudFormation.

---

## AWS Elastic Beanstalk

> **What is it?** A Platform-as-a-Service (PaaS). You upload your code, and AWS automatically handles deployment, capacity provisioning, load balancing, auto-scaling, and health monitoring.

### What Beanstalk Supports

Elastic Beanstalk supports the following platforms out of the box:

- ☕ **Java** (Tomcat, Corretto)
- 🐍 **Python**
- 🟢 **Node.js**
- 💎 **Ruby**
- 🐘 **PHP**
- 🔵 **.NET** (on Linux or Windows)
- 🐹 **Go**
- 🐳 **Docker** (single or multi-container)

### What Beanstalk Creates for You Automatically

When you deploy an app to Elastic Beanstalk, it creates and manages:

1. **EC2 Instances** — The servers running your code (see [[AWS Compute]])
2. **Elastic Load Balancer** — Distributes traffic across instances (see [[AWS Networking]])
3. **Auto Scaling Group** — Adds/removes instances based on load
4. **Security Groups** — Firewall rules for your instances (see [[AWS Security]])
5. **CloudWatch Monitoring** — Metrics and health checks (see [[AWS Monitoring]])
6. **S3 Bucket** — Stores your application versions

> 🎯 **Key insight**: Beanstalk is NOT a separate service that runs your code. It **orchestrates other AWS services** for you. Under the hood, it's EC2, ELB, ASG, etc. You're just not managing them manually.

### Hands-On: Deploy a Sample Python App

#### Step 1: Install the EB CLI

```bash
# Install the Elastic Beanstalk CLI tool
pip install awsebcli

# Verify installation
eb --version
```

> 📋 **Prerequisite**: You need the AWS CLI configured with credentials first. See [[AWS Getting Started]] for setup.

#### Step 2: Initialize Your Project

```bash
# Navigate to your project directory
cd my-web-app

# Initialize Elastic Beanstalk in your project
# -p specifies the platform (python-3.12)
# --region specifies the AWS region
eb init -p python-3.12 my-app --region ap-south-1
```

**What this does**:
- Creates a `.elasticbeanstalk/` folder in your project
- Stores configuration (region, platform, app name)
- Does NOT create any AWS resources yet

#### Step 3: Create an Environment and Deploy

```bash
# Create a new environment and deploy your code
# This is where AWS actually provisions resources
eb create my-app-env
```

**What happens behind the scenes**:
1. Beanstalk zips your code and uploads it to S3
2. Creates an EC2 instance with your chosen platform
3. Creates a load balancer (if you chose a load-balanced env)
4. Sets up auto-scaling rules
5. Deploys your code to the instance(s)
6. Runs health checks

> ⏱️ This takes 5-10 minutes the first time.

#### Step 4: Access Your App

```bash
# Open your deployed app in the browser
eb open

# Check environment status
eb status

# View recent events
eb events
```

#### Step 5: Deploy Updates

```bash
# Make changes to your code, then:
eb deploy

# This uploads the new version and does a rolling update
```

#### Step 6: View Logs

```bash
# View recent logs
eb logs

# View logs in real-time (tail)
eb logs --stream
```

#### Step 7: Clean Up (IMPORTANT!)

```bash
# Terminate the environment to stop ALL charges
# This deletes EC2 instances, load balancers, etc.
eb terminate my-app-env

# Type the environment name to confirm
```

> ⚠️ **ALWAYS terminate** environments you're not using. Beanstalk environments cost money because they run EC2 instances, load balancers, etc.

### Environment Types

| Type | Purpose | Example |
|------|---------|---------|
| **Web Server** | Handles HTTP requests from users | A website, REST API |
| **Worker** | Processes background tasks from SQS queue | Email sending, image processing |

- **Web Server environments** get a URL (like `my-app-env.ap-south-1.elasticbeanstalk.com`)
- **Worker environments** poll an SQS queue for messages to process

### Configuration with .ebextensions

You can customize your Beanstalk environment by creating `.ebextensions/` in your project root:

```yaml
# .ebextensions/01-packages.config
packages:
  yum:
    git: []
    postgresql-devel: []

option_settings:
  aws:elasticbeanstalk:application:environment:
    DB_HOST: "my-rds-endpoint.rds.amazonaws.com"
    DEBUG: "false"

  aws:autoscaling:asg:
    MinSize: 2
    MaxSize: 4
```

**Common .ebextensions uses**:
- Install OS packages
- Set environment variables
- Configure auto-scaling limits
- Run commands during deployment
- Set up HTTPS with certificates

---

## AWS CodePipeline CI/CD

> **What is CI/CD?** Continuous Integration / Continuous Deployment. Automatically build, test, and deploy your code every time you push changes. No more "it works on my machine" or manual deployments.

AWS provides **four services** that work together as a CI/CD pipeline:

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ CodeCommit   │───▶│ CodeBuild    │───▶│ CodeDeploy   │───▶│ Deployed!   │
│ (Source)     │    │ (Build/Test) │    │ (Deploy)     │    │             │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
         │                                                        ▲
         └────── CodePipeline (Orchestrator) ─────────────────────┘
```

### AWS CodeCommit (Source Stage)

> **What**: A fully managed Git repository service hosted on AWS. Think of it as GitHub, but inside your AWS account.

```bash
# Create a CodeCommit repository
aws codecommit create-repository --repository-name my-app --repository-description "My application"

# Clone it
git clone https://git-codecommit.ap-south-1.amazonaws.com/v1/repos/my-app

# Or use an existing GitHub/Bitbucket repo — CodePipeline supports those too!
```

> 💡 **Practical tip**: Most teams use **GitHub** as the source and skip CodeCommit entirely. CodePipeline integrates with GitHub, Bitbucket, and CodeCommit.

### AWS CodeBuild (Build Stage)

> **What**: A fully managed build service. Compiles your code, runs tests, and produces deployable artifacts (JAR files, Docker images, etc.).

CodeBuild uses a **`buildspec.yml`** file in your project root to know what to do:

```yaml
# buildspec.yml — Instructions for CodeBuild
version: 0.2

phases:
  install:
    runtime-versions:
      java: corretto17     # Use Java 17 (Amazon Corretto)
    commands:
      - echo Installing dependencies...

  pre_build:
    commands:
      - echo Running tests...
      - mvn test            # Run unit tests FIRST

  build:
    commands:
      - echo Building the application...
      - mvn package -DskipTests    # Build the JAR/WAR

  post_build:
    commands:
      - echo Build completed on $(date)

artifacts:
  files:
    - target/*.jar          # Upload the built JAR as an artifact
  discard-paths: yes

cache:
  paths:
    - '/root/.m2/**/*'      # Cache Maven dependencies for faster builds
```

**buildspec.yml phases explained**:

| Phase | When | Purpose |
|-------|------|---------|
| `install` | First | Install dependencies, set runtime |
| `pre_build` | Before build | Run tests, login to registries |
| `build` | Main phase | Compile, package |
| `post_build` | After build | Push Docker images, notify |

```bash
# Create a CodeBuild project via CLI
aws codebuild create-project \
  --name my-app-build \
  --source type=GITHUB,location=https://github.com/user/my-app \
  --artifacts type=S3,location=my-build-bucket \
  --environment type=LINUX_CONTAINER,computeType=BUILD_GENERAL1_SMALL,image=aws/codebuild/amazonlinux2-x86_64-standard:4.0

# Start a build manually
aws codebuild start-build --project-name my-app-build

# View build status
aws codebuild batch-get-builds --ids my-app-build:build-id
```

### AWS CodeDeploy (Deploy Stage)

> **What**: Automates code deployment to EC2 instances, ECS services, or Lambda functions.

CodeDeploy uses an **`appspec.yml`** file to define how to deploy:

```yaml
# appspec.yml — Deployment instructions for EC2
version: 0.0
os: linux

files:
  - source: /
    destination: /var/www/my-app

hooks:
  BeforeInstall:
    - location: scripts/stop-server.sh
      timeout: 60
  AfterInstall:
    - location: scripts/install-deps.sh
      timeout: 120
  ApplicationStart:
    - location: scripts/start-server.sh
      timeout: 60
  ValidateService:
    - location: scripts/health-check.sh
      timeout: 60
```

**Deployment strategies in CodeDeploy**:

| Strategy | How | Risk | Speed |
|----------|-----|------|-------|
| **AllAtOnce** | Deploy to all instances simultaneously | High — all down if it fails | Fast |
| **OneAtATime** | Deploy one instance at a time | Low — rollback quickly | Slow |
| **HalfAtATime** | Deploy to half, then the other half | Medium | Medium |
| **Blue/Green** | Create new instances, switch traffic | Lowest — instant rollback | Medium |

> See [[Cloud Deployment Strategies]] for more on blue/green and canary deployments.

### AWS CodePipeline (The Orchestrator)

> **What**: Connects CodeCommit/GitHub → CodeBuild → CodeDeploy into an automated pipeline. Every push triggers the full pipeline.

#### Console Walkthrough: Setting Up a Pipeline

**Step 1: Open CodePipeline Console**
- Go to AWS Console → CodePipeline → "Create pipeline"
- Give your pipeline a name (e.g., `my-app-pipeline`)
- Let AWS create a new service role

**Step 2: Configure Source Stage**
- Source provider: **GitHub (Version 2)** — recommended
- Click "Connect to GitHub" → Authorize AWS
- Select your repository and branch (`main`)
- Detection: "Push in a branch" (triggers on every push)

**Step 3: Configure Build Stage**
- Build provider: **AWS CodeBuild**
- Click "Create project" (inline)
  - Environment: Managed image → Amazon Linux 2 → Standard runtime
  - Buildspec: "Use a buildspec file" (reads `buildspec.yml` from your repo)
- Click "Continue to CodePipeline"

**Step 4: Configure Deploy Stage**
- Deploy provider: **AWS Elastic Beanstalk**
- Select your Beanstalk application and environment
- (Alternatively: Choose ECS, EC2/CodeDeploy, S3, etc.)

**Step 5: Review and Create**
- Review all stages
- Click "Create pipeline"
- Pipeline runs automatically on creation!

**Now every `git push` to `main` will**:
1. Pull the latest code from GitHub
2. Build and test it with CodeBuild
3. Deploy it to Elastic Beanstalk
4. All automatically! 🎉

```bash
# List your pipelines
aws codepipeline list-pipelines

# View pipeline status
aws codepipeline get-pipeline-state --name my-app-pipeline

# Manually trigger a pipeline run
aws codepipeline start-pipeline-execution --name my-app-pipeline
```

---

## AWS CloudFormation

> **What is Infrastructure as Code (IaC)?** Instead of clicking around the AWS Console to create resources, you write a **template file** (YAML or JSON) that describes EXACTLY what resources you want. Then CloudFormation creates, updates, or deletes them for you.

### Why IaC Matters

| Manual (Console) | IaC (CloudFormation) |
|-------------------|----------------------|
| Click, click, click... | Write once, deploy everywhere |
| "How did we set up production?" | Template IS the documentation |
| Hard to replicate | Identical environments every time |
| Risky changes | Preview changes before applying |
| No version history | Git history for all infra changes |

### Key Concepts

- **Template** — The YAML/JSON file describing your resources (the blueprint)
- **Stack** — A running instance of a template (the actual resources)
- **Change Set** — Preview what will change before updating a stack
- **Drift Detection** — Check if someone manually changed resources outside CloudFormation
- **Nested Stacks** — Templates that reference other templates (for modularity)

### Template Structure

Every CloudFormation template has these sections:

```yaml
AWSTemplateFormatVersion: '2010-09-09'   # Always this value
Description: What this template does       # Human-readable description

Parameters:        # Input values (like function arguments)
  # ...

Mappings:          # Static lookup tables
  # ...

Conditions:        # Conditional logic
  # ...

Resources:         # THE MAIN SECTION — what to create (REQUIRED)
  # ...

Outputs:           # Values to display/export after creation
  # ...
```

### Example Template: EC2 Instance

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Simple EC2 Instance for learning CloudFormation

Parameters:
  InstanceType:
    Type: String
    Default: t2.micro
    AllowedValues:
      - t2.micro
      - t2.small
      - t2.medium
    Description: EC2 instance type (t2.micro is free tier)

  KeyPairName:
    Type: AWS::EC2::KeyPair::KeyName
    Description: Name of an existing EC2 key pair for SSH access

Resources:
  MySecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow SSH and HTTP
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      ImageId: ami-0c55b159cbfafe1f0
      KeyName: !Ref KeyPairName
      SecurityGroupIds:
        - !Ref MySecurityGroup
      Tags:
        - Key: Name
          Value: CloudFormation-Demo-Instance

Outputs:
  InstanceId:
    Description: Instance ID of the created EC2 instance
    Value: !Ref MyInstance

  PublicIP:
    Description: Public IP address of the instance
    Value: !GetAtt MyInstance.PublicIp

  PublicDNS:
    Description: Public DNS name of the instance
    Value: !GetAtt MyInstance.PublicDnsName
```

### Intrinsic Functions (CloudFormation Magic)

| Function | What It Does | Example |
|----------|-------------|---------|
| `!Ref` | Reference a parameter or resource | `!Ref InstanceType` |
| `!GetAtt` | Get an attribute of a resource | `!GetAtt MyInstance.PublicIp` |
| `!Sub` | String substitution | `!Sub "Hello ${AWS::Region}"` |
| `!Join` | Join strings | `!Join ["-", ["my", "stack"]]` |
| `!Select` | Pick from a list | `!Select [0, [a, b, c]]` |
| `!If` | Conditional value | `!If [IsProd, t2.large, t2.micro]` |

### Hands-On: Using CloudFormation

```bash
# Create a stack from a template file
aws cloudformation create-stack \
  --stack-name my-first-stack \
  --template-body file://template.yaml \
  --parameters ParameterKey=InstanceType,ParameterValue=t2.micro

# Wait for creation to complete
aws cloudformation wait stack-create-complete --stack-name my-first-stack

# Check stack status
aws cloudformation describe-stacks --stack-name my-first-stack

# View stack events (see what happened step by step)
aws cloudformation describe-stack-events --stack-name my-first-stack

# View outputs
aws cloudformation describe-stacks --stack-name my-first-stack \
  --query "Stacks[0].Outputs"
```

#### Updating a Stack

```bash
# Create a change set (PREVIEW changes before applying)
aws cloudformation create-change-set \
  --stack-name my-first-stack \
  --change-set-name my-update \
  --template-body file://template-v2.yaml

# Review the change set
aws cloudformation describe-change-set \
  --stack-name my-first-stack \
  --change-set-name my-update

# Execute the change set (apply changes)
aws cloudformation execute-change-set \
  --stack-name my-first-stack \
  --change-set-name my-update

# Or update directly (skips change set preview)
aws cloudformation update-stack \
  --stack-name my-first-stack \
  --template-body file://template-v2.yaml
```

#### Deleting a Stack

```bash
# Delete the stack — THIS DELETES ALL RESOURCES IT CREATED
aws cloudformation delete-stack --stack-name my-first-stack

# Wait for deletion
aws cloudformation wait stack-delete-complete --stack-name my-first-stack
```

> ⚠️ **Important**: Deleting a stack deletes ALL the resources it created (EC2 instances, security groups, etc.). This is actually a feature — clean up is easy!

### CloudFormation vs Terraform

| Feature | CloudFormation | Terraform |
|---------|---------------|-----------|
| Provider | AWS only | Multi-cloud (AWS, Azure, GCP, etc.) |
| Language | YAML/JSON | HCL (HashiCorp Configuration Language) |
| State | Managed by AWS automatically | You manage state file |
| Cost | Free (you pay for resources) | Free (open source) |
| Rollback | Automatic on failure | Manual |
| Ecosystem | AWS-native, deep integration | Huge community, many providers |

> 💡 **When to choose**: If you're 100% AWS, CloudFormation is simpler. If you use multiple clouds or want portability, choose Terraform. Many teams use both!

---

## End-to-End Deployment Walkthrough

> **Scenario**: "I have a web application. How do I get it running on AWS?"

### Option A: Simple — Elastic Beanstalk (5 minutes)

**Best for**: Quick demos, small apps, learning

```bash
# 1. Navigate to your project
cd my-web-app

# 2. Initialize Beanstalk
eb init -p python-3.12 my-web-app --region ap-south-1

# 3. Deploy!
eb create my-web-app-prod

# 4. Open in browser
eb open

# Done! Your app is live with load balancing and auto-scaling.
```

**What you get**: URL like `my-web-app-prod.ap-south-1.elasticbeanstalk.com`

### Option B: Containers — Docker → ECR → ECS Fargate

**Best for**: Microservices, Docker-based apps

```bash
# 1. Build your Docker image
docker build -t my-web-app .

# 2. Create an ECR repository (AWS's Docker Hub)
aws ecr create-repository --repository-name my-web-app

# 3. Authenticate Docker with ECR
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-south-1.amazonaws.com

# 4. Tag and push your image
docker tag my-web-app:latest <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-web-app:latest
docker push <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-web-app:latest

# 5. Create ECS cluster and service (via Console or CloudFormation)
# AWS Console → ECS → Create Cluster → Fargate → Create Service
```

> See [[AWS Compute]] for more on ECS and Fargate.

### Option C: Full Control — EC2 + CodePipeline + CloudFormation

**Best for**: Production workloads, teams, enterprise

1. **Define infrastructure** in a CloudFormation template (VPC, EC2, RDS, ALB)
2. **Deploy infrastructure**: `aws cloudformation create-stack ...`
3. **Set up CI/CD**: CodePipeline (GitHub → CodeBuild → CodeDeploy)
4. **Every git push** automatically builds, tests, and deploys

This is the most work upfront, but gives you:
- Full control over every resource
- Repeatable, version-controlled infrastructure
- Automated deployments with rollback
- Separate environments (dev/staging/prod) from the same template

---

## Deployment Best Practices

### 1. Use Infrastructure as Code (IaC) for EVERYTHING

```
❌ Bad:  Create resources by clicking around the Console
✅ Good: Write a CloudFormation template or Terraform config
```

**Why**: Reproducible, version-controlled, reviewable, auditable.

### 2. Use CI/CD Pipelines — Never Deploy Manually in Production

```
❌ Bad:  SSH into server, git pull, restart
✅ Good: Push to main → Pipeline builds, tests, deploys automatically
```

**Why**: Human error is the #1 cause of outages. Automation is consistent.

### 3. Use Separate Environments

```
dev  → For development and testing
staging → Mirror of production for final testing
prod → The real thing — users interact with this
```

- Use the SAME CloudFormation template for all environments
- Pass different parameters (instance size, database config)
- Never test in production!

### 4. Use Safe Deployment Strategies

| Strategy | How It Works | When to Use |
|----------|-------------|-------------|
| **Rolling** | Update instances in batches | Standard deployments |
| **Blue/Green** | Deploy to new env, switch traffic | Zero-downtime required |
| **Canary** | Send 5% traffic to new version first | High-risk changes |
| **All-at-Once** | Update everything simultaneously | Dev/test only |

> See [[Cloud Deployment Strategies]] for detailed explanations of each strategy.

### 5. Tag Everything, Version Everything

```bash
# Tag all resources for cost tracking and management
Tags:
  - Key: Environment
    Value: production
  - Key: Team
    Value: backend
  - Key: Application
    Value: my-web-app
  - Key: CostCenter
    Value: engineering
```

### 6. Implement Rollback Plans

- **Elastic Beanstalk**: Automatic rollback on health check failure
- **CodeDeploy**: Automatic rollback on deployment failure
- **CloudFormation**: Automatic rollback on stack update failure
- **Always have a plan to go back to the previous version**

### 7. Monitor Every Deployment

- Watch [[AWS Monitoring]] dashboards during deployments
- Set up CloudWatch Alarms for error rates
- Use X-Ray to trace request latency changes
- Roll back immediately if metrics degrade

---

## Quick Reference: Deployment Commands Cheat Sheet

```bash
# === Elastic Beanstalk ===
eb init -p python-3.12 my-app --region ap-south-1
eb create my-app-env
eb deploy
eb open
eb logs
eb terminate my-app-env

# === CloudFormation ===
aws cloudformation create-stack --stack-name my-stack --template-body file://template.yaml
aws cloudformation describe-stacks --stack-name my-stack
aws cloudformation update-stack --stack-name my-stack --template-body file://updated.yaml
aws cloudformation delete-stack --stack-name my-stack

# === CodePipeline ===
aws codepipeline list-pipelines
aws codepipeline get-pipeline-state --name my-pipeline
aws codepipeline start-pipeline-execution --name my-pipeline

# === CodeBuild ===
aws codebuild start-build --project-name my-build
aws codebuild batch-get-builds --ids <build-id>

# === ECR (Docker Registry) ===
aws ecr create-repository --repository-name my-app
aws ecr get-login-password | docker login --username AWS --password-stdin <url>
docker push <account>.dkr.ecr.<region>.amazonaws.com/my-app:latest
```

---

## What to Learn Next

- [[AWS Getting Started]] — Set up your AWS account and CLI
- [[AWS Compute]] — Deep dive into EC2, ECS, Lambda
- [[AWS Networking]] — VPC, Load Balancers, DNS with Route 53
- [[AWS Monitoring]] — CloudWatch, alarms, dashboards
- [[AWS Security]] — IAM, encryption, firewalls
- [[Cloud Deployment Strategies]] — Blue/green, canary, rolling deployments
- [[Azure Deployment]] — Compare with Azure DevOps and App Service

---

> 📝 **Created for**: AWS Cloud Learning Path
> 🏷️ Tags: #aws #deployment #cicd #cloudformation #elastic-beanstalk #codepipeline #iac
