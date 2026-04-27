# ☁️ AWS vs Azure Comparison

> A side-by-side cheat sheet mapping AWS services to their Azure equivalents.
> Use this as a quick reference when learning either platform — or both!

---

## 📖 Overview

**Amazon Web Services (AWS)** and **Microsoft Azure** are the two largest cloud providers in the world. Together they hold a significant majority of the global cloud market share.

### A Brief History

- **AWS** launched in **2006** with S3 (storage) and EC2 (compute). It was the first major public cloud platform and has had a head start of several years over competitors.
- **Azure** launched in **2010** (originally as "Windows Azure," rebranded to "Microsoft Azure" in 2014). It leveraged Microsoft's massive enterprise customer base to grow rapidly.

### Market Position

- **AWS** has the **largest market share** (~31%) and the widest range of services (200+). It is the default choice for many startups, tech companies, and cloud-native organizations.
- **Azure** is the **second largest** (~25%) and is especially popular with **enterprises already using Microsoft** products like Office 365, Active Directory, Windows Server, and SQL Server.
- Both platforms are available in **dozens of regions worldwide** with multiple availability zones per region for high availability.

> 💡 The good news: core cloud concepts are the same across both. Learning one makes learning the other **much** easier!

For a deeper dive into the fundamentals that apply to both, see [[Cloud Fundamentals]].

---

## 🗂️ Service Mapping Table

This is the core of the cheat sheet. Find the AWS service you know and see the Azure equivalent — or vice versa.

### Compute

| AWS Service | Azure Equivalent | What It Does |
|-------------|-----------------|--------------|
| **EC2** (Elastic Compute Cloud) | **Virtual Machines** | On-demand virtual servers. Both support Linux and Windows. You choose the OS, size, and configuration. See [[AWS Compute]] and [[Azure Compute]]. |
| **Lambda** | **Azure Functions** | Serverless, event-driven functions. You write code; the cloud runs it. Pay only for execution time. No servers to manage. |
| **EKS** (Elastic Kubernetes Service) | **AKS** (Azure Kubernetes Service) | Managed Kubernetes for container orchestration. Both handle the control plane; you manage your pods and deployments. |
| **ECS** (Elastic Container Service) / **Fargate** | **Container Instances** | Simplified container hosting without managing Kubernetes. Fargate and Container Instances both let you run containers without managing VMs. |
| **Elastic Beanstalk** | **App Service** | Platform-as-a-Service (PaaS) for web apps. Deploy code without managing infrastructure. Supports multiple languages (Java, Python, Node.js, .NET, etc.). |

> 📝 For a full breakdown of compute options, see [[AWS Compute]] and [[Azure Compute]].

### Storage

| AWS Service | Azure Equivalent | What It Does |
|-------------|-----------------|--------------|
| **S3** (Simple Storage Service) | **Blob Storage** | Unlimited, scalable object storage. Store files, images, backups, logs — anything. The backbone of cloud storage. See [[AWS Storage]] and [[Azure Storage]]. |
| **EBS** (Elastic Block Store) | **Managed Disks** | Block storage attached to VMs. Think of it like a virtual hard drive plugged into your virtual machine. |
| **EFS** (Elastic File System) | **Azure Files** | Shared file systems that multiple VMs can mount simultaneously. Supports NFS (AWS) and SMB (Azure). |
| **S3 Glacier** | **Archive Storage** | Very cheap storage for cold/long-term data (backups, compliance archives). Retrieval takes minutes to hours. |

> 📝 For a full breakdown of storage options, see [[AWS Storage]] and [[Azure Storage]].

### Databases

