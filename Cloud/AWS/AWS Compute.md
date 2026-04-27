# AWS Compute

> **Compute** is the processing power that runs your applications in the cloud. AWS offers a range of compute services — from full virtual machines (EC2) to serverless functions (Lambda) to managed containers (ECS/EKS/Fargate).

---

## What is Compute?

In cloud computing, **compute** refers to the CPU, memory, and processing power you need to run applications. Instead of buying physical servers, you rent compute capacity from AWS.

### The Compute Spectrum

```
More Control ◄──────────────────────────────────────► Less Management

  EC2              ECS on EC2         Fargate          Lambda
  (Virtual         (Containers        (Serverless      (Serverless
   Machines)        on your VMs)       Containers)      Functions)

  You manage:      You manage:        You manage:      You manage:
  - OS             - Containers       - Containers     - Code only
  - Patches        - Task definitions - Task defs
  - Scaling        AWS manages:       AWS manages:     AWS manages:
  - Networking     - Orchestration    - Everything     - Everything
                                        else             else
```

### When to Use What

| Service      | Best For                                    | Example Use Case                     |
| ------------ | ------------------------------------------- | ------------------------------------ |
| **EC2**      | Full control, custom OS, legacy apps        | Running a Java Spring Boot server    |
| **Lambda**   | Event-driven, short tasks, APIs             | Processing S3 uploads, REST APIs     |
| **ECS**      | Containerised apps on AWS                   | Microservices with Docker            |
| **EKS**      | Kubernetes workloads                        | Teams already using K8s              |
| **Fargate**  | Containers without managing servers         | Serverless microservices             |

---

## Amazon EC2 (Elastic Compute Cloud)

**EC2** gives you virtual servers (called **instances**) in the cloud. You choose the OS, CPU, memory, storage, and network — then SSH in and use it like any server.

### Key Concepts

| Concept              | Description                                                       |
| -------------------- | ----------------------------------------------------------------- |
| **Instance**         | A virtual server running in the cloud                             |
| **AMI**              | Amazon Machine Image — template for the OS and software           |
| **Instance Type**    | Defines CPU, memory, storage, and network capacity                |
| **Key Pair**         | SSH keys for secure login to your instance                        |
| **Security Group**   | Firewall rules controlling inbound/outbound traffic               |
| **EBS Volume**       | Persistent block storage (like a hard drive) — see [[AWS Storage]]|
| **Elastic IP**       | Static public IP address                                          |
| **User Data**        | Bootstrap script that runs on first launch                        |

---

### Instance Types

EC2 instance types define the hardware your virtual server runs on. The naming convention is:

```
t3.micro
│  │
│  └── Size (nano, micro, small, medium, large, xlarge, 2xlarge...)
└───── Family (t=burstable, m=general, c=compute, r=memory...)
```

#### Instance Families Explained

| Family    | Optimised For      | Use Case                              | Examples           |
| --------- | ------------------ | ------------------------------------- | ------------------ |
| **t2/t3** | General + Burstable| Dev/test, small websites, microservices| `t3.micro`, `t3.small`|
| **m5/m6** | General Purpose    | Web servers, app servers, databases   | `m5.large`, `m6i.xlarge`|
| **c5/c6** | Compute            | Batch processing, gaming, ML inference| `c5.xlarge`, `c6i.2xlarge`|
| **r5/r6** | Memory             | In-memory caches, real-time analytics | `r5.large`, `r6i.xlarge`|
| **i3/i4** | Storage            | Data warehousing, distributed file systems| `i3.large`       |
| **p4/p5** | GPU (Accelerated)  | Machine learning training, video processing| `p4d.24xlarge`  |
| **g5**    | Graphics           | Graphics-intensive, game streaming    | `g5.xlarge`        |

#### Free Tier Instance

- **`t2.micro`** (or `t3.micro` in some regions) — 750 hours/month free for 12 months
- 1 vCPU, 1 GB RAM
- Perfect for learning and small experiments

#### Burstable Instances (T-series)

T-series instances accumulate **CPU credits** when idle and spend them when busy. Great for workloads that don't need constant CPU.

```
Idle period → Accumulate credits ☁️☁️☁️☁️
Burst period → Spend credits 🔥🔥🔥🔥
Credits depleted → Baseline performance only 📉
```

---

### AMIs (Amazon Machine Images)

An AMI is a template that contains the OS, software, and configuration for your instance.

#### Common AMIs

