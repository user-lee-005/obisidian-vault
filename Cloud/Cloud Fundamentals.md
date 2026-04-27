# Cloud Fundamentals

> [!info] Purpose
> This is the foundational note for understanding cloud computing. Start here if you're completely new to the cloud. Everything else in this vault builds on these concepts.

---

## What is Cloud Computing?

### Simple Definition

Cloud computing is **using someone else's computers over the internet** — instead of buying and maintaining your own servers, you rent computing power, storage, and services from a cloud provider and pay only for what you use.

### The Renting vs Buying Analogy

Think of it like housing:

| Scenario | Traditional IT (On-Premise) | Cloud Computing |
|---|---|---|
| What you do | Buy a house, maintain it yourself | Rent an apartment |
| Upfront cost | Huge (buy servers, build data centres) | Minimal (pay monthly) |
| Maintenance | You fix everything | Landlord (provider) handles infrastructure |
| Scaling | Buy a bigger house (takes months) | Move to a bigger apartment (takes minutes) |
| Risk | Stuck with the house if needs change | Cancel the lease anytime |

### A Brief History

```
1960s  →  Mainframes: Giant shared computers. Users connected via terminals.
1990s  →  Colocation: Rent space in a data centre, bring your own servers.
2000s  →  VPS (Virtual Private Servers): Rent a virtual slice of a physical server.
2006   →  AWS launches EC2: The modern cloud era begins.
2008   →  Google App Engine launches (PaaS).
2010   →  Microsoft Azure goes live.
2010s+ →  Cloud becomes the default for startups and enterprises.
```

### Why Did Businesses Move to the Cloud?

- **No upfront capital expense** — No need to buy expensive hardware.
- **Pay-as-you-go pricing** — Only pay for what you actually use.
- **Speed and agility** — Spin up a server in seconds, not weeks.
- **Global reach** — Deploy to any region in the world.
- **Automatic scaling** — Handle traffic spikes without manual intervention.
- **Focus on business logic** — Stop worrying about hardware failures.
- **Built-in security and compliance** — Providers invest billions in security.

---

## Key Characteristics of Cloud Computing

The **NIST (National Institute of Standards and Technology)** defines 5 essential characteristics:

### 1. On-Demand Self-Service

> You can provision resources (servers, storage, databases) **yourself** through a web console or CLI — no need to call anyone or raise a ticket.

**Example:** Go to the AWS Console, click "Launch Instance," and you have a virtual server running in 60 seconds.

### 2. Broad Network Access

> Cloud services are accessible **over the internet** from any device — laptop, phone, tablet, or another server.

**Example:** You can SSH into your cloud server from your home laptop, office desktop, or even your phone.

### 3. Resource Pooling

> The provider's resources (CPU, memory, storage) are **shared across many customers** using multi-tenant architecture. You don't know (or care) which physical server you're on.

**Example:** Your virtual machine might be running on the same physical server as 10 other customers' VMs — but they're completely isolated from each other.

### 4. Rapid Elasticity

> Resources can **scale up or down automatically** based on demand. Need 100 servers for Black Friday? Done. Scale back to 5 after the sale? Done.

**Example:** An e-commerce site automatically adds more web servers when traffic spikes and removes them when traffic drops.

### 5. Measured Service

> Cloud usage is **metered and billed** precisely. You pay for exactly what you use — per hour, per GB, per request.

**Example:** You used a virtual machine for 73 hours this month → you pay for 73 hours. Not a flat monthly rate for a physical server sitting idle.

---

## Cloud Service Models

> [!tip] The Pizza Analogy
> Think of cloud service models like different ways to get pizza — from making everything yourself to having it delivered to your door.

### IaaS — Infrastructure as a Service

**What it is:** The cloud provider gives you the **raw building blocks** — virtual machines, storage, and networking. You install and manage everything on top: the operating system, applications, and data.

**Analogy:** Renting a **plot of land**. You get the land and utilities (electricity, water), but you design and build the house yourself.

**What YOU manage:**
- Operating System (Linux, Windows)
- Applications and runtime
- Data
- Security patches on the OS

**What the PROVIDER manages:**
- Physical servers and hardware
- Networking equipment
- Data centre facilities (power, cooling)
- Virtualisation layer (hypervisor)

**Examples:**
- **AWS EC2** (Elastic Compute Cloud) — Virtual machines
- **Azure Virtual Machines**
- **Google Compute Engine**
- **DigitalOcean Droplets**

**When to use:** When you need full control over the operating system and software stack.

---

### PaaS — Platform as a Service

**What it is:** The provider manages the **infrastructure AND the platform** (OS, runtime, middleware). You just deploy your application code and data.

