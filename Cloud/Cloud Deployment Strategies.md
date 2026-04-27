# Cloud Deployment Strategies

> **Goal**: Understand how code goes from your laptop to the cloud where real users can access it — and the different strategies for doing it safely.

---

## What is Deployment?

**Deployment** is the process of taking code you've written on your computer and putting it somewhere that users can actually use it.

Think of it like this:
- You write a web application on your laptop
- It works perfectly at `localhost:3000`
- But **nobody else in the world can access it**
- Deployment = putting that app on a cloud server so anyone with the URL can use it

### What Happens During Deployment?

```
Your Code (Laptop)
    ↓
Build/Compile (create runnable version)
    ↓
Package (zip, container image, etc.)
    ↓
Transfer to Server (upload to cloud)
    ↓
Configure (environment variables, database connections)
    ↓
Start the Application
    ↓
Route Traffic (point your domain to the new server)
    ↓
Users Can Access It! 🎉
```

> **Key Insight**: Deployment isn't just copying files — it's a coordinated process that needs to happen without breaking things for existing users.

---

## Deployment Environments

Before your code reaches real users, it should pass through multiple environments. Think of these as checkpoints.

### The Three Main Environments

| Environment | Purpose | Who Uses It | Data |
|-------------|---------|-------------|------|
| **Development (Dev)** | Developers test their changes | Developers only | Fake/test data |
| **Staging** | Final testing before production | QA team, stakeholders | Copy of production data (anonymized) |
| **Production (Prod)** | Real users use the application | Everyone (customers) | Real data |

### Why Separate Environments?

Imagine you're building a banking app:

1. **Development** — You're testing a new "Transfer Money" feature. You don't want to accidentally transfer real money while testing!
2. **Staging** — QA team tests the feature with realistic data. Catches bugs that dev missed.
3. **Production** — Feature is proven safe. Now real customers can use it.

```
Developer's Laptop → Dev Environment → Staging Environment → Production
     "It works!"      "Tests pass!"    "QA approved!"      "Users love it!"
```

> **Rule of Thumb**: Never test in production. Your staging environment should be as close to production as possible — same configurations, similar data, same infrastructure size.

### Additional Environments You Might See

- **QA Environment** — Dedicated for quality assurance testing
- **UAT (User Acceptance Testing)** — Business stakeholders verify features
- **Pre-production** — A final mirror of production for last checks
- **Sandbox** — Experimental environment for trying new ideas

---

## Deployment Strategies

When you deploy new code, you need a **strategy** — how do you replace the old version with the new one without breaking things?

### 1. Recreate (All-at-Once) Deployment

**How it works**: Stop everything running the old version → Deploy the new version → Start everything again.

```
Time →

Old Version:  [████████████]
                              ↓ DOWNTIME ↓
New Version:                  [████████████████████]

Users see:    [  Working   ] [ NOTHING ] [  Working again  ]
```

**Pros:**
- ✅ Simplest strategy — easy to understand and implement
- ✅ No version conflicts (only one version runs at a time)
- ✅ Clean deployment — fresh start every time

**Cons:**
- ❌ **Downtime** — users can't access the app during deployment
- ❌ Risky — if new version has bugs, everyone is affected
- ❌ Rollback requires another full deployment

**When to use**: Internal tools, development environments, applications where brief downtime is acceptable (e.g., deploying at 2 AM on a Sunday).

---

### 2. Rolling Deployment

**How it works**: Gradually replace instances one at a time (or in batches). If you have 4 servers, update server 1, then server 2, then server 3, then server 4.

```
Time →

Server 1: [Old Old Old][New New New New New New New]
Server 2: [Old Old Old Old Old][New New New New New]
Server 3: [Old Old Old Old Old Old Old][New New New]
Server 4: [Old Old Old Old Old Old Old Old Old][New]

Users see: Always at least some servers running ✅
```

**Pros:**
- ✅ Zero downtime — some servers always running
- ✅ Gradual rollout — can catch issues early
- ✅ Uses existing infrastructure (no extra servers needed)

