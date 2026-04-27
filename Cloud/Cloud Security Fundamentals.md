# Cloud Security Fundamentals

> [!info] Prerequisites
> Read [[Cloud Fundamentals]] first (especially the Shared Responsibility Model section), then [[Cloud Networking Basics]] for firewall and network concepts referenced here.

---

## Why Cloud Security Matters

Cloud security is not optional — it's the foundation of everything you build. A single misconfiguration can expose sensitive data to the entire internet.

### Key Facts

- **Most cloud security breaches are caused by the CUSTOMER, not the provider.** The cloud provider secures the infrastructure; YOU must secure what you put on it.
- The **Shared Responsibility Model** (covered in [[Cloud Fundamentals]]) defines the boundary.
- Cloud providers spend **billions** on security — their infrastructure is more secure than most on-premises data centres. But that doesn't help if YOU leave a database open to the internet.

### Common Cloud Security Mistakes

| Mistake | Consequence |
|---|---|
| Using root/admin account for daily work | One compromised password = total control |
| No MFA enabled | Accounts easily hijacked |
| Overly permissive IAM policies (`*:*`) | Anyone can do anything |
| Public S3 buckets / Blob containers | Sensitive data exposed to the internet |
| Hardcoded secrets in code | Credentials leaked via Git repositories |
| Unused open security group ports | Attack surface unnecessarily large |
| No encryption on data at rest | Data theft if storage is breached |
| No logging or monitoring | Breaches go undetected for months |

> [!warning] Real World
> In 2017, Uber had 57 million user records exposed because developers hardcoded AWS access keys in a GitHub repository. Attackers found the keys and accessed Uber's S3 bucket. This is preventable.

---

## Identity and Access Management (IAM)

### What Is IAM?

**IAM** stands for **Identity and Access Management**. It is the system that controls:
- **WHO** can access your cloud resources (authentication)
- **WHAT** they can do with those resources (authorisation)

Every cloud provider has an IAM service:
- AWS → **AWS IAM**
- Azure → **Microsoft Entra ID** (formerly Azure AD) + **Azure RBAC**
- GCP → **Google Cloud IAM**

### Core IAM Concepts

#### Users

A **user** represents a single person (or system) that needs access to cloud resources.

- Each user has their own **credentials** (username + password, or access keys).
- Users should be **individual** — never share accounts.

#### Groups

A **group** is a collection of users. Permissions are assigned to the group, and all users in that group inherit those permissions.

**Example:**
- Group: `Developers` → Permissions: read/write to code repositories, deploy to dev environment
- Group: `DBAdmins` → Permissions: manage databases, read production data
- Group: `ReadOnly` → Permissions: view resources, no modifications

> [!tip] Best Practice
> **Never assign permissions directly to users.** Always assign permissions to **groups**, then add users to groups. This is much easier to manage at scale.

#### Roles

A **role** is an identity with permissions that can be **assumed** by users, services, or applications — without needing permanent credentials.

**Key difference from users:**
- A **user** has permanent credentials (password, access keys).
- A **role** has **temporary** credentials that expire after a set time.

**When to use roles:**
- When an EC2 instance needs to access S3 (assign a role to the instance)
- When one AWS account needs to access resources in another account
- When a Lambda function needs to write to a database

#### Policies and Permissions

A **policy** is a JSON document that defines what actions are allowed or denied on which resources.

**Policy structure (simplified):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

**Breaking it down:**
- **Effect** — `Allow` or `Deny`
- **Action** — What operations (e.g., `s3:GetObject` means "read from S3")
- **Resource** — Which specific resources (e.g., a specific S3 bucket)

#### Types of Policies

| Type | Description |
|---|---|
| **Managed Policies** | Pre-built by the provider (e.g., `AmazonS3ReadOnlyAccess`) |
| **Custom Policies** | Written by you for specific needs |
| **Inline Policies** | Embedded directly in a user/group/role (not reusable) |

### Principle of Least Privilege

> [!important] The Golden Rule of IAM
> Grant only the **minimum permissions** needed to perform a task. Nothing more.

**Bad example:**
```json
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}
```
This allows EVERYTHING on EVERY resource. This is essentially giving someone the master key to your entire cloud account.