**Analogy:** Renting a **furnished apartment**. The furniture (runtime, OS, middleware) is already there. You just bring your personal belongings (your code and data).

**What YOU manage:**
- Application code
- Data

**What the PROVIDER manages:**
- Everything in IaaS, PLUS:
- Operating System
- Runtime (Java, .NET, Node.js)
- Middleware
- Scaling and load balancing

**Examples:**
- **AWS Elastic Beanstalk**
- **Azure App Service**
- **Google App Engine**
- **Heroku**

**When to use:** When you want to focus purely on writing code and don't want to manage servers.

---

### SaaS — Software as a Service

**What it is:** The provider manages **everything**. You just open a browser and use the application. No installation, no maintenance, no infrastructure.

**Analogy:** Staying at a **hotel**. Everything is provided — the room, the bed, the cleaning, the food. You just show up and use it.

**What YOU manage:**
- Your data and user settings
- That's it!

**What the PROVIDER manages:**
- Everything — infrastructure, platform, AND application

**Examples:**
- **Gmail / Google Workspace**
- **Microsoft 365 (Word, Excel, Teams online)**
- **Salesforce**
- **Slack**
- **Zoom**

**When to use:** When you need ready-to-use software without any technical setup.

---

### FaaS — Function as a Service (Serverless)

**What it is:** You write a **single function** (a small piece of code), upload it, and the cloud runs it in response to an event. No servers to manage, no runtime to configure. You pay only when the function executes.

**Analogy:** Ordering **room service** at a hotel. You don't cook, you don't even go to the restaurant. You just say what you want, and it appears. You pay per order.

**What YOU manage:**
- Your function code (a few lines to a few hundred lines)

**What the PROVIDER manages:**
- Everything else — servers, scaling, runtime, OS

**Examples:**
- **AWS Lambda**
- **Azure Functions**
- **Google Cloud Functions**

**When to use:** For event-driven tasks — processing an uploaded image, responding to an API call, running a scheduled job.

---

### Service Models Comparison Table

| Aspect | IaaS | PaaS | SaaS | FaaS |
|---|---|---|---|---|
| **You manage** | OS, Apps, Data | Apps, Data | Data only | Function code only |
| **Provider manages** | Hardware, Network | Hardware to Runtime | Everything | Everything |
| **Control level** | High | Medium | Low | Very Low |
| **Flexibility** | Very High | Medium | Low | Medium |
| **Complexity** | High | Medium | Low | Low |
| **Scaling** | Manual or auto | Automatic | Automatic | Automatic |
| **Pricing** | Per hour/resource | Per app/request | Per user/month | Per execution |
| **Example** | AWS EC2 | Azure App Service | Gmail | AWS Lambda |
| **Analogy** | Plot of land | Furnished apartment | Hotel room | Room service |

> [!note] Remember
> As you move from IaaS → SaaS, you give up **control** but gain **convenience**. The right choice depends on your needs.

---

## Cloud Deployment Models

### Public Cloud

**What it is:** Cloud infrastructure that is **shared across multiple organisations** and accessible over the public internet. Managed entirely by the cloud provider.

**Characteristics:**
- Pay-as-you-go pricing — no upfront investment
- Massive scale — virtually unlimited resources
- Managed by the provider — you don't touch hardware
- Multi-tenant — resources are shared (but isolated) between customers

**Examples:** AWS, Microsoft Azure, Google Cloud Platform (GCP)

**Best for:** Startups, web applications, development/testing environments, any workload without strict data residency requirements.

---

### Private Cloud

**What it is:** Cloud infrastructure dedicated to a **single organisation**. Can be hosted on-premises or by a third party, but it's not shared.

**Characteristics:**
- Full control over hardware and data
- Higher security and compliance
- More expensive to build and maintain
- Less elastic than public cloud

**Examples:** VMware vSphere, OpenStack, Azure Stack

**Best for:** Government agencies, healthcare, financial institutions — anywhere with strict regulatory or security requirements.

---

### Hybrid Cloud

**What it is:** A **combination of public and private cloud**, connected together. Workloads can move between them based on needs.

**Characteristics:**
- Keep sensitive data in private cloud
- Burst to public cloud for extra capacity
- Requires networking between environments (VPN, dedicated connections)
- More complex to manage

**Use case example:** A hospital keeps patient records in a private cloud (HIPAA compliance) but uses AWS for their public website and appointment booking system.

**Examples:** Azure Arc, AWS Outposts, Google Anthos

---

### Multi-Cloud

**What it is:** Using **two or more public cloud providers** simultaneously (e.g., AWS + Azure + GCP).

**Why companies do this:**
- Avoid vendor lock-in
- Use the best service from each provider
- Redundancy — if one provider has an outage, others keep running
- Compliance — some data must stay in specific regions only available on certain providers

