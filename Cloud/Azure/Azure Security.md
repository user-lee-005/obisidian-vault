# Azure Security

> **Beginner-friendly guide** to securing your Azure environment — from managing secrets with Key Vault to enforcing policies, protecting against threats, and building a secure-by-default posture.

---

## Table of Contents

- [[#Azure Security Overview]]
- [[#Azure Key Vault]]
- [[#Microsoft Defender for Cloud]]
- [[#Azure WAF (Web Application Firewall)]]
- [[#Azure DDoS Protection]]
- [[#Azure Firewall]]
- [[#Azure Policy]]
- [[#Azure Blueprints and Landing Zones]]
- [[#Network Security]]
- [[#Azure vs AWS Security Comparison]]
- [[#Security Best Practices Checklist]]

---

## Azure Security Overview

### The Shared Responsibility Model

Security in the cloud is a **shared responsibility** between you and Microsoft. What each party is responsible for depends on the service type:

```
                    On-Premises    IaaS (VMs)    PaaS (App Svc)    SaaS (M365)
                    ──────────     ──────────    ──────────────    ───────────
Data & Access       YOU            YOU           YOU               YOU
Applications        YOU            YOU           YOU               Microsoft
OS / Runtime        YOU            YOU           Microsoft         Microsoft
Network Controls    YOU            YOU           Microsoft         Microsoft
Physical Security   YOU            Microsoft     Microsoft         Microsoft
```

**The simple rule:**
- **Azure secures the platform** (physical data centres, network infrastructure, hypervisor)
- **You secure your stuff** (data, identities, access, application code, configurations)

> 💡 **Key Insight:** The higher up the service model (IaaS → PaaS → SaaS), the MORE Azure handles for you. This is why PaaS services like App Service are often more secure out of the box than VMs.

### Azure Security Services at a Glance

| Service | What It Does | AWS Equivalent |
|---------|-------------|----------------|
| **Key Vault** | Manage secrets, keys, certificates | Secrets Manager + KMS |
| **Defender for Cloud** | Security posture + threat protection | Security Hub + GuardDuty |
| **Azure WAF** | Web application firewall | AWS WAF |
| **DDoS Protection** | Network DDoS mitigation | AWS Shield |
| **Azure Firewall** | Managed network firewall | AWS Network Firewall |
| **Azure Policy** | Enforce governance rules | AWS Config + SCP |
| **Entra ID (AAD)** | Identity & access management | IAM + Cognito |
| **NSGs** | Network-level access control | Security Groups |
| **Azure Bastion** | Secure VM access (no public IP) | SSM Session Manager |

---

## Azure Key Vault

Key Vault is Azure's **centralized secret management service**. It stores three types of sensitive data:

1. **Secrets** — Connection strings, API keys, passwords
2. **Keys** — Encryption keys (RSA, EC) for data encryption
3. **Certificates** — SSL/TLS certificates for HTTPS

### Why Use Key Vault?

**Without Key Vault (❌ Bad):**
```python
# Hardcoded secrets in code — anyone with repo access can see them!
DB_PASSWORD = "MySecretP@ss123!"
API_KEY = "sk-12345abcdef"
```

**With Key Vault (✅ Good):**
```python
# Secrets stored securely in Key Vault, accessed at runtime
from azure.keyvault.secrets import SecretClient
client = SecretClient(vault_url="https://mykeyvault123.vault.azure.net/", credential=credential)
db_password = client.get_secret("DatabasePassword").value
```

### Creating and Managing Key Vault

```bash
# Create a Key Vault
az keyvault create \
  --name mykeyvault123 \
  --resource-group myResourceGroup \
  --location centralindia

# ============================================
# SECRETS — Store sensitive strings
# ============================================

# Store a secret
az keyvault secret set \
  --vault-name mykeyvault123 \
  --name DatabasePassword \
  --value "MySecretP@ss123!"

# Retrieve a secret
az keyvault secret show \
  --vault-name mykeyvault123 \
  --name DatabasePassword \
  --query value \
  --output tsv

# List all secrets
az keyvault secret list --vault-name mykeyvault123 --output table

# Delete a secret (soft-delete — recoverable!)
az keyvault secret delete --vault-name mykeyvault123 --name DatabasePassword

# Recover a deleted secret
az keyvault secret recover --vault-name mykeyvault123 --name DatabasePassword

# ============================================
# KEYS — Encryption keys
# ============================================

# Create an RSA encryption key
az keyvault key create \
  --vault-name mykeyvault123 \
  --name MyEncryptionKey \
  --kty RSA \
  --size 2048

# List keys
az keyvault key list --vault-name mykeyvault123 --output table

# ============================================
# ACCESS CONTROL
# ============================================

# Grant a user access to secrets (using access policies)
az keyvault set-policy \
  --name mykeyvault123 \
  --upn user@domain.com \
  --secret-permissions get list set

# Grant an app/service principal access
az keyvault set-policy \
  --name mykeyvault123 \
  --spn <app-id> \
  --secret-permissions get list
```

### Access Policies vs RBAC for Key Vault

Key Vault supports **two access models**:

| Feature | Access Policies (Legacy) | RBAC (Recommended) |
|---------|------------------------|---------------------|
| **Granularity** | Vault-level | Individual secret/key level |
| **Management** | Per-vault configuration | Centralized in Entra ID |
| **Audit** | Basic | Full Azure RBAC audit trail |
| **Recommendation** | Existing vaults | New deployments |

```bash
# Switch Key Vault to RBAC mode
az keyvault update \
  --name mykeyvault123 \
  --resource-group myResourceGroup \
  --enable-rbac-authorization true

# Assign RBAC role for Key Vault
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee user@domain.com \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.KeyVault/vaults/mykeyvault123
```

### Soft-Delete and Purge Protection

Key Vault has built-in safety nets to prevent accidental data loss:

- **Soft-delete** (enabled by default) — Deleted secrets/keys go to a "recycle bin" for 7-90 days
- **Purge protection** — Once enabled, NOBODY can permanently delete secrets during the retention period

```bash
# Enable purge protection (cannot be disabled once enabled!)
az keyvault update \
  --name mykeyvault123 \
  --resource-group myResourceGroup \
  --enable-purge-protection true

# Set soft-delete retention period (days)
az keyvault update \
  --name mykeyvault123 \
  --resource-group myResourceGroup \
  --retention-days 90
```

### Key Vault Integration with Azure Services

The real power of Key Vault is that other Azure services can **reference secrets directly**:

**App Service — Reference Key Vault secret in app settings:**
```bash
# Instead of storing the password in app settings directly:
# ❌ az webapp config appsettings set --settings DB_PASS="MySecretP@ss123!"

# Reference Key Vault (the app never sees the actual secret in config):
# ✅ Use this format in app settings:
# @Microsoft.KeyVault(SecretUri=https://mykeyvault123.vault.azure.net/secrets/DatabasePassword/)
az webapp config appsettings set \
  --name myuniqueapp123 \
  --resource-group myResourceGroup \
  --settings "DB_PASS=@Microsoft.KeyVault(SecretUri=https://mykeyvault123.vault.azure.net/secrets/DatabasePassword/)"
```

> 💡 **Managed Identities** make this seamless — your App Service gets an identity in Entra ID, and you grant that identity access to Key Vault. No credentials needed. See [[Azure IAM]] for details.

---

## Microsoft Defender for Cloud

Defender for Cloud is Azure's **Cloud Security Posture Management (CSPM)** and **Cloud Workload Protection Platform (CWPP)**. In simpler terms: it tells you what's insecure and protects your workloads from threats.

### What It Does

1. **Security Posture Assessment** — Scans your environment and tells you what's misconfigured
2. **Secure Score** — A 0-100% score showing how secure your environment is
3. **Recommendations** — Specific, actionable steps to improve security
4. **Threat Protection** — Detects and alerts on active threats (paid plans)

### Free vs Paid Plans

| Feature | Free (Foundational CSPM) | Paid (Defender Plans) |
|---------|-------------------------|----------------------|
| **Secure Score** | ✅ | ✅ |
| **Recommendations** | ✅ Basic | ✅ Advanced |
| **Asset inventory** | ✅ | ✅ |
| **Threat detection** | ❌ | ✅ Per-resource |
| **Vulnerability scanning** | ❌ | ✅ VMs, containers, SQL |
| **Just-in-time VM access** | ❌ | ✅ |
| **Adaptive controls** | ❌ | ✅ |

**Paid plan pricing** — you enable per resource type:
- Defender for Servers (VMs)
- Defender for App Service
- Defender for SQL
- Defender for Storage
- Defender for Containers
- Defender for Key Vault

### Using Defender for Cloud

```bash
# View Defender for Cloud pricing/status for all resource types
az security pricing list --output table

# Get security recommendations
az security assessment list --output table

# Get your Secure Score
az security secure-score list --output table

# List security alerts (threat detections)
az security alert list --output table
```

### Secure Score

Secure Score is your **security report card**. A higher score means better security posture.

```
Your Secure Score: 72/100
─────────────────────────
✅ MFA enabled for all admins                          (+10 pts)
✅ Network security groups on all subnets               (+5 pts)
❌ Key Vault soft-delete not enabled                    (-3 pts)  ← Fix this!
❌ Diagnostic logs not enabled on SQL Database           (-5 pts)  ← Fix this!
❌ Public IP addresses on VMs                            (-5 pts)  ← Fix this!
```

**How to enable Defender for Cloud:**
1. Azure Portal → **Microsoft Defender for Cloud**
2. **Environment settings** → Select your subscription
3. **Enable** the plans you want (start with free tier)
4. Review **Recommendations** tab and start fixing issues

### Example: Just-in-Time (JIT) VM Access

Instead of leaving SSH/RDP ports open permanently (dangerous!), JIT allows access **only when needed** for a limited time:

```
Without JIT: Port 22 (SSH) open 24/7 → attackers can try to brute-force
With JIT:    Port 22 closed by default → you request access for 2 hours → auto-closes after
```

Enable JIT in: **Defender for Cloud → Workload protections → Just-in-time VM access**

---

## Azure WAF (Web Application Firewall)

WAF protects your web applications from common web exploits. It sits **in front of** your app and inspects every HTTP request.

### What WAF Protects Against

| Attack Type | Description | Example |
|------------|-------------|---------|
| **SQL Injection** | Malicious SQL in user input | `'; DROP TABLE users; --` |
| **Cross-Site Scripting (XSS)** | Injecting malicious scripts | `<script>steal(cookies)</script>` |
| **Bot attacks** | Automated malicious requests | Credential stuffing, scraping |
| **DDoS (Layer 7)** | Application-level floods | Millions of HTTP requests |
| **File inclusion** | Accessing server files | `../../etc/passwd` |

### Where to Deploy WAF

WAF can be deployed on three Azure services:

1. **Application Gateway** — Regional load balancer with WAF (most common)
2. **Azure Front Door** — Global CDN/load balancer with WAF
3. **Azure CDN** — Content delivery network with WAF

### WAF Rule Sets

- **OWASP Core Rule Set (CRS) 3.2** — Managed rules covering the OWASP Top 10 vulnerabilities
- **Microsoft Bot Manager** — Rules to detect and block malicious bots
- **Custom Rules** — Your own rules (e.g., block requests from specific countries, rate limit specific endpoints)

### WAF Modes

| Mode | Behaviour |
|------|-----------|
| **Detection** | Logs malicious requests but doesn't block them (safe for testing) |
| **Prevention** | Blocks malicious requests (use in production) |

> 💡 **Tip:** Always start in **Detection** mode to see what would be blocked, then switch to **Prevention** after verifying no legitimate traffic is affected.

---

## Azure DDoS Protection

DDoS (Distributed Denial of Service) attacks flood your resources with traffic to make them unavailable. Azure provides two tiers of protection:

### DDoS Protection Tiers

| Feature | Basic (Free) | Standard (Paid) |
|---------|-------------|-----------------|
| **Coverage** | All Azure resources | VNet resources |
| **Mitigation** | Automatic, always on | Adaptive tuning |
| **Metrics** | ❌ | ✅ Attack metrics |
| **Alerting** | ❌ | ✅ Attack alerts |
| **Support** | Standard | DDoS Rapid Response team |
| **Cost protection** | ❌ | ✅ Credits for scale-out costs during attack |
| **Price** | Free | ~$2,944/month per plan |

> 💡 **For most beginners:** Basic protection is included free and is sufficient. Standard is for production workloads that need guaranteed protection and cost coverage.

### How DDoS Protection Works

```
Attacker → Millions of fake requests
                │
                ▼
    ┌──────────────────────┐
    │  Azure DDoS          │  ← Detects abnormal traffic patterns
    │  Protection          │  ← Scrubs malicious traffic
    │  (Network Edge)      │  ← Lets legitimate traffic through
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │  Your Application    │  ← Only sees clean, legitimate traffic
    └──────────────────────┘
```

---

## Azure Firewall

Azure Firewall is a **managed, cloud-native network firewall**. It controls traffic flowing in and out of your Azure virtual networks.

### What Azure Firewall Does

Think of it as a security guard at the entrance of your network:

- **Application Rules** — Control outbound HTTP/HTTPS by FQDN (e.g., allow `*.github.com`, block `*.gambling.com`)
- **Network Rules** — Control traffic by IP address and port (e.g., allow TCP port 443 to `10.0.0.5`)
- **NAT Rules** — Translate public IPs to private IPs (inbound access)
- **Threat Intelligence** — Automatically block known malicious IPs and domains

### Azure Firewall vs NSGs

| Feature | NSG | Azure Firewall |
|---------|-----|---------------|
| **Level** | Subnet/NIC | Centralized (VNet) |
| **Filtering** | IP + Port only | IP + Port + FQDN + Threat Intel |
| **Logging** | Basic | Rich diagnostics |
| **Cost** | Free | ~$912/month + data processing |
| **Use case** | Basic access control | Advanced network security |

> 💡 **Use both:** NSGs for basic subnet-level filtering (free), Azure Firewall for centralized advanced filtering (when needed).

---

## Azure Policy

Azure Policy lets you **enforce rules** across your Azure environment. It ensures resources comply with your organization's standards.

### How Azure Policy Works

```
Developer tries to create a VM
        │
        ▼
Azure Policy checks: "Is this VM size allowed?"
        │
        ├── VM size is Standard_B2s (allowed) → ✅ Resource created
        └── VM size is Standard_M128s (too expensive, not allowed) → ❌ Denied!
```

### Policy Modes

| Mode | Behaviour | Use Case |
|------|-----------|----------|
| **Audit** | Allows resource but flags it as non-compliant | Start here — see what would fail |
| **Deny** | Blocks resource creation if it doesn't comply | Production enforcement |
| **DeployIfNotExists** | Auto-creates a resource if missing | Auto-enable diagnostics |
| **Modify** | Auto-adds/changes properties | Auto-add tags |

### Common Built-in Policies

| Policy | What It Does |
|--------|-------------|
| Require a tag on resources | Ensures every resource has a specific tag |
| Allowed locations | Restrict resource creation to specific regions |
| Allowed VM sizes | Prevent expensive VM sizes |
| Require encryption on storage | Ensure storage accounts use encryption |
| Require HTTPS on App Service | Ensure web apps only accept HTTPS |
| Audit VMs without managed disks | Flag VMs using legacy disks |

### Using Azure Policy

```bash
# List available policy definitions (there are hundreds!)
az policy definition list --query "[?contains(displayName, 'tag')]" --output table

# List policies related to encryption
az policy definition list --query "[?contains(displayName, 'encrypt')]" --output table

# Assign a policy to a resource group
az policy assignment create \
  --name require-tag-environment \
  --display-name "Require Environment Tag" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/<policy-id>" \
  --scope "/subscriptions/<sub-id>/resourceGroups/myResourceGroup"

# Assign a policy with parameters
az policy assignment create \
  --name allowed-locations \
  --display-name "Allowed Locations" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/e56962a6-4747-49cd-b67b-bf8b01975c4c" \
  --scope "/subscriptions/<sub-id>" \
  --params '{"listOfAllowedLocations": {"value": ["centralindia", "southindia"]}}'

# Check compliance
az policy state list --resource-group myResourceGroup --output table

# Get non-compliant resources
az policy state list \
  --resource-group myResourceGroup \
  --filter "complianceState eq 'NonCompliant'" \
  --output table
```

### Policy Initiatives (Policy Sets)

An **Initiative** is a group of policies bundled together. For example, the "CIS Microsoft Azure Foundations Benchmark" initiative includes dozens of individual policies for security compliance.

```bash
# Assign a built-in initiative
az policy assignment create \
  --name cis-benchmark \
  --display-name "CIS Azure Benchmark" \
  --policy-set-definition "/providers/Microsoft.Authorization/policySetDefinitions/<initiative-id>" \
  --scope "/subscriptions/<sub-id>"
```

---

## Azure Blueprints and Landing Zones

### Azure Blueprints

Blueprints let you define a **repeatable set of Azure resources** that comply with standards. Think of it as a template for an entire environment:

A Blueprint can include:
- Role assignments (who has access)
- Policy assignments (what rules apply)
- ARM templates (what resources to create)
- Resource groups (how to organize)

> ⚠️ **Note:** Azure Blueprints is being deprecated. Microsoft now recommends **Azure Landing Zones** with Bicep/Terraform modules instead.

### Azure Landing Zones

Landing Zones are Microsoft's **best-practice architecture** for setting up a cloud environment. They define:

- **Management groups** and **subscription structure**
- **Networking** (hub-spoke, Virtual WAN)
- **Identity** (Entra ID integration)
- **Governance** (policies, compliance)
- **Security** (Defender for Cloud, Sentinel)
- **Monitoring** (Azure Monitor, Log Analytics)

For beginners, don't worry about Landing Zones yet — they're for organizations setting up enterprise Azure environments.

---

## Network Security

This section summarizes network security concepts. For full details, see [[Azure Networking]].

### Network Security Groups (NSGs)

NSGs are **free, subnet-level firewalls** that control inbound and outbound traffic:

```bash
# Create an NSG
az network nsg create --resource-group myResourceGroup --name myNSG

# Allow SSH from a specific IP only
az network nsg rule create \
  --resource-group myResourceGroup \
  --nsg-name myNSG \
  --name AllowSSH \
  --priority 100 \
  --source-address-prefixes 203.0.113.50 \
  --destination-port-ranges 22 \
  --access Allow \
  --protocol Tcp \
  --direction Inbound

# Deny all other inbound traffic (lower priority = processed first)
az network nsg rule create \
  --resource-group myResourceGroup \
  --nsg-name myNSG \
  --name DenyAllInbound \
  --priority 4096 \
  --source-address-prefixes '*' \
  --destination-port-ranges '*' \
  --access Deny \
  --protocol '*' \
  --direction Inbound
```

### Azure Bastion — Secure VM Access

Azure Bastion provides **SSH/RDP access to VMs through the Azure portal** — no public IP needed on the VM, no VPN client required.

```
Traditional (❌ Risky):
  You → SSH over internet → VM with public IP (exposed to attacks)

Azure Bastion (✅ Secure):
  You → Azure Portal → Bastion → Private connection → VM (no public IP)
```

### Private Endpoints — Keep Traffic Off the Internet

Private Endpoints give Azure services (Storage, SQL, Key Vault) a **private IP address inside your VNet**. Traffic never goes over the public internet.

```bash
# Create a private endpoint for a storage account
az network private-endpoint create \
  --name myPrivateEndpoint \
  --resource-group myResourceGroup \
  --vnet-name myVNet \
  --subnet PrivateSubnet \
  --private-connection-resource-id <storage-account-id> \
  --group-id blob \
  --connection-name myConnection

# Verify the private endpoint
az network private-endpoint show \
  --name myPrivateEndpoint \
  --resource-group myResourceGroup \
  --output table
```

**Before Private Endpoint:**
```
Your App → Internet → Storage Account (public endpoint)
```

**After Private Endpoint:**
```
Your App → VNet → Private Endpoint (10.0.1.5) → Storage Account
                  (traffic stays inside Azure network)
```

---

## Azure vs AWS Security Comparison

| Azure Service | AWS Equivalent | Purpose |
|--------------|---------------|---------|
| **Key Vault** | Secrets Manager + KMS | Secret & key management |
| **Defender for Cloud** | Security Hub + GuardDuty + Inspector | Security posture + threat detection |
| **Azure Policy** | AWS Config + SCP (Service Control Policies) | Governance & compliance |
| **Azure WAF** | AWS WAF | Web application firewall |
| **DDoS Protection** | AWS Shield | DDoS mitigation |
| **Azure Firewall** | AWS Network Firewall | Managed network firewall |
| **NSGs** | Security Groups | Network access control |
| **Azure Bastion** | SSM Session Manager | Secure VM access |
| **Private Endpoints** | VPC Endpoints (PrivateLink) | Private access to services |
| **Entra ID** | IAM + Cognito + SSO | Identity management |
| **Azure Sentinel** | Amazon Detective + SIEM | Security analytics (SIEM/SOAR) |

### Key Differences

1. **Key Vault = Secrets Manager + KMS combined** — Azure keeps secrets and encryption keys in one service; AWS separates them
2. **Defender for Cloud** is more comprehensive out-of-the-box than any single AWS service — it combines posture management, vulnerability scanning, and threat detection
3. **Azure Policy** is more powerful than AWS Config for enforcement — it can deny, modify, and auto-remediate
4. **Entra ID** (formerly Azure AD) is a full enterprise identity platform — AWS IAM is simpler but less feature-rich

See [[AWS Security]] for the AWS side of this comparison.

---

## Security Best Practices Checklist

Use this checklist for every Azure deployment. Each item is ordered by impact and ease of implementation:

### Identity & Access

- ✅ **Enable MFA for all users** — Use Entra ID Conditional Access to require multi-factor authentication. This single step prevents 99.9% of identity attacks.
  ```
  Portal → Entra ID → Security → Conditional Access → New Policy → Require MFA
  ```
- ✅ **Use Managed Identities for services** — No hardcoded credentials! Let Azure handle authentication between services.
  ```bash
  # Enable system-assigned managed identity on an App Service
  az webapp identity assign --name myuniqueapp123 --resource-group myResourceGroup
  ```
- ✅ **Apply least privilege** — Give users/services only the permissions they need, nothing more. See [[Azure IAM]].
- ✅ **Review access regularly** — Use Entra ID Access Reviews to audit who has access to what.

### Secrets & Encryption

- ✅ **Store ALL secrets in Key Vault** — Never in code, config files, or environment variables.
- ✅ **Enable soft-delete on Key Vault** — Protect against accidental deletion.
- ✅ **Encrypt all data at rest** — Azure Storage, SQL, and most services encrypt at rest by default. Verify it's enabled.
- ✅ **Encrypt all data in transit** — Enforce HTTPS/TLS everywhere.
  ```bash
  # Require HTTPS on App Service
  az webapp update --name myuniqueapp123 --resource-group myResourceGroup --https-only true
  ```

### Network

- ✅ **Use NSGs on all subnets** — Default deny, explicitly allow only needed traffic.
- ✅ **Use Azure Bastion** instead of public IPs for VM management.
- ✅ **Use Private Endpoints** for sensitive services (Storage, SQL, Key Vault).
- ✅ **Enable DDoS Protection Standard** for production workloads (if budget allows).

### Governance & Compliance

- ✅ **Enable Azure Policy** for governance — Start with audit mode, then enforce.
- ✅ **Enable Defender for Cloud** — At minimum, the free tier for Secure Score and recommendations.
- ✅ **Review Secure Score regularly** and act on recommendations.
- ✅ **Enable diagnostic logs on all resources** — Send to Log Analytics. See [[Azure Monitoring]].
- ✅ **Tag all resources** — At minimum: `Environment`, `Owner`, `Project`, `CostCenter`.

### Operational Security

- ✅ **Enable soft-delete on storage accounts** — Protect against accidental data loss.
  ```bash
  az storage account blob-service-properties update \
    --account-name mystorageaccount \
    --resource-group myResourceGroup \
    --enable-delete-retention true \
    --delete-retention-days 30
  ```
- ✅ **Use deployment slots** for safe deployments — See [[Azure Deployment]].
- ✅ **Enable WAF** for public-facing web applications.
- ✅ **Rotate secrets and keys** regularly — Key Vault supports automatic rotation.

### Quick Security Audit Commands

```bash
# Check Secure Score
az security secure-score list --output table

# List security recommendations
az security assessment list --output table

# Check which Defender plans are enabled
az security pricing list --output table

# Find VMs with public IPs (potential risk!)
az vm list-ip-addresses --resource-group myResourceGroup --output table

# Check if Key Vault has soft-delete enabled
az keyvault show --name mykeyvault123 --query "properties.enableSoftDelete"

# Check if App Service is HTTPS only
az webapp show --name myuniqueapp123 --resource-group myResourceGroup --query httpsOnly

# List NSGs and their rules
az network nsg list --resource-group myResourceGroup --output table
az network nsg rule list --nsg-name myNSG --resource-group myResourceGroup --output table

# Check policy compliance
az policy state list --resource-group myResourceGroup --filter "complianceState eq 'NonCompliant'" --output table
```

---

## Quick Reference — Security Commands Cheat Sheet

```bash
# === Key Vault ===
az keyvault create --name <vault> -g <rg> --location centralindia
az keyvault secret set --vault-name <vault> --name <secret> --value "<value>"
az keyvault secret show --vault-name <vault> --name <secret> --query value -o tsv
az keyvault set-policy --name <vault> --upn user@domain.com --secret-permissions get list

# === Defender for Cloud ===
az security pricing list --output table
az security assessment list --output table
az security secure-score list --output table

# === Azure Policy ===
az policy definition list --query "[?contains(displayName, '<keyword>')]" -o table
az policy assignment create --name <name> --policy <policy-id> --scope <scope>
az policy state list -g <rg> --filter "complianceState eq 'NonCompliant'" -o table

# === Network Security ===
az network nsg create -g <rg> --name <nsg>
az network nsg rule create -g <rg> --nsg-name <nsg> --name <rule> --priority 100 ...
az network private-endpoint create --name <pe> -g <rg> --vnet-name <vnet> --subnet <subnet> ...

# === Identity ===
az webapp identity assign --name <app> -g <rg>
az role assignment create --role "<role>" --assignee <user> --scope <scope>
```

---

## Related Notes

- [[Cloud Security Fundamentals]] — General cloud security concepts and frameworks
- [[Azure IAM]] — Deep dive into Entra ID, RBAC, and Managed Identities
- [[Azure Networking]] — VNets, NSGs, Bastion, and network architecture
- [[Azure Monitoring]] — Monitor security events and set up alerts
- [[Azure Deployment]] — Secure your deployment pipeline
- [[AWS Security]] — Compare Azure security with AWS (IAM, KMS, GuardDuty, etc.)

---

> **Review:** Combine security with monitoring → [[Azure Monitoring]] and identity → [[Azure IAM]]