**Good example:**
```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject"],
  "Resource": "arn:aws:s3:::reports-bucket/*"
}
```
This allows ONLY reading objects from ONE specific bucket.

### Multi-Factor Authentication (MFA)

**What it is:** A second layer of security beyond just a password.

**How it works:**
1. You enter your password (something you **know**)
2. You enter a code from your phone/hardware key (something you **have**)
3. Both must be correct to log in

**Types of MFA:**
- **Virtual MFA** — Authenticator app on your phone (Google Authenticator, Authy)
- **Hardware MFA** — Physical device (YubiKey, hardware token)
- **SMS MFA** — Code sent via text message (least secure, avoid if possible)

> [!warning] Non-Negotiable
> **ALWAYS enable MFA on root/admin accounts.** This is the single most impactful security action you can take. A password alone is not enough.

### Service Accounts / Service Principals

These are **non-human identities** used by applications, scripts, and services to authenticate with the cloud.

| Provider | Term | Purpose |
|---|---|---|
| AWS | **IAM Roles** (for services) | EC2 instances, Lambda functions, etc. |
| Azure | **Service Principals** | Applications that need Azure access |
| Azure | **Managed Identities** | Azure resources that access other Azure resources |
| GCP | **Service Accounts** | Applications running on GCP |

> [!tip] Best Practice
> Never use personal user credentials in application code. Always use **roles** or **service accounts** with temporary credentials.

---

## Authentication vs Authorisation

These are two different concepts that people often confuse:

| Concept | Question It Answers | Example |
|---|---|---|
| **Authentication** (AuthN) | **WHO are you?** | Logging in with username + password + MFA |
| **Authorisation** (AuthZ) | **WHAT can you do?** | User has permission to read S3 but not delete |

**Real-world analogy:**
- **Authentication** = Showing your badge to enter the office building (proving you are who you say you are).
- **Authorisation** = Your badge only opens certain doors (you can access the engineering floor but not the server room).

**The flow:**
```
1. User tries to access a resource
2. Authentication: "Prove who you are" → username, password, MFA
3. Authorisation: "Check what you're allowed to do" → evaluate IAM policies
4. Access granted or denied
```

---

## Role-Based Access Control (RBAC)

### What Is RBAC?

**RBAC** assigns permissions based on **roles** rather than individual users. A role represents a job function, and users are assigned to roles.

### Why RBAC?

- **Scalable** — Add a new developer? Assign the "Developer" role. Done.
- **Auditable** — Easy to see who has what access.
- **Consistent** — All developers get the same permissions.
- **Secure** — Remove a role when someone changes teams.

### Example RBAC Setup

| Role | Permissions | Who Gets It |
|---|---|---|
| **Owner / Admin** | Full control over everything | Cloud account administrators |
| **Contributor / Developer** | Create, update, delete resources (no IAM changes) | Development teams |
| **Reader / Viewer** | View resources only (no modifications) | Auditors, stakeholders |
| **Security Admin** | Manage IAM, policies, encryption | Security team |
| **Billing Admin** | View and manage billing | Finance team |
| **Database Admin** | Manage databases and backups | DBA team |

### RBAC in Cloud Providers

| Provider | RBAC System |
|---|---|
| AWS | IAM Policies + IAM Roles (plus AWS SSO for enterprise) |
| Azure | Azure RBAC (built-in roles: Owner, Contributor, Reader + custom roles) |
| GCP | Cloud IAM (predefined roles + custom roles) |

---

## Encryption

### Why Encrypt?

Encryption converts readable data into unreadable ciphertext. Even if someone steals your data, they can't read it without the encryption key.

### Encryption at Rest

**What it means:** Encrypting data **where it is stored** — on disk, in a database, in object storage.

**How it works:**
```
Your Data: "Customer SSN: 123-45-6789"
   ↓ Encrypt with AES-256
Stored as: "a3f8b2c1e9d4..." (unreadable gibberish)
   ↓ Decrypt with the key
Your Data: "Customer SSN: 123-45-6789"
```

**Common standards:**
- **AES-256** — The industry standard. Used by AWS, Azure, GCP, governments, and military.