**Challenges:**
- Increased complexity
- Different APIs and tools for each provider
- Higher operational overhead
- Data transfer costs between providers

---

## Regions and Availability Zones

### What is a Region?

A **Region** is a **geographic location** where a cloud provider has data centres. Each provider has many regions around the world.

**Examples:**
- AWS: `us-east-1` (Virginia), `ap-south-1` (Mumbai), `eu-west-1` (Ireland)
- Azure: `East US`, `West Europe`, `Southeast Asia`

### What is an Availability Zone (AZ)?

An **Availability Zone** is one or more **isolated data centres within a region**. Each AZ has its own power, cooling, and networking.

```
Region: us-east-1 (N. Virginia)
├── AZ: us-east-1a  (Data Centre Cluster A)
├── AZ: us-east-1b  (Data Centre Cluster B)
├── AZ: us-east-1c  (Data Centre Cluster C)
├── AZ: us-east-1d  (Data Centre Cluster D)
├── AZ: us-east-1e  (Data Centre Cluster E)
└── AZ: us-east-1f  (Data Centre Cluster F)
```

### Why Do Regions and AZs Matter?

| Factor | Why It Matters |
|---|---|
| **Latency** | Deploy closer to your users for faster response times |
| **Compliance** | Some regulations require data to stay in a specific country |
| **Disaster Recovery** | Spread across AZs so if one data centre fails, others keep running |
| **Cost** | Pricing varies by region — some regions are cheaper |
| **Service Availability** | Not all services are available in all regions |

### How to Choose a Region

1. **Where are your users?** → Pick the closest region for low latency.
2. **What are your compliance needs?** → GDPR may require EU regions.
3. **What services do you need?** → Check if the region supports them.
4. **What's the cost?** → Compare pricing across regions.
5. **Do you need disaster recovery?** → Use multiple AZs or regions.

> [!tip] Best Practice
> Always deploy across **at least 2 Availability Zones** for high availability. If one AZ goes down, your application keeps running in the other.

---

## The Shared Responsibility Model

> [!warning] Critical Concept
> This is one of the MOST important concepts in cloud computing. Misunderstanding it is the #1 cause of cloud security breaches.

### What Is It?

The shared responsibility model defines **who is responsible for what** in cloud security:

- **Cloud Provider** → Responsible for security **OF** the cloud (physical infrastructure, hypervisor, network).
- **You (the Customer)** → Responsible for security **IN** the cloud (your data, access controls, application security).

### Responsibility by Service Model

| Responsibility | IaaS | PaaS | SaaS |
|---|---|---|---|
| **Data** | 🧑 You | 🧑 You | 🧑 You |
| **Application** | 🧑 You | 🧑 You | ☁️ Provider |
| **Runtime** | 🧑 You | ☁️ Provider | ☁️ Provider |
| **Operating System** | 🧑 You | ☁️ Provider | ☁️ Provider |
| **Middleware** | 🧑 You | ☁️ Provider | ☁️ Provider |
| **Virtualisation** | ☁️ Provider | ☁️ Provider | ☁️ Provider |
| **Networking** | ☁️ Provider | ☁️ Provider | ☁️ Provider |
| **Physical Hardware** | ☁️ Provider | ☁️ Provider | ☁️ Provider |
| **Data Centre** | ☁️ Provider | ☁️ Provider | ☁️ Provider |

> [!important] Key Takeaway
> **Your data is ALWAYS your responsibility**, no matter what service model you use. The cloud provider will never manage your data classification, access policies, or encryption keys for you.

### What This Means in Practice

**The provider secures:**
- Physical security of data centres (guards, biometrics, cameras)
- Hardware maintenance and replacement
- Network infrastructure
- Hypervisor patching

**YOU must secure:**
- Who can access your resources (IAM) → See [[Cloud Security Fundamentals]]
- Encryption of your data (at rest and in transit)
- Firewall rules and network configuration → See [[Cloud Networking Basics]]
- Application vulnerabilities
- Operating system patches (for IaaS)

---

## Core Cloud Services Overview

Cloud providers offer hundreds of services, but they all fall into a few major categories. Here's a map of what you'll learn in this vault:

### Compute — Running Your Code

Compute services provide the **processing power** to run your applications.

| Type | What It Does | AWS | Azure |
|---|---|---|---|
| Virtual Machines | Full OS, full control | EC2 | Virtual Machines |
| Containers | Lightweight, portable apps | ECS, EKS | ACI, AKS |
| Serverless | Run functions on demand | Lambda | Functions |
| App Platforms | Deploy code, no infra management | Elastic Beanstalk | App Service |

→ Deep dive: [[AWS Compute]] / [[Azure Compute]]