**Cons:**
- ❌ Two versions run simultaneously (can cause compatibility issues)
- ❌ **Slow rollback** — have to roll back server by server
- ❌ Takes longer than all-at-once

**When to use**: Standard web applications where you can handle two versions running briefly.

---

### 3. Blue-Green Deployment

**How it works**: Maintain TWO identical environments. "Blue" runs the current version, "Green" gets the new version. Once Green is ready, switch ALL traffic from Blue to Green instantly.

```
            Load Balancer
                 |
        ┌────────┼────────┐
        ▼                  ▼
   [Blue Env]         [Green Env]
   Old Version        New Version
   (currently live)   (being prepared)

Step 1: Deploy new version to Green
Step 2: Test Green thoroughly
Step 3: Switch traffic: Blue → Green (instant!)
Step 4: Green is now live, Blue is standby
```

**Pros:**
- ✅ **Zero downtime** — traffic switch is instant
- ✅ **Instant rollback** — just switch back to Blue
- ✅ Can test the new version in Green before switching
- ✅ Very safe deployment

**Cons:**
- ❌ **Expensive** — need to pay for TWO full environments
- ❌ Database migrations can be tricky (both environments share the DB)
- ❌ More complex infrastructure setup

**When to use**: Mission-critical applications (banking, e-commerce) where downtime = lost money.

---

### 4. Canary Deployment

**How it works**: Deploy the new version to a **small percentage** of users first (like 5%). Monitor for issues. If everything's good, gradually increase to 100%.

```
Phase 1:  [95% → Old Version] [5% → New Version]  ← "Canary" group
Phase 2:  [75% → Old Version] [25% → New Version]  ← Looks good!
Phase 3:  [50% → Old Version] [50% → New Version]  ← Still good!
Phase 4:  [0%  → Old Version] [100% → New Version] ← Full rollout!
```

> **Why "Canary"?** Coal miners used to bring canaries into mines. If the canary stopped singing, it meant dangerous gas — get out! Similarly, your canary users are the first to encounter any problems.

**Pros:**
- ✅ **Safest strategy** — only a small % of users affected if there's a bug
- ✅ Real-world testing with real users and real traffic
- ✅ Easy rollback — just route the 5% back to old version
- ✅ Can monitor performance differences between old and new

**Cons:**
- ❌ Slower rollout (have to wait and monitor at each phase)
- ❌ Requires sophisticated traffic routing (load balancer rules)
- ❌ Two versions running simultaneously

**When to use**: Large-scale applications with millions of users. Any app where a bad deployment could cause significant damage.

---

### 5. A/B Testing Deployment

**How it works**: Similar to Canary, but the routing is based on **user segments**, not random percentages. You're testing which version performs better.

```
Users in Segment A → Version A (blue button)
Users in Segment B → Version B (green button)

Measure: Which button gets more clicks?
```

**Pros:**
- ✅ Data-driven decisions
- ✅ Can test new features with specific user groups
- ✅ Combines deployment with product experimentation

**Cons:**
- ❌ Complex routing logic needed
- ❌ Need analytics to measure results
- ❌ Not purely a deployment strategy — more of a product strategy

**When to use**: When you want to test feature variations (UI changes, pricing, workflows).

---

### Strategy Comparison Table

| Strategy | Downtime | Rollback Speed | Cost | Risk | Complexity |
|----------|----------|----------------|------|------|------------|
| **Recreate** | ❌ Yes | Slow (redeploy) | Low | High | Very Low |
| **Rolling** | ✅ None | Slow (gradual) | Low | Medium | Low |
| **Blue-Green** | ✅ None | ⚡ Instant | High (2x infra) | Low | Medium |
| **Canary** | ✅ None | ⚡ Fast | Medium | Very Low | High |
| **A/B Testing** | ✅ None | ⚡ Fast | Medium | Low | High |

---

## Infrastructure as Code (IaC)

### The Problem IaC Solves