| AMI                     | Description                       | User          |
| ----------------------- | --------------------------------- | ------------- |
| **Amazon Linux 2023**   | AWS's own Linux (CentOS-based)    | `ec2-user`    |
| **Ubuntu**              | Popular Debian-based Linux        | `ubuntu`      |
| **Red Hat Enterprise**  | Enterprise Linux                  | `ec2-user`    |
| **Windows Server**      | Microsoft Windows                 | `Administrator`|
| **SUSE Linux**          | Enterprise Linux                  | `ec2-user`    |

```bash
# Search for Amazon Linux 2023 AMIs
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*" "Name=architecture,Values=x86_64" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].[ImageId, Name]' \
  --output table

# Search for Ubuntu AMIs
aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04*" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].[ImageId, Name]' \
  --output table
```

You can also create **custom AMIs** from your configured instances to use as templates.

---

### Hands-On: Launch an EC2 Instance

#### Via the Console (Step-by-Step)

1. Go to **EC2 Console** → **Launch Instance**
2. **Name**: `MyFirstInstance`
3. **AMI**: Amazon Linux 2023 (free tier eligible)
4. **Instance Type**: `t2.micro` (free tier eligible)
5. **Key Pair**: Create a new key pair → Download the `.pem` file
6. **Network Settings**:
   - VPC: Default VPC
   - Subnet: Any
   - Auto-assign Public IP: **Enable**
   - Security Group: Create new → Allow SSH (port 22) from your IP
7. **Storage**: 8 GB gp3 (default, free tier eligible)
8. Click **Launch Instance**

#### Via the CLI

```bash
# ===== Step 1: Create a Key Pair =====
# This creates an SSH key pair for connecting to your instance
aws ec2 create-key-pair \
  --key-name my-key \
  --query 'KeyMaterial' \
  --output text > my-key.pem

# Set correct permissions on the key file
# Linux/Mac:
chmod 400 my-key.pem
# Windows PowerShell:
# icacls my-key.pem /inheritance:r /grant:r "$($env:USERNAME):(R)"

# ===== Step 2: Create a Security Group =====
# This acts as a firewall for your instance
aws ec2 create-security-group \
  --group-name my-web-sg \
  --description "Security group for my first instance"

# Allow SSH access (port 22) from anywhere (for learning only!)
# In production, restrict to your IP: --cidr YOUR_IP/32
aws ec2 authorize-security-group-ingress \
  --group-name my-web-sg \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

# Allow HTTP access (port 80) from anywhere
aws ec2 authorize-security-group-ingress \
  --group-name my-web-sg \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# ===== Step 3: Launch the Instance =====
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t2.micro \
  --key-name my-key \
  --security-groups my-web-sg \
  --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=MyFirstInstance},{Key=Environment,Value=Learning}]'

# ===== Step 4: Check the Instance Status =====
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=MyFirstInstance" \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PublicIpAddress,InstanceType]' \
  --output table

# Wait for the instance to be "running" (may take 30-60 seconds)
aws ec2 wait instance-running \
  --filters "Name=tag:Name,Values=MyFirstInstance"
echo "Instance is now running!"
```

> 💡 **Note**: Replace `ami-0c55b159cbfafe1f0` with the actual AMI ID for your region. AMI IDs differ by region.

---

### Connecting to Your EC2 Instance

#### Via SSH (Linux/Mac)

```bash
# Get the public IP of your instance
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=MyFirstInstance" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text

# SSH into the instance
chmod 400 my-key.pem
ssh -i my-key.pem ec2-user@<public-ip>

# Example:
ssh -i my-key.pem ec2-user@13.233.45.67
```

#### Via SSH (Windows PowerShell)

```powershell
# SSH is built into Windows 10+
ssh -i my-key.pem ec2-user@<public-ip>
```

#### Via EC2 Instance Connect (Browser)

1. Go to **EC2 Console** → **Instances** → Select your instance
2. Click **"Connect"** button
3. Choose **"EC2 Instance Connect"** tab
4. Click **"Connect"** — opens a terminal in your browser!

#### Once Connected

```bash
# You're now inside your EC2 instance!
whoami          # ec2-user
hostname        # ip-172-31-xx-xx
uname -a        # Linux kernel info
cat /etc/os-release  # OS info

# Install a web server
sudo yum update -y
sudo yum install -y httpd
sudo systemctl start httpd
sudo systemctl enable httpd

# Create a simple web page
echo "<h1>Hello from EC2!</h1>" | sudo tee /var/www/html/index.html

# Now visit http://<public-ip> in your browser!
```

---

### Security Groups