### Storage — Saving Your Data

| Type | What It Does | AWS | Azure |
|---|---|---|---|
| Object Storage | Store files, images, backups | S3 | Blob Storage |
| Block Storage | Disk for VMs | EBS | Managed Disks |
| File Storage | Shared file system | EFS | Azure Files |
| Archive | Cheap long-term storage | S3 Glacier | Archive Storage |

→ Deep dive: [[AWS Storage]] / [[Azure Storage]]

### Databases — Structured Data

| Type | What It Does | AWS | Azure |
|---|---|---|---|
| Relational (SQL) | Tables, rows, SQL queries | RDS, Aurora | Azure SQL, MySQL |
| NoSQL | Flexible schema, key-value | DynamoDB | Cosmos DB |
| In-Memory Cache | Ultra-fast data access | ElastiCache | Azure Cache for Redis |

→ Deep dive: [[AWS Databases]] / [[Azure Databases]]

### Networking — Connecting Everything

Networking is the backbone of cloud architecture. It controls how your resources communicate with each other and the internet.

- Virtual networks, subnets, firewalls
- Load balancers, DNS, CDN
- VPN and private connectivity

→ Deep dive: [[Cloud Networking Basics]]

### Security & Identity — Protecting Your Resources

Security is not optional — it's built into every layer.

- Identity and Access Management (IAM)
- Encryption, key management
- Compliance and governance

→ Deep dive: [[Cloud Security Fundamentals]]

### Monitoring & Observability — Knowing What's Happening

You can't fix what you can't see.

- Metrics, logs, traces
- Alerts and dashboards
- Cost monitoring

→ Deep dive: [[Monitoring and Observability]]

---

## Cloud Pricing Concepts

> [!note] Why This Matters
> One of the biggest mistakes beginners make is accidentally leaving resources running and getting a surprise bill.

### Common Pricing Models

| Model | How It Works | Example |
|---|---|---|
| **Pay-As-You-Go** | Pay per hour/second of usage | EC2 instance: $0.10/hour |
| **Reserved** | Commit to 1-3 years for a discount | EC2 Reserved: 30-60% off |
| **Spot/Preemptible** | Use spare capacity at huge discounts (can be interrupted) | EC2 Spot: up to 90% off |
| **Free Tier** | Limited free usage for new accounts | AWS: 750 hrs/month of t2.micro for 12 months |

### Tips to Avoid Surprise Bills

1. **Set billing alerts** — Get notified when costs exceed a threshold.
2. **Use the free tier** — Stay within free limits while learning.
3. **Delete resources when done** — Don't leave VMs and databases running.
4. **Use cost calculators** — AWS Pricing Calculator, Azure Pricing Calculator.
5. **Tag your resources** — Know which project/team is spending what.

---

## What's Next?

Now that you understand the fundamentals, explore these topics:

1. **Networking** → [[Cloud Networking Basics]] — How cloud resources communicate.
2. **Security** → [[Cloud Security Fundamentals]] — How to protect your cloud resources.
3. **AWS Specific** → [[AWS Compute]], [[AWS Storage]], [[AWS Databases]]
4. **Azure Specific** → [[Azure Compute]], [[Azure Storage]], [[Azure Databases]]
5. **Monitoring** → [[Monitoring and Observability]]

---

## Quick Reference: Cloud Glossary

| Term | Definition |
|---|---|
| **Region** | A geographic location with data centres |
| **Availability Zone** | An isolated data centre within a region |
| **Instance** | A virtual machine running in the cloud |
| **Elasticity** | Ability to scale resources up/down automatically |
| **Multi-Tenancy** | Multiple customers sharing the same physical infrastructure |
| **Provisioning** | Setting up and configuring cloud resources |
| **Hypervisor** | Software that creates and manages virtual machines |
| **API** | Application Programming Interface — how you interact with cloud services programmatically |
| **CLI** | Command Line Interface — text-based tool to manage cloud resources |
| **Console** | Web-based dashboard to manage cloud resources (e.g., AWS Console) |
| **IAM** | Identity and Access Management — controls who can do what |
| **VPC** | Virtual Private Cloud — your isolated network in the cloud |
| **ARN** | Amazon Resource Name — unique identifier for AWS resources |
| **SLA** | Service Level Agreement — provider's uptime guarantee |
| **TCO** | Total Cost of Ownership — full cost comparison (on-prem vs cloud) |

---

> [!success] Summary
> Cloud computing lets you rent computing resources instead of buying them. It comes in different service models (IaaS, PaaS, SaaS, FaaS), deployment models (public, private, hybrid, multi-cloud), and is organised into regions and availability zones. Security is a shared responsibility between you and the provider. Master these fundamentals before diving into specific services.