| AWS Service | Azure Equivalent | What It Does |
|-------------|-----------------|--------------|
| **RDS** (Relational Database Service) | **Azure SQL Database** | Managed relational databases. AWS RDS supports MySQL, PostgreSQL, MariaDB, Oracle, SQL Server. Azure SQL is optimized for SQL Server. See [[AWS Databases]] and [[Azure Databases]]. |
| **DynamoDB** | **Cosmos DB** | NoSQL document databases. DynamoDB is key-value/document. Cosmos DB supports multiple APIs (SQL, MongoDB, Cassandra, Gremlin, Table). |
| **ElastiCache** | **Azure Cache for Redis** | In-memory caching (Redis-compatible). Dramatically speeds up read-heavy workloads by caching frequently accessed data. |

> 📝 For a full breakdown of database options, see [[AWS Databases]] and [[Azure Databases]].

### Networking

| AWS Service | Azure Equivalent | What It Does |
|-------------|-----------------|--------------|
| **VPC** (Virtual Private Cloud) | **VNet** (Virtual Network) | Isolated virtual networks. Your own private section of the cloud where you launch resources. See [[AWS Networking]] and [[Azure Networking]]. |
| **Route 53** | **Azure DNS** | Domain Name System (DNS) management. Map domain names (like `example.com`) to IP addresses. Also see [[Cloud Networking Basics]]. |
| **ALB** (Application Load Balancer) | **Application Gateway** | Layer 7 (HTTP/HTTPS) load balancing. Routes traffic based on URLs, headers, or host names. |
| **NLB** (Network Load Balancer) | **Azure Load Balancer** | Layer 4 (TCP/UDP) load balancing. Ultra-fast, handles millions of requests per second. |
| **CloudFront** | **Azure CDN** | Content Delivery Network. Caches content at edge locations worldwide for faster delivery to users. |

> 📝 For networking fundamentals, see [[Cloud Networking Basics]], [[AWS Networking]], and [[Azure Networking]].

### Security & Identity

| AWS Service | Azure Equivalent | What It Does |
|-------------|-----------------|--------------|
| **IAM** (Identity and Access Management) | **Entra ID** (formerly Azure AD) + **RBAC** | User and role management. Controls who can access what. The foundation of cloud security. See [[AWS IAM]] and [[Azure IAM]]. |
| **KMS** (Key Management Service) | **Key Vault** | Encryption key management. Create, store, and control cryptographic keys used to encrypt your data. See [[AWS Security]] and [[Azure Security]]. |
| **Secrets Manager** | **Key Vault** (Secrets) | Secure credential storage. Store API keys, database passwords, tokens — never hardcode secrets in your code! |
| **WAF** (Web Application Firewall) + **Shield** | **Azure WAF** + **DDoS Protection** | Protect web apps from common attacks (SQL injection, XSS) and DDoS attacks. |

> 📝 For security fundamentals, see [[Cloud Security Fundamentals]], [[AWS Security]], and [[Azure Security]].

### Monitoring & Observability

| AWS Service | Azure Equivalent | What It Does |
|-------------|-----------------|--------------|
| **CloudWatch** | **Azure Monitor** | Centralized monitoring: metrics, logs, dashboards, and alarms. The "single pane of glass" for your cloud resources. See [[AWS Monitoring]] and [[Azure Monitoring]]. |
| **CloudTrail** | **Activity Log** | Audit trail. Records every API call — who did what, when, and from where. Essential for compliance and troubleshooting. |
| **X-Ray** | **Application Insights** | Distributed tracing. Follows a single request as it travels through multiple microservices. Helps find bottlenecks. |

> 📝 For monitoring fundamentals, see [[Monitoring and Observability]], [[AWS Monitoring]], and [[Azure Monitoring]].

### DevOps & CI/CD

| AWS Service | Azure Equivalent | What It Does |
|-------------|-----------------|--------------|
| **CodePipeline** | **Azure DevOps Pipelines** | CI/CD pipeline — automatically build, test, and deploy your code when you push changes. See [[AWS Deployment]] and [[Azure Deployment]]. |
| **CloudFormation** | **ARM Templates** / **Bicep** | Infrastructure as Code (IaC). Define your entire cloud infrastructure in template files. See [[Cloud Deployment Strategies]]. |
| **ECR** (Elastic Container Registry) | **ACR** (Azure Container Registry) | Store and manage Docker container images. Your private Docker Hub in the cloud. |