**Security Groups** act as a virtual firewall for your EC2 instances. They control inbound (incoming) and outbound (outgoing) traffic.

#### Key Characteristics

- **Stateful** — If you allow inbound traffic, the response is automatically allowed out
- **Allow rules only** — You can only create ALLOW rules (no DENY rules)
- **Default** — All inbound blocked, all outbound allowed
- Applied at the **instance level** (not subnet level)

#### Common Rules

| Type   | Protocol | Port  | Source          | Use Case                    |
| ------ | -------- | ----- | --------------- | --------------------------- |
| SSH    | TCP      | 22    | Your IP/32      | Remote access               |
| HTTP   | TCP      | 80    | 0.0.0.0/0       | Web traffic                 |
| HTTPS  | TCP      | 443   | 0.0.0.0/0       | Secure web traffic          |
| MySQL  | TCP      | 3306  | sg-xxxx          | Database from app SG only   |
| Custom | TCP      | 8080  | 10.0.0.0/16     | App server from VPC only    |

```bash
# List all security groups
aws ec2 describe-security-groups --output table

# Describe a specific security group
aws ec2 describe-security-groups --group-names my-web-sg

# Add a rule (allow HTTPS)
aws ec2 authorize-security-group-ingress \
  --group-name my-web-sg \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Remove a rule
aws ec2 revoke-security-group-ingress \
  --group-name my-web-sg \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

---

### Elastic IPs

An **Elastic IP** is a static, public IPv4 address. By default, EC2 instances get a new public IP every time they stop/start.

```bash
# Allocate an Elastic IP
aws ec2 allocate-address
# Note the AllocationId from the output

# Associate it with your instance
aws ec2 associate-address \
  --instance-id i-0abcdef1234567890 \
  --allocation-id eipalloc-0abcdef1234567890

# Release (free up) an Elastic IP
aws ec2 release-address --allocation-id eipalloc-0abcdef1234567890
```

> ⚠️ **Elastic IPs are free when attached to a running instance. You are CHARGED if the IP is allocated but NOT attached to a running instance.**

---

### Instance Lifecycle

```
         launch
  ┌──────────────────┐
  │                  ▼
  │              PENDING ──────► RUNNING ◄─────── start
  │                                │
  │                          stop  │  terminate
  │                                ▼
  │                           STOPPING
  │                                │
  │                    ┌───────────┴───────────┐
  │                    ▼                       ▼
  │                 STOPPED              SHUTTING-DOWN
  │                    │                       │
  │               start│                       ▼
  │                    │                  TERMINATED
  └────────────────────┘
```

#### What You Pay For

| State          | Compute Cost | EBS Cost | Elastic IP Cost |
| -------------- | ------------ | -------- | --------------- |
| **Running**    | ✅ Yes       | ✅ Yes   | ❌ Free         |
| **Stopped**    | ❌ No        | ✅ Yes   | ✅ Yes (charged)|
| **Terminated** | ❌ No        | ❌ No*   | N/A             |

*EBS volumes are deleted by default on termination (configurable).

```bash
# Stop an instance (keeps data, stops compute charges)
aws ec2 stop-instances --instance-ids i-0abcdef1234567890

# Start a stopped instance
aws ec2 start-instances --instance-ids i-0abcdef1234567890

# Terminate an instance (permanently destroys it)
aws ec2 terminate-instances --instance-ids i-0abcdef1234567890

# Reboot an instance (like pressing restart)
aws ec2 reboot-instances --instance-ids i-0abcdef1234567890
```

---

### User Data (Bootstrap Scripts)

**User Data** is a script that runs automatically when an EC2 instance launches for the first time. Great for automated setup.

```bash
# Launch an instance with User Data
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t2.micro \
  --key-name my-key \
  --security-groups my-web-sg \
  --user-data '#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hello from $(hostname)</h1>" > /var/www/html/index.html'
```

Or from a file:

```bash
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t2.micro \
  --key-name my-key \
  --user-data file://bootstrap.sh