**Cloud services that support encryption at rest:**
- AWS S3 (SSE-S3, SSE-KMS, SSE-C)
- Azure Blob Storage (Azure Storage Service Encryption)
- All managed databases (RDS, Azure SQL, Cloud SQL)

### Encryption in Transit

**What it means:** Encrypting data **while it moves** between two points — between your browser and a server, between two cloud services, etc.

**How it works:**
- **TLS (Transport Layer Security)** — The protocol that secures HTTPS connections.
- When you see `https://` in your browser, TLS is encrypting the data in transit.
- Version: TLS 1.2 or TLS 1.3 (older versions like SSL and TLS 1.0/1.1 are deprecated).

### Client-Side vs Server-Side Encryption

| Type | Who Encrypts | Who Holds the Key | Use Case |
|---|---|---|---|
| **Server-Side** | The cloud provider | Provider manages keys (or you via KMS) | Most common — easiest to implement |
| **Client-Side** | You (before uploading) | You manage keys entirely | Maximum control — data encrypted before it leaves your machine |

### Key Management Service (KMS)

A **KMS** is a managed service for creating, storing, and controlling encryption keys.

**Why use KMS instead of managing keys yourself?**
- Keys are stored in **hardware security modules (HSMs)** — tamper-resistant hardware.
- Automatic **key rotation** — Keys are periodically changed.
- **Audit trail** — Every use of a key is logged.
- **Access control** — IAM policies determine who can use which keys.

**Cloud KMS services:**

| Provider | Service |
|---|---|
| AWS | **AWS KMS** (Key Management Service) |
| Azure | **Azure Key Vault** |
| GCP | **Cloud KMS** |

---

## Secrets Management

### The Problem

Applications need credentials to access databases, APIs, and other services. Where do you store these credentials?

**❌ NEVER do this:**

```java
// NEVER hardcode secrets in source code
String dbPassword = "SuperSecret123!";
String apiKey = "AKIAIOSFODNN7EXAMPLE";
```

**Why not?**
- Source code is stored in Git → credentials get committed → anyone with repo access sees them.
- If the repo is public, bots scan GitHub and find exposed keys within **minutes**.
- Credentials in code are hard to rotate — you have to redeploy the application.

### The Solution: Secrets Vaults

Use a **secrets management service** to store and retrieve credentials securely.

| Provider | Service | Purpose |
|---|---|---|
| AWS | **AWS Secrets Manager** | Store and rotate secrets (DB passwords, API keys) |
| AWS | **AWS Systems Manager Parameter Store** | Store configuration and secrets (simpler, cheaper) |
| Azure | **Azure Key Vault** | Store secrets, keys, and certificates |
| GCP | **Secret Manager** | Store and access secrets |
| Multi-Cloud | **HashiCorp Vault** | Open-source, works with any provider |

### How It Works

```
1. Store the secret in the vault:
   Secret Name: "prod/database/password"
   Secret Value: "SuperSecret123!"

2. Your application retrieves the secret at runtime:
   password = secrets_manager.get_secret("prod/database/password")

3. The secret is never in your code or config files.
```

### Secrets Best Practices

- ✅ **Use a secrets vault** — Never hardcode or store in environment variables in code.
- ✅ **Enable automatic rotation** — Change passwords/keys automatically on a schedule.
- ✅ **Restrict access** — Only the application that needs the secret should be able to read it.
- ✅ **Audit access** — Log every time a secret is read.
- ✅ **Use temporary credentials** — Prefer IAM roles with temporary tokens over long-lived access keys.

---

## Network Security

### Firewalls Recap

As covered in [[Cloud Networking Basics]], cloud networking provides two firewall layers:

- **Security Groups** — Instance-level, stateful firewall (allow rules only).
- **Network ACLs** — Subnet-level, stateless firewall (allow and deny rules).

### Web Application Firewall (WAF)

**What it is:** A firewall specifically designed to protect **web applications** from common attacks.

**What it protects against:**
- **SQL Injection** — Malicious SQL queries in input fields
- **Cross-Site Scripting (XSS)** — Injecting malicious scripts into web pages
- **Bot attacks** — Automated scraping, credential stuffing
- **Common exploits** — Known vulnerability patterns