> 📝 For deployment fundamentals, see [[Cloud Deployment Strategies]], [[AWS Deployment]], and [[Azure Deployment]].

### Cost Management

| AWS Service | Azure Equivalent | What It Does |
|-------------|-----------------|--------------|
| **Free Tier** | **Free Tier** | Both offer **12 months of free services** for new accounts, plus always-free tiers for selected services. See [[AWS Billing]] and [[Azure Billing]]. |
| **Cost Explorer** | **Cost Management** | Analyze and visualize your cloud spending. Find out which services cost the most and where to optimize. |
| **Pricing Calculator** | **Pricing Calculator** | Estimate costs before you build. Enter your expected usage and get a monthly cost estimate. |

> 📝 For cost management fundamentals, see [[Cost Management]], [[AWS Billing]], and [[Azure Billing]].

---

## 🔑 Key Differences

While AWS and Azure offer very similar capabilities, there are important differences to be aware of:

### AWS Strengths

- **More services**: AWS has 200+ services — the widest catalog of any cloud provider. If a niche service exists, AWS probably has it.
- **More mature**: With a 4-year head start, many AWS services are more battle-tested and have more documentation, tutorials, and community support.
- **Larger community**: More Stack Overflow answers, blog posts, courses, and certifications available.
- **Startup-friendly**: AWS is the default choice for many startups and tech companies, so skills are highly marketable.

### Azure Strengths

- **Microsoft integration**: If you already use Office 365, Active Directory, Windows Server, SQL Server, or .NET, Azure integrates seamlessly.
- **Enterprise-friendly**: Azure's licensing model often benefits enterprises with existing Microsoft Enterprise Agreements.
- **Simpler RBAC**: Azure's Role-Based Access Control (via Entra ID) can be simpler to set up than AWS's JSON-based IAM policies. See [[Azure IAM]] for details.
- **Hybrid cloud**: Azure Arc and Azure Stack make it easier to extend Azure into your on-premises data centers.

### CLI Comparison

Both clouds have command-line tools:

| | AWS | Azure |
|--|-----|-------|
| **CLI Tool** | `aws` | `az` |
| **Install** | `pip install awscli` or installer | `pip install azure-cli` or installer |
| **Configure** | `aws configure` | `az login` |
| **Example** | `aws s3 ls` | `az storage account list` |

See [[AWS Getting Started]] and [[Azure Getting Started]] for installation guides.

### Console Comparison

| | AWS | Azure |
|--|-----|-------|
| **Name** | AWS Management Console | Azure Portal |
| **URL** | `console.aws.amazon.com` | `portal.azure.com` |
| **Style** | Service-centric (navigate by service) | Resource-group-centric (organize by project) |
| **Search** | Service search bar at top | Global search bar at top |

---

## 📝 Terminology Differences

One of the biggest hurdles when switching between AWS and Azure is that they use **different names for the same things**. Here is a quick reference:

| Concept | AWS Term | Azure Term |
|---------|----------|------------|
| Virtual server | **Instance** (EC2 instance) | **VM** (Virtual Machine) |
| Firewall rules for a VM | **Security Group** | **NSG** (Network Security Group) |
| Object storage bucket | **S3 Bucket** | **Storage Account / Container** |
| Access control policy | **IAM Policy** (JSON document) | **RBAC Role Assignment** |
| Infrastructure template | **CloudFormation Stack** | **ARM Deployment** / **Bicep Deployment** |
| Serverless function | **Lambda Function** | **Azure Function** |
| Managed Kubernetes | **EKS Cluster** | **AKS Cluster** |
| Container image registry | **ECR Repository** | **ACR Registry** |
| DNS zone | **Route 53 Hosted Zone** | **Azure DNS Zone** |
| Private network | **VPC** | **VNet** |
| Network subdivision | **Subnet** | **Subnet** (same term!) |
| Availability Zone | **AZ** (e.g., `us-east-1a`) | **Availability Zone** (e.g., Zone 1, 2, 3) |
| Region | **Region** (e.g., `us-east-1`) | **Region** (e.g., `East US`) |
| Auto-scaling group | **Auto Scaling Group** | **VM Scale Set** |
| Secret storage | **Secrets Manager** | **Key Vault Secret** |
| Encryption keys | **KMS Key** | **Key Vault Key** |
| Monitoring dashboard | **CloudWatch Dashboard** | **Azure Monitor Dashboard** |
| API call log | **CloudTrail Event** | **Activity Log Entry** |