```

---

### EC2 Pricing Models

| Model              | Discount    | Commitment      | Best For                          |
| ------------------ | ----------- | --------------- | --------------------------------- |
| **On-Demand**      | None        | None            | Short-term, unpredictable workloads|
| **Reserved**       | Up to 72%   | 1 or 3 years    | Steady-state, predictable usage   |
| **Spot**           | Up to 90%   | None (can be interrupted)| Fault-tolerant, batch jobs |
| **Savings Plans**  | Up to 72%   | 1 or 3 years    | Flexible commitment across services|
| **Dedicated Hosts**| Varies      | On-demand or reserved| Compliance, licensing requirements|

> 💡 **For learning**: Always use **On-Demand** with **t2.micro** (free tier).

---

## AWS Lambda (Serverless)

**Lambda** lets you run code without provisioning or managing servers. You upload your code, and AWS handles everything else.

### How Lambda Works

```
1. You write a function (e.g., Python, Node.js, Java)
2. You upload it to Lambda
3. You define a TRIGGER (what event invokes the function)
4. When the event occurs:
   ├── AWS spins up a container
   ├── Runs your code
   ├── Returns the result
   └── Shuts down the container
5. You pay ONLY for the compute time used
```

### Lambda Key Facts

| Detail                  | Info                                           |
| ----------------------- | ---------------------------------------------- |
| **Max execution time**  | 15 minutes                                     |
| **Max memory**          | 10,240 MB (10 GB)                              |
| **Max package size**    | 50 MB zipped, 250 MB unzipped                  |
| **Supported runtimes**  | Python, Node.js, Java, Go, .NET, Ruby, Custom  |
| **Free tier**           | 1 million requests/month (always free)          |
| **Pricing**             | $0.20 per 1M requests + $0.0000166667/GB-second|

### Hands-On: Create a Lambda Function

#### Step 1: Write the Function Code

Create a file called `index.py`:

```python
import json

def handler(event, context):
    """
    Simple Lambda function that returns a greeting.
    
    - event: The input data (JSON)
    - context: Runtime information (request ID, time remaining, etc.)
    """
    name = event.get('name', 'World')
    
    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': f'Hello, {name}! Welcome to AWS Lambda!',
            'requestId': context.aws_request_id
        })
    }
```

#### Step 2: Package and Deploy

```bash
# Zip the function code
zip function.zip index.py

# Create the Lambda function
aws lambda create-function \
  --function-name HelloFunction \
  --runtime python3.12 \
  --handler index.handler \
  --zip-file fileb://function.zip \
  --role arn:aws:iam::123456789012:role/lambda-execution-role \
  --description "My first Lambda function" \
  --timeout 30 \
  --memory-size 128

# The --role must be an IAM role with Lambda execution permissions
# See the "Creating a Lambda Execution Role" section below
```

#### Step 3: Invoke the Function

```bash
# Invoke with default event (empty)
aws lambda invoke \
  --function-name HelloFunction \
  --payload '{}' \
  output.json
cat output.json

# Invoke with a custom event
aws lambda invoke \
  --function-name HelloFunction \
  --payload '{"name": "Leela"}' \
  output.json
cat output.json
# Output: {"statusCode": 200, "body": "{\"message\": \"Hello, Leela! Welcome to AWS Lambda!\", ...}"}
```

#### Step 4: Update the Function

```bash
# Update the code
zip function.zip index.py
aws lambda update-function-code \
  --function-name HelloFunction \
  --zip-file fileb://function.zip

# Update configuration (e.g., increase memory)
aws lambda update-function-configuration \
  --function-name HelloFunction \
  --memory-size 256 \
  --timeout 60
```

### Creating a Lambda Execution Role

Lambda needs an IAM role to run. See [[AWS IAM]] for more on roles.

```bash
# Create the trust policy (trust-policy.json):
# {
#   "Version": "2012-10-17",
#   "Statement": [{
#     "Effect": "Allow",
#     "Principal": { "Service": "lambda.amazonaws.com" },
#     "Action": "sts:AssumeRole"
#   }]
# }

aws iam create-role \
  --role-name lambda-execution-role \
  --assume-role-policy-document file://trust-policy.json

# Attach basic logging permissions
aws iam attach-role-policy \
  --role-name lambda-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

### Lambda Triggers

Lambda functions are invoked by **events** from other AWS services:

| Trigger                | Use Case                                        |
| ---------------------- | ----------------------------------------------- |
| **API Gateway**        | Build REST/HTTP APIs                            |
| **S3**                 | Process files when uploaded (resize images, etc.)|
| **DynamoDB Streams**   | React to database changes                       |
| **SQS**               | Process messages from a queue                   |
| **SNS**               | React to notifications                          |
| **CloudWatch Events**  | Scheduled tasks (cron jobs)                     |
| **CloudWatch Logs**    | Process log data                                |
| **Kinesis**            | Process streaming data                          |

#### Example: Trigger Lambda on S3 Upload