**Cloud WAF services:**

| Provider | Service |
|---|---|
| AWS | **AWS WAF** |
| Azure | **Azure Web Application Firewall** (with Front Door or App Gateway) |
| GCP | **Cloud Armor** |

### DDoS Protection

**DDoS (Distributed Denial of Service)** is an attack where millions of fake requests flood your application to make it unavailable.

**Cloud DDoS protection:**

| Provider | Service | What It Does |
|---|---|---|
| AWS | **AWS Shield Standard** (free) | Automatic protection against common DDoS attacks |
| AWS | **AWS Shield Advanced** (paid) | Enhanced DDoS protection with 24/7 response team |
| Azure | **Azure DDoS Protection** | Network-level DDoS mitigation |
| GCP | **Cloud Armor** | DDoS and WAF combined |

---

## Compliance and Governance

### What Is Compliance?

**Compliance** means following specific **rules, regulations, and standards** that apply to your industry or data type. Cloud providers help you meet compliance requirements, but **you are ultimately responsible** for ensuring your workloads are compliant.

### Common Compliance Standards

| Standard | What It Covers | Who Needs It |
|---|---|---|
| **SOC 2** | Security, availability, confidentiality of services | SaaS companies, any company handling customer data |
| **HIPAA** | Protected Health Information (PHI) | Healthcare organisations in the US |
| **GDPR** | Personal data of EU residents | Any company processing EU citizen data |
| **PCI DSS** | Credit card data | Any company handling payment cards |
| **ISO 27001** | Information security management | Enterprises wanting certified security practices |
| **FedRAMP** | US government workloads | Cloud services used by US government agencies |

### Shared Responsibility for Compliance

```
Cloud Provider's Responsibility:
- Physical data centre security and certifications
- Infrastructure compliance (SOC reports, ISO certifications)
- Providing compliant SERVICES (encrypted storage, logging, access controls)

Your Responsibility:
- Configuring services correctly (encryption enabled, access restricted)
- Managing your data classification
- Implementing required controls for YOUR application
- Documenting and proving compliance for auditors
```

### Audit Trails and Logging

Every action in the cloud should be **logged** for security and compliance:

| Provider | Service | What It Logs |
|---|---|---|
| AWS | **CloudTrail** | Every API call made in your AWS account |
| Azure | **Azure Activity Log** | Management operations on Azure resources |
| Azure | **Azure Monitor / Log Analytics** | Detailed metrics and logs |
| GCP | **Cloud Audit Logs** | Admin activity, data access, system events |

> [!important] Always Enable Logging
> Enable cloud audit logging from **day one**. If a breach occurs, logs are the first thing investigators look at. Without logs, you're flying blind.

---

## Security Best Practices Checklist

Use this checklist for every cloud environment you set up:

### Identity & Access

- [ ] **Enable MFA** on all user accounts, especially root/admin
- [ ] **Delete or disable the root account access keys** (AWS)
- [ ] **Use IAM roles**, not long-lived access keys, for applications
- [ ] **Apply least privilege** — every user/role gets only what they need
- [ ] **Use groups** to assign permissions, not individual users
- [ ] **Review and remove unused users/roles** regularly
- [ ] **Enable SSO** (Single Sign-On) for enterprise accounts

### Data Protection

- [ ] **Encrypt all data at rest** — Enable default encryption on all storage
- [ ] **Encrypt all data in transit** — Use HTTPS/TLS everywhere
- [ ] **Use KMS** for key management — Don't manage encryption keys manually
- [ ] **Never hardcode secrets** — Use a secrets vault
- [ ] **Enable automatic secret rotation**
- [ ] **Block public access** on storage by default (S3 Block Public Access, Azure private containers)

### Network Security

- [ ] **Use private subnets** for databases and application servers
- [ ] **Restrict Security Group rules** — No `0.0.0.0/0` on SSH (port 22) or RDP (port 3389)
- [ ] **Use a WAF** for web applications
- [ ] **Enable DDoS protection**
- [ ] **Use VPC endpoints / Private Links** instead of public internet for service-to-service communication

### Monitoring & Response