> 💡 Don't worry about memorizing all of these at once. As you work through [[AWS Getting Started]] or [[Azure Getting Started]], the terminology will start to feel natural.

---

## 🤔 Which One Should You Choose?

This is one of the most common questions beginners ask. Here is some practical guidance:

### Choose Azure if…

- You or your company **already uses Microsoft products** (Office 365, Active Directory, SQL Server, .NET, Visual Studio).
- Your organization has a **Microsoft Enterprise Agreement** — you may get discounted Azure pricing.
- You want **tight integration** with tools like Azure DevOps, Visual Studio, and GitHub (Microsoft owns GitHub!).
- You are focused on **hybrid cloud** scenarios (mixing on-premises and cloud).

### Choose AWS if…

- You want the **widest range of services** and the most options for every category.
- You are joining a **startup or tech company** — AWS is the most commonly used cloud in these environments.
- You want the **largest community** of tutorials, courses, and Stack Overflow answers.
- You want the most **certifications and career opportunities** — AWS certifications are among the most sought-after in IT.

### The Real Answer

> **The skills are highly transferable.** The core concepts — virtual machines, object storage, networking, IAM, monitoring, IaC — are the same. If you learn one platform deeply, you can pick up the other in weeks, not months.

Pick the one that is most relevant to your job or career goals, learn it well, and then expand to the other when needed.

For a deeper understanding of the concepts that apply to **both** platforms, start with:

- [[Cloud Fundamentals]] — the big picture
- [[Cloud Networking Basics]] — how cloud networking works
- [[Cloud Security Fundamentals]] — security principles
- [[Cloud Deployment Strategies]] — modern deployment practices
- [[Monitoring and Observability]] — keeping your systems healthy
- [[Cost Management]] — keeping your wallet healthy

---

## 🔗 Related Notes

### AWS Deep Dives

- [[AWS Getting Started]] — Set up your AWS account and tools
- [[AWS IAM]] — Identity and access management
- [[AWS Compute]] — EC2, Lambda, ECS, EKS
- [[AWS Storage]] — S3, EBS, EFS
- [[AWS Databases]] — RDS, DynamoDB, ElastiCache
- [[AWS Networking]] — VPC, Route 53, load balancers
- [[AWS Deployment]] — Elastic Beanstalk, CodePipeline, CloudFormation
- [[AWS Monitoring]] — CloudWatch, CloudTrail, X-Ray
- [[AWS Security]] — KMS, Secrets Manager, WAF
- [[AWS Billing]] — Free tier and cost optimization

### Azure Deep Dives

- [[Azure Getting Started]] — Set up your Azure account and tools
- [[Azure IAM]] — Entra ID, RBAC, managed identities
- [[Azure Compute]] — VMs, Functions, AKS, App Service
- [[Azure Storage]] — Blob, Disk, Files
- [[Azure Databases]] — Azure SQL, Cosmos DB, Redis Cache
- [[Azure Networking]] — VNet, NSGs, DNS, Application Gateway
- [[Azure Deployment]] — App Service, Azure DevOps, ARM/Bicep
- [[Azure Monitoring]] — Azure Monitor, Log Analytics, Application Insights
- [[Azure Security]] — Key Vault, Defender, WAF
- [[Azure Billing]] — Free tier and cost management

---

> 🗺️ Go back to the [[Index]] to see all available cloud notes.