**Without IaC (The Old Way)**:
1. Log into the AWS/Azure console
2. Click "Create VM"
3. Choose an OS, pick a size, configure networking...
4. Click through 15 screens of options
5. Hope you remember all the settings for next time
6. Your colleague asks "how did you set up the server?" — You: "Uh... I clicked stuff"

**With IaC (The New Way)**:
1. Write a file describing what you want
2. Run a command
3. Infrastructure is created exactly as described
4. The file IS your documentation
5. Store it in Git — now you have history of every change

### Declarative vs Imperative

| Approach | Description | Example |
|----------|-------------|---------|
| **Declarative** | "I want 3 servers with 4GB RAM" — the tool figures out HOW | Terraform, CloudFormation |
| **Imperative** | "Create server 1, then create server 2, then create server 3" — YOU specify HOW | Scripts, AWS CLI commands |

**Declarative example (Terraform)**:
```hcl
# I want this to exist — Terraform figures out the steps
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  count         = 3

  tags = {
    Name = "web-server"
  }
}
```

**Imperative example (AWS CLI script)**:
```bash
# I specify every step
aws ec2 run-instances --image-id ami-0c55b159cbfafe1f0 --instance-type t2.micro
aws ec2 run-instances --image-id ami-0c55b159cbfafe1f0 --instance-type t2.micro
aws ec2 run-instances --image-id ami-0c55b159cbfafe1f0 --instance-type t2.micro
```

### Popular IaC Tools

| Tool | Provider | Language | Style |
|------|----------|----------|-------|
| **AWS CloudFormation** | AWS only | JSON/YAML | Declarative |
| **Azure ARM Templates** | Azure only | JSON | Declarative |
| **Azure Bicep** | Azure only | Bicep (simpler than ARM) | Declarative |
| **Terraform** | Multi-cloud | HCL | Declarative |
| **Pulumi** | Multi-cloud | Python, JS, Go, etc. | Both |
| **Ansible** | Any | YAML | Imperative |

### Why Version Control Your Infrastructure?

```
git log --oneline

a3f2d1c  Add load balancer to production
b7e9c3a  Increase web servers from 2 to 4
c1d4e5f  Add Redis cache cluster
d8f2a1b  Initial infrastructure setup
```

Now you can:
- See **who** changed infrastructure and **when**
- **Rollback** to a previous infrastructure state
- **Review** changes before applying (pull requests for infrastructure!)
- **Reproduce** the exact same setup in another region or account

---

## Containerization Basics

### What is a Container?

A **container** is a lightweight, standalone package that includes everything needed to run a piece of software:
- Your application code
- Runtime (Node.js, Python, Java, etc.)
- Libraries and dependencies
- Configuration files

> **Analogy**: Think of a shipping container. No matter what's inside (furniture, electronics, food), the container is a standard size and shape. Any ship, truck, or crane can handle it. Software containers work the same way — they package your app in a standard format that can run anywhere.

### Container vs Virtual Machine

| Feature | Container | Virtual Machine |
|---------|-----------|----------------|
| **Size** | Megabytes (50-500 MB) | Gigabytes (5-20 GB) |
| **Startup Time** | Seconds | Minutes |
| **OS** | Shares host OS kernel | Full OS per VM |
| **Isolation** | Process-level | Hardware-level |
| **Resource Usage** | Lightweight | Heavy |
| **Portability** | Extremely portable | Less portable |
| **Use Case** | Microservices, CI/CD | Legacy apps, different OS needs |

```
┌─────────────────────────────────┐
│         Virtual Machines         │
│  ┌─────────┐  ┌─────────┐      │
│  │  App A   │  │  App B   │     │
│  │  Libs    │  │  Libs    │     │
│  │  Full OS │  │  Full OS │     │ ← Each VM has its own OS (heavy!)
│  └─────────┘  └─────────┘      │
│       Hypervisor                 │
│       Host OS                    │
│       Hardware                   │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│          Containers              │
│  ┌─────────┐  ┌─────────┐      │
│  │  App A   │  │  App B   │     │
│  │  Libs    │  │  Libs    │     │ ← Containers share the host OS (light!)
│  └─────────┘  └─────────┘      │
│       Container Runtime (Docker) │
│       Host OS                    │
│       Hardware                   │
└─────────────────────────────────┘
```