```bash
# Add permission for S3 to invoke the function
aws lambda add-permission \
  --function-name HelloFunction \
  --statement-id s3-trigger \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::my-bucket

# Configure S3 bucket to trigger Lambda (via console is easier)
# Or use the S3 put-bucket-notification-configuration API
```

### Managing Lambda Functions

```bash
# List all functions
aws lambda list-functions --output table

# Get function details
aws lambda get-function --function-name HelloFunction

# View recent invocation logs
aws logs filter-log-events \
  --log-group-name /aws/lambda/HelloFunction \
  --start-time $(date -d '1 hour ago' +%s)000

# Delete a function
aws lambda delete-function --function-name HelloFunction
```

### Lambda vs EC2 — When to Use Which

| Factor             | EC2                              | Lambda                          |
| ------------------ | -------------------------------- | ------------------------------- |
| **Execution time** | Unlimited                        | Max 15 minutes                  |
| **Control**        | Full OS access                   | Code only                       |
| **Scaling**        | Manual or Auto Scaling Groups    | Automatic (instant)             |
| **Pricing**        | Pay per hour (even if idle)      | Pay per request + compute time  |
| **Cold starts**    | None (always running)            | Yes (first invocation is slower)|
| **State**          | Persistent (files on disk)       | Stateless                       |
| **Best for**       | Long-running, stateful apps      | Short, event-driven tasks       |

---

## Container Services

### What are Containers?

Containers package your application code, dependencies, and runtime into a single, portable unit. Think of it as a lightweight virtual machine that starts in seconds.

```
Traditional VM:               Container:
┌─────────────────┐          ┌─────────────────┐
│   Application   │          │   Application   │
├─────────────────┤          ├─────────────────┤
│   Dependencies  │          │   Dependencies  │
├─────────────────┤          ├─────────────────┤
│   Guest OS      │          │   Container     │
├─────────────────┤          │   Runtime       │
│   Hypervisor    │          ├─────────────────┤
├─────────────────┤          │   Host OS       │
│   Host OS       │          ├─────────────────┤
├─────────────────┤          │   Hardware      │
│   Hardware      │          └─────────────────┘
└─────────────────┘
     Heavy (GBs)                 Light (MBs)
     Minutes to start            Seconds to start
```

---

### Amazon ECS (Elastic Container Service)

**ECS** is AWS's own container orchestration service. It manages running, scaling, and monitoring Docker containers.

#### ECS Concepts

| Concept          | Description                                                |
| ---------------- | ---------------------------------------------------------- |
| **Cluster**      | Logical grouping of tasks/services                         |
| **Task Definition**| Blueprint for your container (image, CPU, memory, ports) |
| **Task**         | A running instance of a Task Definition                    |
| **Service**      | Maintains a desired number of tasks (auto-replaces failed) |
| **Container Instance**| EC2 instance running the ECS agent (if using EC2 launch type)|

#### ECS Launch Types

1. **EC2 Launch Type** — Containers run on EC2 instances YOU manage
2. **Fargate Launch Type** — Containers run on serverless infrastructure (AWS manages)

```bash
# Create an ECS cluster
aws ecs create-cluster --cluster-name my-cluster

# Register a task definition (requires a JSON file)
aws ecs register-task-definition --cli-input-json file://task-def.json

# Run a task
aws ecs run-task \
  --cluster my-cluster \
  --task-definition my-app:1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx],assignPublicIp=ENABLED}"

# Create a service (maintains N running copies)
aws ecs create-service \
  --cluster my-cluster \
  --service-name my-service \
  --task-definition my-app:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx],assignPublicIp=ENABLED}"

# List services
aws ecs list-services --cluster my-cluster

# Update a service (e.g., scale to 3 tasks)
aws ecs update-service --cluster my-cluster --service my-service --desired-count 3
```

---

### Amazon EKS (Elastic Kubernetes Service)

**EKS** is AWS's managed Kubernetes service. If your team already uses Kubernetes, EKS lets you run it on AWS without managing the control plane.

#### Key Facts

| Detail               | Info                                            |
| -------------------- | ----------------------------------------------- |
| **What it manages**  | Kubernetes control plane (API server, etcd)     |
| **What you manage**  | Worker nodes (or use Fargate for serverless)    |
| **Pricing**          | $0.10/hour for the control plane + worker costs |
| **Best for**         | Teams with Kubernetes expertise                 |

