# Azure IAM — Identity and Access Management

> **Control WHO can access your Azure resources and WHAT they can do.**
> Make sure you've read [[Azure Getting Started]] first.

---

## Table of Contents

- [[#How Identity Works in Azure]]
- [[#Microsoft Entra ID (formerly Azure AD)]]
- [[#Creating and Managing Users]]
- [[#Creating and Managing Groups]]
- [[#Azure RBAC — Role-Based Access Control]]
- [[#Built-in Roles]]
- [[#Assigning Roles — Hands-On]]
- [[#Service Principals]]
- [[#Managed Identities]]
- [[#Multi-Factor Authentication (MFA)]]
- [[#Conditional Access]]
- [[#Azure IAM vs AWS IAM]]
- [[#Best Practices]]

---

## How Identity Works in Azure

In Azure, identity and access management is handled by **two separate but connected systems**:

```
┌──────────────────────────┐     ┌──────────────────────────┐
│   Microsoft Entra ID     │     │       Azure RBAC         │
│   (Identity Provider)    │     │    (Authorization)       │
│                          │     │                          │
│  "WHO are you?"          │────▶│  "WHAT can you do?"      │
│                          │     │                          │
│  • Users                 │     │  • Roles                 │
│  • Groups                │     │  • Role Assignments      │
│  • Applications          │     │  • Scopes                │
│  • Service Principals    │     │  • Permissions           │
└──────────────────────────┘     └──────────────────────────┘
```

1. **Microsoft Entra ID** (formerly Azure Active Directory / Azure AD) — The **identity provider**.
   It answers the question: *"Who are you?"* It handles authentication — verifying that you are
   who you claim to be.

2. **Azure RBAC (Role-Based Access Control)** — The **authorization system**. It answers the
   question: *"What are you allowed to do?"* It controls what actions users can perform on
   which resources.

> 💡 **Key Insight**: In AWS, a single service (AWS IAM) handles both identity and authorization.
> In Azure, these are two separate systems that work together. This is one of the biggest
> conceptual differences between Azure and AWS.

---

## Microsoft Entra ID (formerly Azure AD)

### What is it?

Microsoft Entra ID is Microsoft's **cloud-based identity and access management service**. It's
the backbone of identity in Azure and Microsoft 365.

> ⚠️ **Name Change Alert**: Microsoft renamed "Azure Active Directory" (Azure AD) to
> "Microsoft Entra ID" in 2023. You'll see both names everywhere. They are the same thing.
> Old documentation and CLI commands may still reference "Azure AD."

### Key Concepts

#### Tenant

- An **instance** of Microsoft Entra ID.
- Every organization gets one tenant when they sign up for Azure.
- It's your organization's dedicated identity directory.
- Has a unique tenant ID (a GUID) and a primary domain (e.g., `yourcompany.onmicrosoft.com`).
- Think of it as your organization's identity database.

#### Users

- Represent **people** in your organization.
- Each user has a unique username (User Principal Name / UPN) like `john@yourcompany.onmicrosoft.com`.
- Two types:
  - **Member users** — Belong to your organization (employees).
  - **Guest users** — External users invited to collaborate (contractors, partners).

#### Groups

- **Collections of users** that make managing access easier.
- Two types:
  - **Security groups** — Used for assigning permissions to resources.
  - **Microsoft 365 groups** — Also give access to shared mailbox, SharePoint, etc.
- **Dynamic groups** — Membership is automatic based on user attributes (e.g., all users in
  the "Engineering" department are automatically added).

#### Applications (App Registrations)

- Register **applications** that need to interact with Azure resources or Entra ID.
- Each registered app gets an Application (client) ID.
- Used for web apps, APIs, mobile apps, daemon services, etc.

#### Service Principals

- The **identity that an application uses** to access Azure resources.
- When you register an app, a service principal is automatically created.
- Think of it as a "robot user" for your application.
- Has credentials (password or certificate) that the app uses to authenticate.

### Viewing Entra ID in the Portal

1. Go to the **Azure Portal** → search for **"Microsoft Entra ID"** (or "Azure Active Directory").
2. You'll see your tenant overview with:
   - Tenant name and ID
   - Primary domain
   - Number of users, groups, and applications
3. Left sidebar has sections for Users, Groups, App registrations, Enterprise applications, etc.

---

## Creating and Managing Users

### Via the Azure Portal

1. Go to **Microsoft Entra ID** → **Users** → **+ New user** → **Create new user**.
2. Fill in:
   - **User principal name**: `john@yourcompany.onmicrosoft.com`
   - **Display name**: `John Developer`
   - **Password**: Auto-generated or set manually
   - Check **"Require password change on first login"**
3. Optionally fill in: First name, Last name, Job title, Department.
4. Click **Create**.

### Via Azure CLI

```bash
# Create a new user
az ad user create \
  --display-name "John Developer" \
  --user-principal-name john@yourcompany.onmicrosoft.com \
  --password "TempP@ss123!" \
  --force-change-password-next-sign-in true
```

The `--force-change-password-next-sign-in true` flag means John will have to set his own password
when he first logs in. **Always use this for security!**

```bash
# List all users in your tenant
az ad user list --output table
```

Output:
```
DisplayName       UserPrincipalName                        ObjectId
----------------  ---------------------------------------  ------------------------------------
John Developer    john@yourcompany.onmicrosoft.com         xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Jane Admin        jane@yourcompany.onmicrosoft.com         xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

```bash
# Get details of a specific user
az ad user show --id john@yourcompany.onmicrosoft.com
```

This returns a JSON object with all user properties including object ID, display name, mail, etc.

```bash
# Update a user's display name
az ad user update --id john@yourcompany.onmicrosoft.com --display-name "John Senior Developer"
```

```bash
# Delete a user
az ad user delete --id john@yourcompany.onmicrosoft.com
```

> ⚠️ **Warning**: Deleted users go to a "Deleted users" recycle bin and can be restored within
> 30 days. After that, they're permanently deleted.

```bash
# List deleted users
az ad user list --filter "accountEnabled eq false" --output table
```

---

## Creating and Managing Groups

Groups make managing permissions **much** easier. Instead of assigning roles to each user
individually, assign roles to a group — then just add/remove users from the group.

### Via Azure CLI

```bash
# Create a security group
az ad group create \
  --display-name "Developers" \
  --mail-nickname "developers"
```

The `--mail-nickname` is required but is just an identifier (it doesn't create an email address
for security groups).

```bash
# List all groups
az ad group list --output table
```

```bash
# Get the object ID of a user (you need this to add them to a group)
az ad user show --id john@yourcompany.onmicrosoft.com --query id --output tsv
```

```bash
# Add a user to a group
az ad group member add \
  --group "Developers" \
  --member-id <user-object-id>
```

Replace `<user-object-id>` with the actual GUID from the previous command.

```bash
# List members of a group
az ad group member list --group "Developers" --output table
```

```bash
# Check if a user is in a group
az ad group member check --group "Developers" --member-id <user-object-id>
```

```bash
# Remove a user from a group
az ad group member remove --group "Developers" --member-id <user-object-id>
```

```bash
# Delete a group
az ad group delete --group "Developers"
```

### Group Best Practices

- Create groups that match your organizational structure:
  - `Developers` — Access to development resource groups
  - `DevOps-Engineers` — Access to deployment pipelines and infrastructure
  - `Readers-Prod` — Read-only access to production resources
  - `DB-Admins` — Access to database resources
- **Never assign permissions to individual users** — always use groups.
- Use descriptive names that indicate the level of access.

---

## Azure RBAC — Role-Based Access Control

### What is RBAC?

RBAC is Azure's authorization system. It determines what actions a user (or group, or service
principal) can perform on a specific set of resources.

### The RBAC Formula

A **Role Assignment** is the combination of three things:

```
Role Assignment = Security Principal + Role Definition + Scope
                  (WHO)               (WHAT)            (WHERE)
```

Let's break each one down:

#### Security Principal (WHO)

The entity that needs access. Can be:
- A **User** — A person in Entra ID
- A **Group** — A collection of users
- A **Service Principal** — An application identity
- A **Managed Identity** — An Azure-managed identity for services

#### Role Definition (WHAT)

A collection of permissions. Defines what actions are allowed or denied.
- Written as: `Microsoft.Compute/virtualMachines/read` (read VMs)
- Includes **Actions** (allowed) and **NotActions** (explicitly denied)
- Azure provides 100+ built-in roles, or you can create custom roles.

#### Scope (WHERE)

The boundary where the role applies. Azure scopes are hierarchical:

```
Management Group        ← Broadest scope
    └── Subscription
        └── Resource Group
            └── Resource    ← Narrowest scope
```

**Key Rule**: Roles assigned at a **higher scope are inherited** by all lower scopes.

Example: If you give someone "Reader" at the subscription level, they can read every resource
in every resource group in that subscription.

---

## Built-in Roles

Azure has 100+ built-in roles, but you only need to know a few to start:

### The Big Four

| Role | Permissions | Use Case |
|------|------------|----------|
| **Owner** | Full access to all resources + can assign roles to others | Subscription admins, team leads |
| **Contributor** | Full access to all resources, CANNOT assign roles | Developers who need to create/modify resources |
| **Reader** | View-only access to all resources | Auditors, new team members, stakeholders |
| **User Access Administrator** | Can only manage role assignments (who gets what access) | Security team managing permissions |

### Important Differences

```
Owner        = Can do EVERYTHING (including assigning roles to others)
Contributor  = Can do everything EXCEPT assign roles
Reader       = Can only VIEW, cannot change anything
```

> 💡 **Beginner Tip**: Start by giving everyone **Reader** access. Only grant Contributor or
> Owner when they specifically need it. This is the **principle of least privilege**.

### Other Useful Built-in Roles

| Role | What It Does |
|------|-------------|
| **Virtual Machine Contributor** | Manage VMs but not the network or storage |
| **Storage Blob Data Contributor** | Read/write/delete blob data in storage accounts |
| **SQL DB Contributor** | Manage SQL databases but not access their data |
| **Network Contributor** | Manage networking resources |
| **Key Vault Secrets User** | Read secrets from Key Vault |
| **AKS Cluster Admin** | Full access to AKS clusters |

### View Available Roles via CLI

```bash
# List all built-in roles
az role definition list --output table

# Search for specific roles
az role definition list --query "[?contains(roleName, 'Virtual Machine')]" --output table

# Get details of a specific role
az role definition list --name "Contributor" --output json
```

---

## Assigning Roles — Hands-On

### Via the Azure Portal

1. Go to the **resource** (or resource group, or subscription) you want to grant access to.
2. Click **"Access control (IAM)"** in the left menu.
3. Click **"+ Add"** → **"Add role assignment"**.
4. **Role tab**: Search for and select the role (e.g., "Contributor").
5. **Members tab**: Select "User, group, or service principal" → search for the user/group.
6. **Review + assign**: Review and confirm.

### Via Azure CLI

```bash
# Assign "Contributor" role to a user on a resource group
az role assignment create \
  --assignee john@yourcompany.onmicrosoft.com \
  --role "Contributor" \
  --resource-group myResourceGroup
```

Let's break down the parameters:
- `--assignee` — WHO gets the role (email, object ID, or service principal)
- `--role` — WHAT role to assign (name or ID)
- `--resource-group` — WHERE (the scope). You can also use `--scope` for other levels.

```bash
# Assign "Reader" at the subscription level
az role assignment create \
  --assignee john@yourcompany.onmicrosoft.com \
  --role "Reader" \
  --scope /subscriptions/<subscription-id>
```

```bash
# Assign a role to a GROUP (recommended over individual users)
az role assignment create \
  --assignee-object-id <group-object-id> \
  --assignee-principal-type Group \
  --role "Contributor" \
  --resource-group myResourceGroup
```

### Viewing Role Assignments

```bash
# List all role assignments on a resource group
az role assignment list --resource-group myResourceGroup --output table
```

Output:
```
Principal                              RoleDefinitionName    Scope
-------------------------------------  --------------------  --------------------------------------------------
john@yourcompany.onmicrosoft.com       Contributor           /subscriptions/.../resourceGroups/myResourceGroup
Developers                             Reader                /subscriptions/.../resourceGroups/myResourceGroup
```

```bash
# List role assignments for a specific user
az role assignment list --assignee john@yourcompany.onmicrosoft.com --output table

# List role assignments at the subscription level
az role assignment list --scope /subscriptions/<subscription-id> --output table
```

### Removing Role Assignments

```bash
# Remove a role assignment
az role assignment delete \
  --assignee john@yourcompany.onmicrosoft.com \
  --role "Contributor" \
  --resource-group myResourceGroup
```

---

## Service Principals

### What is a Service Principal?

A **Service Principal** is an identity for **applications, scripts, and automation tools** to
access Azure resources. Think of it as a "robot user account" for your code.

When do you need one?
- CI/CD pipelines (GitHub Actions, Azure DevOps) deploying to Azure
- Scripts that run unattended (no human to log in)
- Applications that need to access Azure resources

### Create a Service Principal

```bash
# Create a service principal with Contributor access to a resource group
az ad sp create-for-rbac \
  --name "my-cicd-pipeline" \
  --role Contributor \
  --scopes /subscriptions/<subscription-id>/resourceGroups/myResourceGroup
```

**Important Output** — Save this securely! You won't see the password again:

```json
{
  "appId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "displayName": "my-cicd-pipeline",
  "password": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "tenant": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

- `appId` — The username for the service principal (also called client ID)
- `password` — The password (also called client secret)
- `tenant` — Your Entra ID tenant ID

### Login as a Service Principal

```bash
# Login using service principal credentials
az login \
  --service-principal \
  --username <appId> \
  --password <password> \
  --tenant <tenant>
```

This is how your CI/CD pipeline or script would authenticate with Azure.

### Manage Service Principals

```bash
# List all service principals
az ad sp list --all --output table

# Get details of a specific service principal
az ad sp show --id <appId>

# Reset credentials (generate a new password)
az ad sp credential reset --id <appId>

# Delete a service principal
az ad sp delete --id <appId>
```

> ⚠️ **Security Warning**: Service principal credentials (password/secret) are like passwords.
> Store them securely (use Azure Key Vault, not plain text files!). Rotate them regularly.

---

## Managed Identities

### What are Managed Identities?

**Managed Identities** solve the biggest problem with service principals: **credential management**.

With a service principal, you have to:
- Create credentials
- Store them securely
- Rotate them regularly
- Worry about them leaking

With a **Managed Identity**, Azure handles all of this for you. No passwords. No secrets. Azure
automatically manages the identity's lifecycle and credentials.

### Two Types

#### System-Assigned Managed Identity

- **Tied to a specific Azure resource** (e.g., a VM or App Service).
- Created when you enable it on the resource.
- **Deleted automatically** when the resource is deleted.
- Cannot be shared — one identity per resource.

```bash
# Enable system-assigned managed identity on a VM
az vm identity assign \
  --resource-group myResourceGroup \
  --name myVM
```

#### User-Assigned Managed Identity

- **Standalone Azure resource** — exists independently.
- Can be **shared across multiple resources**.
- You manage its lifecycle (create and delete it yourself).

```bash
# Create a user-assigned managed identity
az identity create \
  --resource-group myResourceGroup \
  --name myManagedIdentity

# Assign it to a VM
az vm identity assign \
  --resource-group myResourceGroup \
  --name myVM \
  --identities myManagedIdentity
```

### Grant Permissions to a Managed Identity

After creating a managed identity, you need to give it permissions (just like any other principal):

```bash
# Get the principal ID of the managed identity
az vm show --resource-group myResourceGroup --name myVM --query identity.principalId --output tsv

# Assign a role to the managed identity
az role assignment create \
  --assignee <principal-id> \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/<sub-id>/resourceGroups/myResourceGroup/providers/Microsoft.Storage/storageAccounts/mystorageaccount
```

### When to Use What?

| Scenario | Use This |
|----------|----------|
| Azure service accessing another Azure service | ✅ Managed Identity |
| CI/CD pipeline (GitHub Actions, Jenkins) | Service Principal |
| On-premises application accessing Azure | Service Principal |
| Azure VM accessing Azure Storage | ✅ Managed Identity |
| Azure Functions accessing Azure SQL | ✅ Managed Identity |

> 💡 **Rule of Thumb**: If your code runs **on Azure**, use a Managed Identity. If it runs
> **outside Azure**, use a Service Principal.

---

## Multi-Factor Authentication (MFA)

### What is MFA?

MFA requires users to provide **two or more verification methods** when signing in:
1. **Something you know** — Password
2. **Something you have** — Phone, authenticator app, security key
3. **Something you are** — Fingerprint, face recognition

### Enabling MFA

#### Per-User MFA (Basic)

1. Go to **Microsoft Entra ID** → **Users** → **Per-user MFA**.
2. Select users → **Enable**.
3. Users will be prompted to set up MFA on next login.

#### Security Defaults (Recommended for Small Orgs)

1. Go to **Microsoft Entra ID** → **Properties** → **Manage security defaults**.
2. Toggle to **Enabled**.
3. This enables MFA for all users, blocks legacy authentication, and more.

### MFA via CLI

```bash
# Check if security defaults are enabled (requires Microsoft Graph permissions)
az rest --method GET --url "https://graph.microsoft.com/v1.0/policies/identitySecurityDefaultsEnforcementPolicy"
```

> 💡 **Always enable MFA.** It blocks 99.9% of account compromise attacks according to Microsoft.

---

## Conditional Access

### What is Conditional Access?

Conditional Access policies are **if-then rules** for access:

- **IF** a user is logging in from an untrusted location **THEN** require MFA.
- **IF** a user is on a non-compliant device **THEN** block access.
- **IF** a user is accessing sensitive data **THEN** require a managed device.

### Common Conditional Access Policies

| Policy | Condition | Action |
|--------|-----------|--------|
| Require MFA for admins | User is in admin role | Require MFA |
| Block legacy auth | App uses legacy authentication | Block access |
| Require managed device | Accessing sensitive apps | Require compliant device |
| Location-based | Login from outside country | Require MFA or block |

> ⚠️ **Note**: Conditional Access requires **Microsoft Entra ID P1 or P2 license** (not included
> in the free tier). But it's important to know about for real-world deployments.

---

## Azure IAM vs AWS IAM

Understanding the differences helps if you're learning both clouds:

| Aspect | Azure | AWS |
|--------|-------|-----|
| **Identity System** | Microsoft Entra ID (separate service) | AWS IAM (unified service) |
| **Authorization System** | Azure RBAC (separate from identity) | AWS IAM Policies (part of IAM) |
| **Users** | Entra ID Users | IAM Users |
| **Groups** | Entra ID Groups | IAM Groups |
| **Roles for services** | Managed Identities | IAM Roles |
| **App identity** | Service Principals | IAM Users with access keys |
| **Permission model** | Role assigned at a scope | Policy attached to principal |
| **Inheritance** | Roles inherit down the scope hierarchy | Policies don't inherit (explicit) |
| **Permission boundary** | Scope (Management Group → Subscription → RG → Resource) | Account → OU (via Organizations) |
| **MFA** | Entra ID MFA + Conditional Access | IAM MFA (simpler) |
| **SSO** | Built into Entra ID | AWS SSO (separate service) |
| **Equivalent** | Managed Identity ≈ | IAM Role for EC2/Lambda |

### Key Conceptual Differences

1. **Separation of concerns**: Azure separates identity (Entra ID) from authorization (RBAC).
   AWS combines them in IAM. Azure's approach is more modular but has a steeper learning curve.

2. **Scope-based vs Policy-based**: In Azure, you assign a role AT a scope. In AWS, you attach
   a policy TO a user/group/role. Azure's model is more intuitive for hierarchical organizations.

3. **Managed Identities vs IAM Roles**: Both solve the same problem (giving cloud services an
   identity without managing credentials), but they work differently. See [[AWS IAM]] for details.

For a full comparison, see [[AWS vs Azure Comparison]].

---

## Best Practices

### 1. Use Groups, Not Individual Users

❌ **Bad**: Assign "Contributor" role to john@company.com, jane@company.com, bob@company.com...

✅ **Good**: Create a "Developers" group → Add John, Jane, Bob to the group → Assign "Contributor"
role to the "Developers" group.

### 2. Apply the Principle of Least Privilege

- Start with **Reader** access. Only grant more when specifically needed.
- Use specific roles (e.g., "Virtual Machine Contributor") over broad roles (e.g., "Contributor").
- Assign at the **narrowest scope** possible (resource > resource group > subscription).

### 3. Use Managed Identities for Azure Services

- If your code runs on Azure → use Managed Identities.
- Avoid storing credentials in code, environment variables, or config files.

### 4. Enable MFA for Everyone

- Enable Security Defaults at minimum.
- Use Conditional Access for more granular control.
- Especially important for admin accounts.

### 5. Review Access Regularly

```bash
# Audit: List all role assignments in your subscription
az role assignment list --all --output table

# Check what a specific user can do
az role assignment list --assignee john@yourcompany.onmicrosoft.com --all --output table
```

### 6. Use Azure AD Privileged Identity Management (PIM)

- For sensitive roles (Owner, Contributor on production), use **just-in-time access**.
- Users request temporary elevation → approved → access expires automatically.
- Requires Entra ID P2 license.

### 7. Protect Service Principal Credentials

- Store secrets in **Azure Key Vault**, not in code or config files.
- Rotate credentials regularly.
- Use certificate-based authentication when possible (more secure than passwords).

---

## Quick Reference Card

```bash
# ===== USER MANAGEMENT =====
az ad user create --display-name "Name" --user-principal-name user@domain.com --password "Pass!"
az ad user list --output table
az ad user show --id user@domain.com
az ad user delete --id user@domain.com

# ===== GROUP MANAGEMENT =====
az ad group create --display-name "GroupName" --mail-nickname "groupname"
az ad group member add --group "GroupName" --member-id <user-object-id>
az ad group member list --group "GroupName" --output table
az ad group member remove --group "GroupName" --member-id <user-object-id>

# ===== ROLE ASSIGNMENTS =====
az role assignment create --assignee user@domain.com --role "Contributor" --resource-group myRG
az role assignment list --resource-group myRG --output table
az role assignment delete --assignee user@domain.com --role "Contributor" --resource-group myRG

# ===== SERVICE PRINCIPALS =====
az ad sp create-for-rbac --name "sp-name" --role Contributor --scopes /subscriptions/<id>/resourceGroups/<rg>
az login --service-principal -u <appId> -p <password> --tenant <tenant>

# ===== MANAGED IDENTITIES =====
az vm identity assign --resource-group myRG --name myVM
az identity create --resource-group myRG --name myIdentity

# ===== ROLE DEFINITIONS =====
az role definition list --output table
az role definition list --name "Contributor" --output json
```

---

**Tags**: #azure #iam #security #identity #rbac #entra-id #beginner
**Related**: [[Cloud Security Fundamentals]] | [[Azure Getting Started]] | [[Azure Security]] | [[AWS IAM]]
