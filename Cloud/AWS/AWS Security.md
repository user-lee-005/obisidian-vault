# AWS Security

> **Summary**: Security in AWS is a shared responsibility — AWS secures the infrastructure, YOU secure your applications, data, and access. This note covers encryption (KMS), secrets management, firewalls (WAF/Shield), threat detection (GuardDuty), vulnerability scanning (Inspector), and a comprehensive security checklist.

---

## Table of Contents

- [[#Shared Responsibility Model]]
- [[#AWS KMS (Key Management Service)]]
- [[#AWS Secrets Manager]]
- [[#Secrets Manager vs Parameter Store]]
- [[#AWS WAF (Web Application Firewall)]]
- [[#AWS Shield (DDoS Protection)]]
- [[#AWS Security Hub]]
- [[#AWS GuardDuty]]
- [[#AWS Inspector]]
- [[#VPC Security]]
- [[#Security Best Practices Checklist]]

---

## Shared Responsibility Model

> **The #1 Most Important Concept in AWS Security**: AWS and you SHARE the responsibility for security. Neither side does everything.

```
┌────────────────────────────────────────────────────────┐
│                   YOUR RESPONSIBILITY                   │
│            "Security IN the Cloud"                       │
│                                                         │
│  ✅ Your data and encryption                             │
│  ✅ Your application code and configuration              │
│  ✅ IAM users, roles, permissions (see [[AWS IAM]])      │
│  ✅ Operating system patches (on EC2)                    │
│  ✅ Network configuration (Security Groups, NACLs)       │
│  ✅ Firewall rules                                       │
├─────────────────────────────────────────────────────────┤
│                   AWS'S RESPONSIBILITY                   │
│            "Security OF the Cloud"                       │
│                                                         │
│  ✅ Physical data center security                        │
│  ✅ Hardware and networking infrastructure               │
│  ✅ Hypervisor and virtualization layer                  │
│  ✅ Managed service infrastructure (RDS, Lambda, etc.)   │
│  ✅ Global network backbone                              │
└─────────────────────────────────────────────────────────┘
```

### Think of it Like Renting an Apartment

- **Landlord (AWS)**: Building structure, plumbing, electrical, locks on the front door
- **Tenant (You)**: What's inside your apartment — furniture, valuables, who you give keys to, whether you lock the door

> 💡 **Key takeaway**: If you leave your S3 bucket open to the public, that's YOUR fault, not AWS's. AWS gave you the lock — you just didn't use it.

### Responsibility Changes by Service Type

| Service Type | Your Responsibility | AWS Responsibility |
|-------------|--------------------|--------------------|
| **IaaS (EC2)** | Everything above the hypervisor: OS, patching, firewall, app | Hardware, networking, hypervisor |
| **PaaS (Beanstalk)** | Application code, data | OS, patching, platform, hardware |
| **SaaS (S3, DynamoDB)** | Data, access control | Everything else |

> The more "managed" a service, the less YOU have to secure. This is one reason to prefer managed services.

---

## AWS KMS (Key Management Service)

> **What is KMS?** A managed service to create, manage, and use encryption keys. When you want to encrypt data in AWS (S3 files, database storage, EBS volumes), KMS provides the keys.

### Why Encryption Matters

```
Without encryption:
  Data on disk: "username: admin, password: MyP@ss123"
  Anyone with disk access can read it ❌

With encryption:
  Data on disk: "a8f3k2...x9z1" (gibberish)
  Need the encryption key to decrypt ✅
```

### Types of KMS Keys

| Type | Managed By | Rotation | Cost | Use Case |
|------|-----------|----------|------|----------|
| **AWS Managed** | AWS | Automatic (yearly) | Free | S3, RDS default encryption |
| **Customer Managed** | You | Optional (configure) | $1/month + API calls | Custom encryption policies |
| **AWS Owned** | AWS | Internal | Free | Used internally by services |

### Hands-On: KMS Operations

```bash
# Create a Customer Managed Key (CMK)
aws kms create-key \
  --description "My encryption key for production data" \
  --key-usage ENCRYPT_DECRYPT \
  --origin AWS_KMS

# The response includes a KeyId — save it!
# Example: "KeyId": "1234abcd-12ab-34cd-56ef-1234567890ab"

# Create a human-readable alias for the key
aws kms create-alias \
  --alias-name alias/my-production-key \
  --target-key-id 1234abcd-12ab-34cd-56ef-1234567890ab

# List all your KMS keys
aws kms list-keys

# Describe a specific key
aws kms describe-key --key-id alias/my-production-key
```

### Encrypting and Decrypting Data

```bash
# Encrypt a small piece of data (up to 4KB)
aws kms encrypt \
  --key-id alias/my-production-key \
  --plaintext fileb://secret.txt \
  --output text \
  --query CiphertextBlob | base64 --decode > encrypted.dat

# Decrypt the data
aws kms decrypt \
  --ciphertext-blob fileb://encrypted.dat \
  --output text \
  --query Plaintext | base64 --decode > decrypted.txt

# Generate a data key (for encrypting large files)
# This returns both a plaintext key (use it to encrypt) and an encrypted copy (store this)
aws kms generate-data-key \
  --key-id alias/my-production-key \
  --key-spec AES_256
```

### Envelope Encryption (How KMS Works at Scale)

> For large data, KMS uses "envelope encryption" — KMS generates a data key, you encrypt your data with the data key, then KMS encrypts the data key itself.

```
1. You ask KMS: "Give me a data key"
2. KMS returns:
   - Plaintext data key (use this to encrypt your file)
   - Encrypted data key (store this alongside your encrypted file)
3. You encrypt your file with the plaintext data key
4. You DELETE the plaintext data key from memory
5. To decrypt: Send the encrypted data key to KMS → get plaintext key back → decrypt file
```

**Why?** KMS can only encrypt 4KB directly. For large files, this two-step process is needed. AWS services (S3, EBS, RDS) do this automatically behind the scenes.

### KMS Key Rotation

```bash
# Enable automatic key rotation (rotates every year)
aws kms enable-key-rotation --key-id alias/my-production-key

# Check if rotation is enabled
aws kms get-key-rotation-status --key-id alias/my-production-key
```

### Where KMS Is Used in AWS

| Service | What KMS Encrypts |
|---------|-------------------|
| **S3** | Objects (files) at rest — see [[AWS Storage]] |
| **EBS** | EC2 disk volumes — see [[AWS Compute]] |
| **RDS** | Database storage and backups — see [[AWS Database]] |
| **DynamoDB** | Table data at rest |
| **Lambda** | Environment variables |
| **SQS** | Messages in queues |
| **SNS** | Messages in topics |

---

## AWS Secrets Manager

> **What**: A secure vault for storing secrets — database passwords, API keys, tokens, certificates. Secrets Manager can also **automatically rotate** secrets (especially useful for RDS database passwords).

### The Problem

```
❌ Bad: Hardcode secrets in your code
   DB_PASSWORD = "MyP@ss123"   # Anyone who sees your code has your password!

❌ Bad: Store in environment variables (better, but still risky)
   export DB_PASSWORD="MyP@ss123"   # Visible in process listings

✅ Good: Store in Secrets Manager, retrieve at runtime
   password = get_secret("MyDBPassword")   # Stored encrypted, access-controlled
```

### Hands-On: Secrets Manager

```bash
# Store a new secret
aws secretsmanager create-secret \
  --name MyApp/DatabasePassword \
  --description "Production database credentials" \
  --secret-string '{"username":"admin","password":"MyP@ss123","host":"mydb.rds.amazonaws.com","port":"5432"}'

# Retrieve a secret
aws secretsmanager get-secret-value --secret-id MyApp/DatabasePassword

# The secret string is in the response:
# "SecretString": "{\"username\":\"admin\",\"password\":\"MyP@ss123\",...}"

# Update a secret
aws secretsmanager update-secret \
  --secret-id MyApp/DatabasePassword \
  --secret-string '{"username":"admin","password":"NewP@ss456","host":"mydb.rds.amazonaws.com","port":"5432"}'

# Delete a secret (has a 7-30 day recovery window)
aws secretsmanager delete-secret \
  --secret-id MyApp/DatabasePassword \
  --recovery-window-in-days 7

# List all secrets
aws secretsmanager list-secrets
```

### Automatic Secret Rotation

> **The killer feature**: Secrets Manager can automatically change your RDS database password on a schedule (e.g., every 30 days) — and update the secret at the same time, so your app always gets the current password.

```bash
# Enable automatic rotation for an RDS secret
aws secretsmanager rotate-secret \
  --secret-id MyApp/DatabasePassword \
  --rotation-lambda-arn arn:aws:lambda:ap-south-1:123456789012:function:SecretsRotation \
  --rotation-rules AutomaticallyAfterDays=30
```

### Retrieving Secrets in Application Code (Python Example)

```python
import boto3
import json

def get_secret(secret_name):
    client = boto3.client('secretsmanager', region_name='ap-south-1')
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response['SecretString'])

# Usage
creds = get_secret('MyApp/DatabasePassword')
db_host = creds['host']
db_user = creds['username']
db_pass = creds['password']
```

---

## Secrets Manager vs Parameter Store

Both store configuration and secrets, but they serve different purposes:

| Feature | Secrets Manager | SSM Parameter Store |
|---------|----------------|---------------------|
| **Primary use** | Secrets (passwords, API keys) | Configuration + secrets |
| **Automatic rotation** | ✅ Yes (built-in for RDS) | ❌ No |
| **Encryption** | Always encrypted (KMS) | Optional (SecureString type) |
| **Cost** | $0.40/secret/month + API calls | Free (Standard tier) |
| **Max size** | 64 KB | 4 KB (Standard) / 8 KB (Advanced) |
| **Versioning** | ✅ Built-in | ✅ Built-in |
| **Cross-account access** | ✅ Yes | Limited |

### When to Use Which

```
Use Secrets Manager when:
  ✅ Storing database passwords that need rotation
  ✅ Storing API keys for third-party services
  ✅ Cross-account secret sharing
  ✅ Compliance requires automatic rotation

Use Parameter Store when:
  ✅ Storing application configuration (feature flags, URLs)
  ✅ Cost-sensitive (it's free!)
  ✅ Simple key-value configuration
  ✅ Hierarchical parameter organization (/app/prod/db/host)
```

```bash
# === SSM Parameter Store Examples ===

# Store a plain text parameter (free)
aws ssm put-parameter \
  --name /my-app/prod/api-url \
  --value "https://api.example.com" \
  --type String

# Store an encrypted parameter (SecureString)
aws ssm put-parameter \
  --name /my-app/prod/db-password \
  --value "MyP@ss123" \
  --type SecureString

# Retrieve a parameter
aws ssm get-parameter --name /my-app/prod/api-url

# Retrieve and decrypt a SecureString
aws ssm get-parameter --name /my-app/prod/db-password --with-decryption

# Get all parameters under a path
aws ssm get-parameters-by-path --path /my-app/prod/ --recursive --with-decryption
```

---

## AWS WAF (Web Application Firewall)

> **What**: A firewall specifically for web applications. It inspects HTTP/HTTPS requests and blocks malicious ones before they reach your application.

### What WAF Protects Against

| Attack | Description | WAF Rule |
|--------|------------|----------|
| **SQL Injection** | Malicious SQL in user input | SQLi rule set |
| **Cross-Site Scripting (XSS)** | Injecting scripts into web pages | XSS rule set |
| **Bad Bots** | Scrapers, credential stuffers | Bot Control |
| **DDoS (Layer 7)** | Flooding with HTTP requests | Rate-based rules |
| **IP Reputation** | Known malicious IPs | IP reputation list |

### WAF Architecture

```
User Request → CloudFront / ALB / API Gateway → WAF (inspects) → Your Application
                                                  │
                                                  ├── Allow ✅
                                                  ├── Block ❌
                                                  └── Count 📊 (monitor only)
```

### Key Concepts

- **Web ACL** (Access Control List) — The main resource. Contains rules.
- **Rules** — Individual checks (e.g., "block if SQL injection detected")
- **Rule Groups** — Bundled sets of rules (AWS provides managed rule groups)
- **IP Sets** — Lists of IP addresses to allow or block

### Where to Attach WAF

| Service | How |
|---------|-----|
| **CloudFront** | Global protection for your CDN |
| **Application Load Balancer** | Protect your ALB directly |
| **API Gateway** | Protect your REST APIs |
| **AppSync** | Protect your GraphQL APIs |

### Console Walkthrough: Setting Up WAF

1. Go to **WAF & Shield** in the Console
2. Click **Create web ACL**
3. Name: `my-app-waf`, Resource: Select your ALB
4. **Add managed rule groups** (AWS provides these):
   - ✅ `AWSManagedRulesCommonRuleSet` — Covers OWASP Top 10
   - ✅ `AWSManagedRulesSQLiRuleSet` — SQL injection protection
   - ✅ `AWSManagedRulesKnownBadInputsRuleSet` — Known attack patterns
5. **Add a rate-based rule**: Block IPs making > 2000 requests per 5 minutes
6. **Default action**: Allow (block only what matches rules)
7. Review and create

### WAF via CLI

```bash
# Create an IP set (to block specific IPs)
aws wafv2 create-ip-set \
  --name blocked-ips \
  --scope REGIONAL \
  --ip-address-version IPV4 \
  --addresses "1.2.3.4/32" "5.6.7.8/32"

# List web ACLs
aws wafv2 list-web-acls --scope REGIONAL

# Get details of a web ACL
aws wafv2 get-web-acl --name my-app-waf --scope REGIONAL --id <web-acl-id>
```

---

## AWS Shield (DDoS Protection)

> **What**: Protection against Distributed Denial of Service (DDoS) attacks — when attackers flood your infrastructure with traffic to make it unavailable.

### Shield Standard vs Shield Advanced

| Feature | Shield Standard | Shield Advanced |
|---------|----------------|-----------------|
| **Cost** | **Free** (automatic) | ~$3,000/month |
| **Protection** | Layer 3/4 (network/transport) | Layer 3/4/7 (includes application) |
| **Detection** | Basic | Advanced, ML-based |
| **Response team** | Self-service | 24/7 AWS DDoS Response Team (DRT) |
| **Cost protection** | ❌ | ✅ Refund for DDoS scaling costs |
| **Visibility** | Basic | Detailed attack diagnostics |
| **WAF integration** | ❌ | ✅ Free WAF for Shield-protected resources |

### Shield Standard

- **Automatically enabled** for ALL AWS customers at no extra cost
- Protects against most common network-level DDoS attacks
- Works with CloudFront, Route 53, and Elastic Load Balancing

### Shield Advanced

- For mission-critical applications that need maximum protection
- Provides real-time attack visibility and diagnostics
- Access to AWS DDoS Response Team (DRT) — they help during attacks
- **Cost protection**: AWS refunds charges if DDoS causes auto-scaling

```bash
# Check if Shield Advanced is enabled
aws shield describe-subscription

# List protected resources (Shield Advanced)
aws shield list-protections
```

---

## AWS Security Hub

> **What**: A central dashboard that aggregates security findings from multiple AWS security services. Instead of checking GuardDuty, Inspector, IAM Access Analyzer, and others separately, Security Hub shows everything in one place.

### What Security Hub Aggregates

```
┌───────────────────────────────────────────────────────┐
│                 AWS Security Hub                       │
│            (Central Security Dashboard)                │
│                                                        │
│  Inputs:                                               │
│  ├── GuardDuty findings (threats)                      │
│  ├── Inspector findings (vulnerabilities)              │
│  ├── IAM Access Analyzer (access issues)               │
│  ├── Firewall Manager (firewall compliance)            │
│  ├── Macie (sensitive data exposure)                   │
│  └── Third-party tools (Qualys, CrowdStrike, etc.)    │
│                                                        │
│  Outputs:                                              │
│  ├── Unified findings list                             │
│  ├── Security score                                    │
│  ├── Compliance status (CIS, PCI DSS, etc.)            │
│  └── Automated remediation (via EventBridge)           │
└───────────────────────────────────────────────────────┘
```

### Compliance Standards

Security Hub checks your AWS environment against industry standards:

| Standard | What It Checks |
|----------|---------------|
| **CIS AWS Foundations** | Best practice security controls |
| **PCI DSS** | Payment card industry requirements |
| **AWS Foundational Best Practices** | AWS-recommended security checks |
| **NIST 800-53** | US government security framework |

### Enabling Security Hub

```bash
# Enable Security Hub
aws securityhub enable-security-hub --enable-default-standards

# Get your security score
aws securityhub get-findings --filters '{"SeverityLabel": [{"Value": "CRITICAL", "Comparison": "EQUALS"}]}'

# List enabled standards
aws securityhub get-enabled-standards

# Get findings summary
aws securityhub get-findings --max-results 10
```

---

## AWS GuardDuty

> **What**: An intelligent threat detection service. It continuously monitors your AWS environment for malicious activity and unauthorized behavior — like a security guard watching your account 24/7.

### What GuardDuty Monitors

| Data Source | What It Looks For |
|-------------|------------------|
| **CloudTrail Logs** | Unusual API calls, privilege escalation |
| **VPC Flow Logs** | Port scanning, data exfiltration, communication with known bad IPs |
| **DNS Logs** | Connections to crypto-mining or command-and-control servers |
| **S3 Data Events** | Unusual access patterns to S3 buckets |
| **EKS Audit Logs** | Kubernetes-level threats |

### Types of Findings

GuardDuty categorizes threats by severity:

| Severity | Example Findings |
|----------|-----------------|
| **Critical/High** | EC2 instance communicating with a known malware server |
| **Medium** | IAM user making API calls from an unusual location |
| **Low** | Port scan detected from an EC2 instance |

### Hands-On

```bash
# Enable GuardDuty
aws guardduty create-detector --enable

# List detectors
aws guardduty list-detectors

# Get findings (threats detected)
aws guardduty list-findings --detector-id <detector-id>

# Get details of specific findings
aws guardduty get-findings \
  --detector-id <detector-id> \
  --finding-ids <finding-id>
```

### What GuardDuty Detects (Real Examples)

- 🔴 **UnauthorizedAccess:EC2/MaliciousIPCaller** — EC2 communicating with a known bad IP
- 🔴 **CryptoCurrency:EC2/BitcoinTool** — EC2 mining cryptocurrency (often means compromised)
- 🟡 **Recon:EC2/PortProbeUnprotectedPort** — Someone probing open ports on your instance
- 🟡 **UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B** — Login from unusual location
- 🟢 **Behavior:EC2/TrafficVolumeUnusual** — Unusual network traffic pattern

---

## AWS Inspector

> **What**: Automated vulnerability scanning. It checks your EC2 instances and container images (ECR) for known vulnerabilities (CVEs), network exposure, and software issues.

### What Inspector Scans

| Target | What It Checks |
|--------|---------------|
| **EC2 Instances** | OS vulnerabilities, network reachability, package CVEs |
| **ECR Container Images** | Known vulnerabilities in Docker image layers |
| **Lambda Functions** | Vulnerabilities in code dependencies |

### How Inspector Works

1. **Enable Inspector** (it auto-discovers your resources)
2. **Inspector scans** continuously (not just once)
3. **Findings** appear in Inspector and Security Hub
4. **You remediate** — patch the software, update the container image

```bash
# Enable Inspector
aws inspector2 enable --resource-types EC2 ECR LAMBDA

# List findings (vulnerabilities)
aws inspector2 list-findings --max-results 10

# Get coverage (what's being scanned)
aws inspector2 list-coverage --max-results 10

# Get finding details
aws inspector2 get-findings-report-status
```

### Inspector Findings Example

```
Finding: CVE-2023-44487 (HTTP/2 Rapid Reset Attack)
Severity: HIGH
Package: nginx 1.24.0
Fix: Update to nginx 1.25.3
Resource: i-0123456789abcdef0
```

> **Action**: SSH into the instance, update nginx, or better yet — update your AMI and redeploy.

---

## VPC Security

> VPC security is your network-level defense. It controls what traffic can reach your instances and what traffic can leave. For full VPC details, see [[AWS Networking]].

### Security Groups (Instance-Level Firewall)

- **Stateful**: If you allow inbound traffic, the response is automatically allowed out
- **Attached to**: EC2 instances, RDS databases, Lambda (in VPC), etc.
- **Default**: All inbound DENIED, all outbound ALLOWED

```bash
# Create a security group
aws ec2 create-security-group \
  --group-name web-server-sg \
  --description "Security group for web servers" \
  --vpc-id vpc-xxx

# Allow HTTP from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Allow SSH from a specific IP only
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx \
  --protocol tcp \
  --port 22 \
  --cidr 203.0.113.0/32

# View security group rules
aws ec2 describe-security-groups --group-ids sg-xxx
```

### Network ACLs (Subnet-Level Firewall)

- **Stateless**: Must explicitly allow both inbound AND outbound
- **Attached to**: Subnets
- **Processed in order**: Rules have numbers, processed lowest to highest
- **Default**: All traffic ALLOWED (modify to restrict)

| Feature | Security Group | NACL |
|---------|---------------|------|
| **Level** | Instance | Subnet |
| **Stateful/Stateless** | Stateful | Stateless |
| **Rules** | Allow only | Allow AND Deny |
| **Evaluation** | All rules evaluated | Rules evaluated in order |
| **Default** | Deny all inbound | Allow all |

### VPC Flow Logs

> **What**: Capture information about all network traffic flowing in and out of your VPC. Essential for security analysis and troubleshooting.

```bash
# Create VPC Flow Logs (send to CloudWatch Logs)
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-xxx \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name VPCFlowLogs \
  --deliver-logs-permission-arn arn:aws:iam::123456789012:role/VPCFlowLogsRole

# Create Flow Logs (send to S3 — cheaper for long-term storage)
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-xxx \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::my-flow-logs-bucket

# View existing flow logs
aws ec2 describe-flow-logs
```

### Flow Log Record Example

```
2 123456789012 eni-abc123 10.0.1.5 203.0.113.50 443 49152 6 10 840 1616729292 1616729349 ACCEPT OK
```

| Field | Value | Meaning |
|-------|-------|---------|
| Account | 123456789012 | Your AWS account |
| Interface | eni-abc123 | Network interface |
| Source IP | 10.0.1.5 | Internal IP |
| Dest IP | 203.0.113.50 | External IP |
| Dest Port | 443 | HTTPS |
| Protocol | 6 | TCP |
| Action | ACCEPT | Traffic was allowed |

### Analyzing Flow Logs with CloudWatch Logs Insights

```sql
# Find rejected traffic (potential attacks)
fields @timestamp, srcAddr, dstAddr, dstPort, action
| filter action = "REJECT"
| stats count(*) as rejections by srcAddr
| sort rejections desc
| limit 10
```

> See [[AWS Monitoring]] for more on CloudWatch Logs Insights.

---

## Security Best Practices Checklist

> **Use this checklist for every AWS account and workload.** Tick these off before going to production.

### Identity and Access (see [[AWS IAM]])

- ✅ **Enable MFA on root account** — Non-negotiable. Do this FIRST.
- ✅ **Enable MFA on all IAM users** — Especially those with console access.
- ✅ **Never use root account for daily work** — Create IAM users or use SSO.
- ✅ **Use IAM roles for services** — EC2 instances, Lambda functions, ECS tasks should use roles, NOT access keys.
- ✅ **Follow least privilege** — Give minimum permissions needed. Start with zero and add.
- ✅ **Review IAM Access Analyzer** — Identifies resources shared outside your account.
- ✅ **Rotate access keys regularly** — If you must use access keys, rotate every 90 days.
- ✅ **Use IAM Identity Center (SSO)** — For organizations with multiple accounts.

### Encryption

- ✅ **Encrypt all data at rest** — S3 (SSE-S3 or SSE-KMS), EBS (encrypted volumes), RDS (storage encryption)
- ✅ **Encrypt all data in transit** — Use HTTPS/TLS everywhere. No exceptions.
- ✅ **Use KMS Customer Managed Keys** for sensitive workloads — gives you rotation and audit control.
- ✅ **Enable S3 default encryption** — Every object encrypted automatically.
- ✅ **Enable EBS default encryption** — Every new volume encrypted automatically.

```bash
# Enable EBS encryption by default in a region
aws ec2 enable-ebs-encryption-by-default

# Enable S3 default encryption on a bucket
aws s3api put-bucket-encryption --bucket my-bucket --server-side-encryption-configuration '{
  "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "aws:kms"}}]
}'
```

### Secrets Management

- ✅ **Never hardcode secrets** in source code, environment variables, or config files.
- ✅ **Use Secrets Manager** for database passwords and API keys.
- ✅ **Enable automatic rotation** for database credentials.
- ✅ **Use Parameter Store** for non-sensitive configuration.

### Network Security (see [[AWS Networking]])

- ✅ **Use private subnets** for databases and backend services.
- ✅ **Use NAT Gateway** for private instances that need internet access.
- ✅ **Restrict Security Group rules** — No `0.0.0.0/0` on SSH (port 22) in production!
- ✅ **Enable VPC Flow Logs** — Monitor all network traffic.
- ✅ **Use VPC Endpoints** for AWS service access (keeps traffic private).

### Detection and Monitoring (see [[AWS Monitoring]])

- ✅ **Enable CloudTrail in ALL regions** — Audit every API call.
- ✅ **Enable CloudTrail log file validation** — Detect tampering.
- ✅ **Enable GuardDuty** — Continuous threat detection.
- ✅ **Enable Security Hub** — Centralized security findings.
- ✅ **Enable AWS Config** — Track configuration changes and compliance.
- ✅ **Set up CloudWatch Alarms** — For security-related metrics.
- ✅ **Set up SNS alerts** — Get notified of critical findings.

### Application Security

- ✅ **Use WAF** on all public-facing web applications.
- ✅ **Enable Shield Advanced** for critical applications (if budget allows).
- ✅ **Use AWS Inspector** for vulnerability scanning.
- ✅ **Scan container images** before deployment (ECR image scanning).
- ✅ **Keep all software patched** and updated.

### Backup and Recovery

- ✅ **Enable versioning** on S3 buckets (protection against accidental deletion).
- ✅ **Enable automated backups** on RDS.
- ✅ **Test restore procedures** regularly — backups are useless if you can't restore.
- ✅ **Use AWS Backup** for centralized backup management.

### Account-Level Security

- ✅ **Enable AWS Organizations** for multi-account management.
- ✅ **Use Service Control Policies (SCPs)** to restrict what accounts can do.
- ✅ **Enable billing alerts** — Unexpected charges can indicate compromised resources.
- ✅ **Tag all resources** for cost tracking and access control.

---

## Security Services Summary

| Service | Purpose | Cost |
|---------|---------|------|
| **IAM** | Identity and access management | Free |
| **KMS** | Encryption key management | $1/key/month |
| **Secrets Manager** | Store and rotate secrets | $0.40/secret/month |
| **Parameter Store** | Configuration storage | Free (Standard) |
| **WAF** | Web application firewall | $5/ACL/month + rules |
| **Shield Standard** | Basic DDoS protection | Free |
| **Shield Advanced** | Enhanced DDoS protection | ~$3,000/month |
| **Security Hub** | Central security dashboard | Per finding |
| **GuardDuty** | Threat detection | Per event volume |
| **Inspector** | Vulnerability scanning | Per scan |
| **CloudTrail** | API audit logging | Free (management events) |
| **Config** | Configuration compliance | Per rule evaluation |
| **Macie** | Sensitive data discovery | Per GB scanned |

---

## Quick Reference: Security Commands Cheat Sheet

```bash
# === KMS ===
aws kms create-key --description "My key"
aws kms create-alias --alias-name alias/my-key --target-key-id <key-id>
aws kms encrypt --key-id alias/my-key --plaintext fileb://file.txt
aws kms decrypt --ciphertext-blob fileb://encrypted.dat

# === Secrets Manager ===
aws secretsmanager create-secret --name MySecret --secret-string '{"key":"value"}'
aws secretsmanager get-secret-value --secret-id MySecret
aws secretsmanager rotate-secret --secret-id MySecret

# === Parameter Store ===
aws ssm put-parameter --name /app/config --value "value" --type String
aws ssm get-parameter --name /app/config --with-decryption

# === GuardDuty ===
aws guardduty create-detector --enable
aws guardduty list-findings --detector-id <id>

# === Security Hub ===
aws securityhub enable-security-hub --enable-default-standards
aws securityhub get-findings --max-results 10

# === Inspector ===
aws inspector2 enable --resource-types EC2 ECR LAMBDA
aws inspector2 list-findings --max-results 10

# === VPC Flow Logs ===
aws ec2 create-flow-logs --resource-type VPC --resource-ids vpc-xxx --traffic-type ALL --log-destination-type cloud-watch-logs --log-group-name VPCFlowLogs

# === Security Groups ===
aws ec2 create-security-group --group-name my-sg --description "My SG" --vpc-id vpc-xxx
aws ec2 authorize-security-group-ingress --group-id sg-xxx --protocol tcp --port 443 --cidr 0.0.0.0/0
```

---

## What to Learn Next

- [[Cloud Security Fundamentals]] — General cloud security concepts
- [[AWS IAM]] — Deep dive into users, roles, policies, and permissions
- [[AWS Networking]] — VPC, subnets, security groups, NACLs
- [[AWS Monitoring]] — CloudWatch, CloudTrail, alarms, and dashboards
- [[AWS Deployment]] — Secure deployment pipelines
- [[Azure Security]] — Compare with Azure Defender and Key Vault

---

> 📝 **Created for**: AWS Cloud Learning Path
> 🏷️ Tags: #aws #security #kms #encryption #waf #shield #guardduty #inspector #secrets-manager #vpc-security