```bash
# Create an EKS cluster (using eksctl — the recommended tool)
# Install eksctl first: https://eksctl.io
eksctl create cluster \
  --name my-cluster \
  --region ap-south-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2

# Configure kubectl to use the new cluster
aws eks update-kubeconfig --name my-cluster --region ap-south-1

# Verify connection
kubectl get nodes
kubectl get pods --all-namespaces
```

---

### AWS Fargate

**Fargate** is a serverless compute engine for containers. It works with both ECS and EKS, removing the need to manage EC2 instances.

#### Fargate vs EC2 for Containers

| Aspect           | Fargate (Serverless)              | EC2 (Self-managed)               |
| ---------------- | --------------------------------- | -------------------------------- |
| **Server management**| AWS manages everything       | You manage EC2 instances         |
| **Scaling**      | Automatic per-task scaling        | You configure Auto Scaling       |
| **Pricing**      | Per task (vCPU + memory per second)| Per EC2 instance (hourly)       |
| **Patching**     | AWS handles it                    | You handle OS patches            |
| **Best for**     | Variable workloads, simplicity    | Predictable, cost-sensitive      |

### When to Use Each Container Service

```
Decision Tree:

Need containers?
├── Yes → Do you know Kubernetes?
│         ├── Yes → EKS
│         │         ├── Want to manage nodes? → EKS on EC2
│         │         └── Want serverless? → EKS on Fargate
│         └── No → ECS
│                   ├── Want to manage nodes? → ECS on EC2
│                   └── Want serverless? → ECS on Fargate
└── No → Consider EC2 or Lambda
```

---

## Amazon ECR (Elastic Container Registry)

**ECR** is AWS's managed Docker container registry. Store, manage, and deploy Docker images.

```bash
# Create a repository
aws ecr create-repository --repository-name my-app

# Authenticate Docker to ECR
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.ap-south-1.amazonaws.com

# Tag and push an image
docker tag my-app:latest 123456789012.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest
docker push 123456789012.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest

# List images
aws ecr list-images --repository-name my-app
```

---

## Compute Comparison Table

| Feature                | EC2              | Lambda           | ECS (Fargate)    | EKS              |
| ---------------------- | ---------------- | ---------------- | ---------------- | ---------------- |
| **Type**               | Virtual Machine  | Serverless Function| Serverless Container| Managed Kubernetes|
| **Max Run Time**       | Unlimited        | 15 minutes       | Unlimited        | Unlimited        |
| **Scaling**            | Auto Scaling Group| Automatic       | Service auto-scale| HPA/Cluster Autoscaler|
| **Min Cost**           | ~$8/month (t3.micro)| Free tier     | ~$10/month       | $72/month (control plane)|
| **Cold Start**         | Minutes          | Milliseconds-seconds| Seconds       | Minutes          |
| **OS Access**          | Full             | None             | Container only   | Container only   |
| **Stateful**           | Yes              | No               | Possible (EBS)   | Possible (EBS)   |
| **Pricing Model**      | Per hour         | Per request + time| Per vCPU + memory| Control plane + workers|
| **Best For**           | Legacy apps, full control| Event-driven, APIs| Microservices | K8s workloads   |
| **Learning Curve**     | Medium           | Low              | Medium           | High             |

---

## Hands-On: Clean Up Resources

Always clean up to avoid unexpected charges:

```bash
# Terminate EC2 instances
aws ec2 terminate-instances --instance-ids i-0abcdef1234567890

# Delete Lambda function
aws lambda delete-function --function-name HelloFunction

# Delete security groups (after terminating instances)
aws ec2 delete-security-group --group-name my-web-sg

# Delete key pairs
aws ec2 delete-key-pair --key-name my-key

# Release Elastic IPs
aws ec2 release-address --allocation-id eipalloc-xxx

# Delete ECS cluster (after deleting services and tasks)
aws ecs delete-cluster --cluster my-cluster
```

---

## What's Next?

- [[AWS Storage]] — Learn about S3, EBS, and EFS for storing your data
- [[AWS Networking]] — Understand VPCs, subnets, and load balancers
- [[AWS Deployment]] — Deploy applications with CI/CD pipelines
- [[AWS IAM]] — Manage roles for EC2 and Lambda
- [[AWS Getting Started]] — Back to basics

---

## Related Notes

- [[Cloud Fundamentals]] — IaaS, PaaS, SaaS and how compute fits in
- [[AWS Getting Started]] — Setting up your account and CLI
- [[AWS Networking]] — VPCs, subnets, security groups in depth
- [[AWS Deployment]] — CI/CD with CodePipeline, CodeBuild, CodeDeploy
- [[AWS IAM]] — Roles and permissions for compute services