### Docker Basics

**Docker** is the most popular container platform. Key concepts:

- **Docker Image** — A blueprint/template for a container. Like a class in programming.
- **Docker Container** — A running instance of an image. Like an object created from a class.
- **Dockerfile** — A text file with instructions to build an image.
- **Docker Hub** — A public registry (like GitHub for Docker images) where you can find and share images.

**Example Dockerfile** for a simple Node.js app:

```dockerfile
# Start from an official Node.js base image
FROM node:18-alpine

# Set working directory inside the container
WORKDIR /app

# Copy package files and install dependencies
COPY package*.json ./
RUN npm install

# Copy the rest of your application code
COPY . .

# Expose port 3000 so traffic can reach the app
EXPOSE 3000

# Command to run when container starts
CMD ["node", "server.js"]
```

**Common Docker Commands**:

```bash
# Build an image from a Dockerfile
docker build -t my-app:v1 .

# Run a container from the image
docker run -p 3000:3000 my-app:v1

# List running containers
docker ps

# Stop a container
docker stop <container-id>

# Pull an image from Docker Hub
docker pull nginx:latest
```

### Container Orchestration with Kubernetes

When you have **hundreds of containers**, you need a system to manage them. That's **Kubernetes** (often abbreviated as **K8s**).

**Key Kubernetes Concepts**:

| Concept | What It Is | Analogy |
|---------|------------|---------|
| **Pod** | Smallest unit — one or more containers | A single apartment |
| **Service** | Stable network endpoint for pods | A phone number that never changes |
| **Deployment** | Manages pod replicas and updates | A property manager for many apartments |
| **Node** | A physical/virtual machine running pods | An apartment building |
| **Cluster** | A group of nodes | A neighborhood of buildings |

**Example Kubernetes Deployment** (YAML):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web-app
spec:
  replicas: 3          # Run 3 copies of the app
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: my-app:v1
        ports:
        - containerPort: 3000
        resources:
          limits:
            memory: "256Mi"
            cpu: "500m"
```

### Cloud Container Services

| Service | Provider | Type | Description |
|---------|----------|------|-------------|
| **ECS** | AWS | Container orchestration | AWS's own container management |
| **EKS** | AWS | Managed Kubernetes | AWS runs Kubernetes for you |
| **AKS** | Azure | Managed Kubernetes | Azure runs Kubernetes for you |
| **ACI** | Azure | Serverless containers | Run containers without managing servers |
| **Fargate** | AWS | Serverless containers | Run ECS/EKS without managing servers |

---

## CI/CD (Continuous Integration / Continuous Deployment)

### What is CI (Continuous Integration)?

**CI** = Every time a developer pushes code, it's automatically **built** and **tested**.

```
Developer pushes code to Git
         ↓
CI Server detects the change
         ↓
Automatically builds the code
         ↓
Automatically runs all tests
         ↓
Reports: ✅ Pass or ❌ Fail
```

**Why CI matters**:
- Catches bugs within minutes, not days
- Ensures all tests pass before code is merged
- Prevents "it works on my machine" problems
- Builds confidence that the codebase is always in a working state

### What is CD (Continuous Deployment)?

**CD** = After CI passes, automatically **deploy** to production (or staging).

There are actually two related terms:
- **Continuous Delivery** — Automatically prepare a release, but a human clicks "Deploy"
- **Continuous Deployment** — Fully automatic — code goes straight to production after tests pass

```
Continuous Delivery:    Code → Build → Test → [Ready to Deploy] → Human clicks "Deploy" → Production
Continuous Deployment:  Code → Build → Test → Automatically Deployed → Production
```

### CI/CD Pipeline Stages

A typical pipeline has these stages:

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  Source   │ → │  Build   │ → │   Test   │ → │  Stage   │ → │  Deploy  │
│          │   │          │   │          │   │          │   │          │
│ Git push │   │ Compile  │   │ Unit     │   │ Deploy   │   │ Deploy   │
│ PR merge │   │ Package  │   │ Integr.  │   │ to stag. │   │ to prod  │
│          │   │ Lint     │   │ Security │   │ Smoke    │   │ Monitor  │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
```