- [ ] **Enable cloud audit logging** (CloudTrail, Azure Activity Log)
- [ ] **Set up alerts** for suspicious activity (root login, policy changes, unusual API calls)
- [ ] **Monitor costs** — Unexpected cost spikes can indicate a breach (crypto mining)
- [ ] **Use security scanning tools** — AWS Security Hub, Azure Defender, GCP Security Command Centre
- [ ] **Have an incident response plan** — Know what to do when something goes wrong

### Governance

- [ ] **Tag all resources** — Environment, owner, project, cost centre
- [ ] **Use infrastructure as code** — Terraform, CloudFormation, Bicep (prevents configuration drift)
- [ ] **Enable compliance scanning** — AWS Config, Azure Policy
- [ ] **Regularly audit** IAM permissions and resource configurations

---

## Zero Trust Security Model

### What Is Zero Trust?

Traditional security follows the **castle-and-moat** model:
- Everything **inside** the network is trusted.
- The firewall (moat) keeps bad actors **outside**.

**Zero Trust** flips this:
- **Never trust, always verify.**
- No user, device, or service is trusted by default — even if they're inside the network.
- Every request must be **authenticated and authorised**, every time.

### Zero Trust Principles

| Principle | What It Means |
|---|---|
| **Verify explicitly** | Always authenticate and authorise based on all available data (user identity, location, device health, service, data classification) |
| **Least privilege access** | Grant just enough permissions for the task, with just-in-time and just-enough-access (JIT/JEA) |
| **Assume breach** | Minimise blast radius. Segment access. Verify end-to-end encryption. Use analytics for threat detection |

### Zero Trust in Practice

```
Traditional: User logs in with VPN → trusted for everything on the network
Zero Trust:  User logs in → verified at every step:
  ├── Is their identity verified? (MFA)
  ├── Is their device compliant? (patched, antivirus running)
  ├── Are they accessing from an expected location?
  ├── Do they have permission for THIS specific resource?
  └── Is their session still valid? (time-limited access)
```

### Cloud Services Supporting Zero Trust

| Provider | Services |
|---|---|
| AWS | IAM, VPC, PrivateLink, Security Hub, GuardDuty |
| Azure | Entra ID (Conditional Access), Private Link, Defender for Cloud |
| GCP | BeyondCorp Enterprise, IAP (Identity-Aware Proxy), VPC Service Controls |

---

## Summary of Cloud Security Services by Provider

| Category | AWS | Azure | GCP |
|---|---|---|---|
| **IAM** | AWS IAM | Microsoft Entra ID + Azure RBAC | Google Cloud IAM |
| **MFA** | AWS MFA (virtual, hardware) | Entra ID MFA | Google 2-Step Verification |
| **Secrets** | Secrets Manager, Parameter Store | Azure Key Vault | Secret Manager |
| **Encryption Keys** | AWS KMS | Azure Key Vault | Cloud KMS |
| **WAF** | AWS WAF | Azure WAF | Cloud Armor |
| **DDoS** | AWS Shield | Azure DDoS Protection | Cloud Armor |
| **Audit Logging** | CloudTrail | Activity Log, Monitor | Cloud Audit Logs |
| **Compliance** | AWS Config, Security Hub | Azure Policy, Defender for Cloud | Security Command Centre |
| **Threat Detection** | GuardDuty | Defender for Cloud | Security Command Centre |

→ Provider-specific deep dives: [[AWS IAM]], [[Azure IAM]], [[AWS Security]], [[Azure Security]]

---

## What's Next?

- Understand the network layer: [[Cloud Networking Basics]]
- Revisit the big picture: [[Cloud Fundamentals]]
- Deep dive into provider-specific IAM: [[AWS IAM]] / [[Azure IAM]]
- Deep dive into provider-specific security: [[AWS Security]] / [[Azure Security]]

---

> [!success] Summary
> Cloud security is built on IAM (controlling who can do what), encryption (protecting data at rest and in transit), secrets management (never hardcoding credentials), network security (firewalls, WAF, DDoS protection), and compliance (meeting regulatory requirements). The most important principles are: enable MFA everywhere, apply least privilege, encrypt everything, never hardcode secrets, and always enable logging. The Zero Trust model takes this further by assuming no one is trusted by default.