### CI/CD Tools

| Tool | Provider | Notes |
|------|----------|-------|
| **GitHub Actions** | GitHub | Built into GitHub, YAML-based workflows |
| **AWS CodePipeline** | AWS | Integrates with CodeBuild, CodeDeploy |
| **Azure DevOps Pipelines** | Azure | Full CI/CD platform |
| **Jenkins** | Open Source | Self-hosted, very flexible |
| **GitLab CI** | GitLab | Built into GitLab |
| **CircleCI** | Third-party | Cloud-based CI/CD |

### Example: GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]        # Trigger on push to main

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4          # Get the code

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci                        # Install packages

      - name: Run linter
        run: npm run lint                  # Check code style

      - name: Run tests
        run: npm test                      # Run all tests

      - name: Build
        run: npm run build                 # Create production build

  deploy:
    needs: build-and-test               # Only runs if build-and-test passes
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to AWS
        run: |
          echo "Deploying to production..."
          # aws deploy commands here
```

---

## Serverless Deployment

### What is Serverless?

**Serverless** doesn't mean "no servers" — it means **you don't manage the servers**. The cloud provider handles all the infrastructure. You just upload your code.

```
Traditional:    You manage: [Servers] [OS] [Runtime] [Scaling] [Patching] [Your Code]
Serverless:     You manage: [Your Code]   ← That's it!
```

### How Serverless Works

1. You write a **function** (small piece of code)
2. You define a **trigger** (what should invoke this function)
3. Upload it to a serverless platform
4. The platform runs your function when triggered
5. You pay only for the execution time (not idle time)

### Common Triggers

- **HTTP Request** — Someone visits a URL → function runs
- **File Upload** — A file is uploaded to storage → function processes it
- **Database Change** — A record is inserted → function sends a notification
- **Timer/Schedule** — Every day at midnight → function generates a report
- **Message Queue** — A message arrives → function processes it

### Serverless Platforms

| Service | Provider | Description |
|---------|----------|-------------|
| **AWS Lambda** | AWS | The original serverless compute |
| **Azure Functions** | Azure | Microsoft's serverless compute |
| **Google Cloud Functions** | GCP | Google's serverless compute |

### Serverless Pros and Cons

**Pros:**
- ✅ Zero server management
- ✅ Auto-scales to zero (no traffic = no cost)
- ✅ Pay per execution (can be very cheap)
- ✅ Built-in high availability

**Cons:**
- ❌ Cold starts (first invocation can be slow)
- ❌ Limited execution time (e.g., Lambda max 15 minutes)
- ❌ Vendor lock-in
- ❌ Harder to debug and test locally
- ❌ Not suitable for long-running processes

---

## Key Takeaways

1. **Deployment** = getting code from your laptop to the cloud
2. **Always use separate environments** (Dev → Staging → Production)
3. **Choose your strategy** based on your tolerance for risk and downtime
4. **Infrastructure as Code** — never click through consoles manually
5. **Containers** make your app portable and consistent
6. **CI/CD** automates the boring (and error-prone) deployment steps
7. **Serverless** lets you focus entirely on code, not infrastructure

---

## Related Notes

- [[AWS Deployment]] — AWS-specific deployment services (Elastic Beanstalk, CodeDeploy, etc.)
- [[Azure Deployment]] — Azure-specific deployment services (App Service, DevOps, etc.)
- [[Cloud Fundamentals]] — Core cloud concepts (regions, availability zones, etc.)
- [[Monitoring and Observability]] — How to monitor your deployments
- [[Cost Management]] — How deployment choices affect your cloud bill
