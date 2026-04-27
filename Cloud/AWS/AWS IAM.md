# AWS IAM (Identity and Access Management)

> **IAM** controls **WHO** can do **WHAT** on **WHICH** AWS resources. It's the security foundation of every AWS account — and it's completely **free**.

---

## What is IAM?

**Identity and Access Management (IAM)** is the service that manages authentication (who are you?) and authorisation (what can you do?) in AWS.

### Key Facts

| Detail              | Info                                                        |
| ------------------- | ----------------------------------------------------------- |
| **Full Name**       | AWS Identity and Access Management                          |
| **Cost**            | Free — no additional charge for IAM                         |
| **Scope**           | **Global** — not tied to any specific region                |
| **Purpose**         | Control access to AWS services and resources                |
| **Core Components** | Users, Groups, Policies, Roles                              |

### Why IAM Matters

- Without IAM, anyone with your root credentials has **unlimited access** to everything
- IAM lets you implement the **principle of least privilege** — give only the permissions needed
- IAM is your first and most important line of defence in AWS security
- Every AWS certification exam tests IAM heavily

### IAM Components Overview

```
IAM
├── Users       → Individual people or applications
├── Groups      → Collections of users (e.g., "Developers", "Admins")
├── Policies    → JSON documents defining permissions
├── Roles       → Temporary credentials for services/applications
└── MFA         → Multi-Factor Authentication for extra security
```

---

## The Root Account

### What is the Root Account?

When you create an AWS account, you create a **root account** — the account tied to your email address. This account has:

- ✅ **Complete, unrestricted access** to every AWS service and resource
- ✅ Can close the account
- ✅ Can change the support plan
- ✅ Can change account settings

### ⚠️ NEVER Use the Root Account for Daily Tasks

The root account is like having the master key to your entire building. You don't carry it around daily — you give people keys to specific rooms.

**What to do instead:**

1. **Enable MFA on root immediately** (see [[AWS Getting Started]])
2. **Create an IAM admin user** for daily administration
3. **Lock away root credentials** — only use for tasks that specifically require root
4. **Never create access keys** for the root account

### Tasks That REQUIRE Root Account

Very few things actually need root access:

- Changing the account name, email, or password
- Changing the AWS support plan
- Closing the AWS account
- Enabling MFA on the root account itself
- Restoring IAM user permissions (when accidentally locked out)
- Configuring certain S3 bucket policies involving the root account
- Registering as a seller in the Reserved Instance Marketplace

For everything else, use an IAM user or role.

---

## IAM Users

An **IAM user** represents a person or application that interacts with AWS. Each user has unique credentials.

### Two Types of Access

| Access Type        | What It Provides                         | Use Case                    |
| ------------------ | ---------------------------------------- | --------------------------- |
| **Console Access** | Username + Password (for web login)      | Human users browsing console|
| **Programmatic**   | Access Key ID + Secret Access Key        | CLI, SDKs, API calls        |

> 💡 A user can have both types of access simultaneously.

### Creating a User via Console (Step-by-Step)

1. Go to the **IAM Console** → **Users** → **Create user**
2. Enter a **username** (e.g., `developer1`)
3. Check **"Provide user access to the AWS Management Console"** if they need console access
4. Choose password options:
   - Auto-generated password OR custom password
   - ✅ Check "User must create a new password at next sign-in"
5. Click **Next**
6. **Set permissions** — choose one of:
   - Add user to group (✅ recommended)
   - Copy permissions from existing user
   - Attach policies directly (not recommended)
7. Click **Next** → Review → **Create user**
8. **Download or save the credentials** — you won't see the password again!

### Creating a User via CLI

```bash
# Step 1: Create the user
aws iam create-user --user-name developer1

# Step 2: Give console access (password login)
aws iam create-login-profile \
  --user-name developer1 \
  --password 'TempP@ss123!' \
  --password-reset-required

# Step 3: Give programmatic access (CLI/SDK)
aws iam create-access-key --user-name developer1
# IMPORTANT: Save the AccessKeyId and SecretAccessKey from the output!
# You will NOT be able to see the SecretAccessKey again.

# Step 4: Verify the user exists
aws iam get-user --user-name developer1

# Step 5: List all users
aws iam list-users --output table
```

### Managing Users

```bash
# Delete a user's access key
aws iam delete-access-key --user-name developer1 --access-key-id AKIAIOSFODNN7EXAMPLE

# Delete a user's login profile (console access)
aws iam delete-login-profile --user-name developer1

# Delete the user (must remove from groups and delete keys first)
aws iam remove-user-from-group --group-name Developers --user-name developer1
aws iam delete-user --user-name developer1

# List access keys for a user
aws iam list-access-keys --user-name developer1
```

### Console Sign-In URL

IAM users don't sign in at the normal AWS sign-in page. They use a special URL:

```
https://<account-id-or-alias>.signin.aws.amazon.com/console
```

You can set a **custom alias**:

```bash
aws iam create-account-alias --account-alias my-company-aws
# Now the sign-in URL is: https://my-company-aws.signin.aws.amazon.com/console
```

---

## IAM Groups

A **group** is a collection of IAM users. You attach permissions (policies) to the group, and all users in the group inherit those permissions.

### Why Use Groups?

- ❌ **Don't** attach policies to individual users — this doesn't scale
- ✅ **Do** create groups for job functions and attach policies to groups
- When a new developer joins, just add them to the "Developers" group
- When someone changes roles, move them to a different group

### Common Group Structure

```
Groups
├── Admins        → AdministratorAccess policy
├── Developers    → PowerUserAccess + CodeCommit access
├── ReadOnly      → ReadOnlyAccess policy
├── DBAdmins      → RDS/DynamoDB full access
└── Billing       → Billing access only
```

### Creating and Managing Groups via CLI

```bash
# Create a group
aws iam create-group --group-name Developers

# Add a user to the group
aws iam add-user-to-group --group-name Developers --user-name developer1

# List users in a group
aws iam get-group --group-name Developers

# List all groups
aws iam list-groups --output table

# List groups a user belongs to
aws iam list-groups-for-user --user-name developer1

# Remove a user from a group
aws iam remove-user-from-group --group-name Developers --user-name developer1

# Delete a group (must remove all users and detach policies first)
aws iam delete-group --group-name Developers
```

### Important Notes on Groups

- A user can belong to **multiple groups** (up to 10)
- Groups **cannot be nested** — you can't put a group inside another group
- Groups are for **users only** — you cannot add roles or other groups
- There is **no default group** — new users don't automatically belong to any group

---

## IAM Policies

**Policies** are JSON documents that define what actions are allowed or denied on which resources.

### Policy Structure

Every IAM policy has this structure:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DescriptiveName",
            "Effect": "Allow",
            "Action": ["service:ActionName"],
            "Resource": "arn:aws:service:region:account:resource",
            "Condition": {
                "ConditionOperator": {
                    "ConditionKey": "ConditionValue"
                }
            }
        }
    ]
}
```

### Policy Elements Explained

| Element       | Required? | Description                                        |
| ------------- | --------- | -------------------------------------------------- |
| **Version**   | Yes       | Always `"2012-10-17"` (the current policy language)|
| **Statement** | Yes       | Array of permission statements                     |
| **Sid**       | No        | Statement ID — a friendly name                     |
| **Effect**    | Yes       | `"Allow"` or `"Deny"`                              |
| **Action**    | Yes       | API actions (e.g., `"s3:GetObject"`)               |
| **Resource**  | Yes       | ARN of the resource(s) this applies to             |
| **Condition** | No        | When this statement applies                        |

### Example: S3 Read/Write Policy (Line-by-Line)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowS3ReadWrite",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::my-bucket",
                "arn:aws:s3:::my-bucket/*"
            ]
        }
    ]
}
```

**Breakdown:**
- `"Effect": "Allow"` → This grants permission (vs blocking it)
- `"s3:GetObject"` → Can download files from the bucket
- `"s3:PutObject"` → Can upload files to the bucket
- `"s3:ListBucket"` → Can list the contents of the bucket
- `"arn:aws:s3:::my-bucket"` → The bucket itself (for ListBucket)
- `"arn:aws:s3:::my-bucket/*"` → All objects inside the bucket (for Get/Put)

### Three Types of Policies

#### 1. AWS Managed Policies

Pre-built by AWS. Ready to use. Automatically updated by AWS.

```bash
# Common AWS Managed Policies
aws iam list-policies --scope AWS --query 'Policies[?PolicyName==`AdministratorAccess`]'
```

| Policy Name              | What It Does                                           |
| ------------------------ | ------------------------------------------------------ |
| `AdministratorAccess`    | Full access to ALL services (use sparingly!)           |
| `ReadOnlyAccess`         | Read-only access to ALL services                       |
| `PowerUserAccess`        | Full access EXCEPT IAM and Organizations               |
| `AmazonS3FullAccess`     | Full access to S3                                      |
| `AmazonS3ReadOnlyAccess` | Read-only access to S3                                 |
| `AmazonEC2FullAccess`    | Full access to EC2                                     |
| `AmazonDynamoDBFullAccess`| Full access to DynamoDB                               |

#### 2. Customer Managed Policies

Policies **you create** for your specific needs. You control the content and can update them.

```bash
# Create a custom policy from a JSON file
aws iam create-policy \
  --policy-name S3-ReadWrite-MyBucket \
  --policy-document file://my-s3-policy.json

# List your custom policies
aws iam list-policies --scope Local --output table
```

#### 3. Inline Policies

Policies embedded directly in a user, group, or role. **Not recommended** — harder to manage and audit.

```bash
# Attach an inline policy to a user (avoid this!)
aws iam put-user-policy \
  --user-name developer1 \
  --policy-name InlineS3Access \
  --policy-document file://my-policy.json
```

### Attaching Policies

```bash
# Attach a managed policy to a GROUP (recommended)
aws iam attach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/PowerUserAccess

# Attach a managed policy to a USER (less ideal)
aws iam attach-user-policy \
  --user-name developer1 \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Attach a managed policy to a ROLE
aws iam attach-role-policy \
  --role-name EC2-S3-Access \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# List policies attached to a group
aws iam list-attached-group-policies --group-name Developers

# Detach a policy from a group
aws iam detach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/PowerUserAccess
```

### Policy Evaluation Logic

When AWS evaluates whether to allow or deny a request:

```
1. By default, everything is DENIED (implicit deny)
2. Check all applicable policies
3. If ANY policy has an explicit DENY → DENIED (deny always wins)
4. If ANY policy has an ALLOW → ALLOWED
5. If no policy matches → DENIED (implicit deny)
```

**Key Rule: Explicit Deny ALWAYS wins over Allow.**

---

## IAM Roles

A **role** is an IAM identity with temporary credentials. Unlike users, roles don't have permanent passwords or access keys. Instead, roles are **assumed** and provide **temporary security credentials**.

### When to Use Roles

| Scenario                                    | Use a Role? |
| ------------------------------------------- | ----------- |
| EC2 instance needs to access S3             | ✅ Yes      |
| Lambda function needs to access DynamoDB    | ✅ Yes      |
| Another AWS account needs access            | ✅ Yes      |
| An application running on-premise needs AWS | ✅ Yes      |
| A human developer needs daily access        | ❌ Use user + group |

### How Roles Work

```
1. Create a role with TWO policies:
   ├── Trust Policy     → WHO can assume this role
   └── Permission Policy → WHAT they can do once assumed

2. An entity (EC2, Lambda, user) "assumes" the role
3. AWS provides temporary credentials (valid 1-12 hours)
4. The entity uses those credentials to make API calls
```

### Creating a Role for EC2 (Step-by-Step)

This example creates a role that allows EC2 instances to read from S3.

#### Step 1: Create the Trust Policy

Create a file called `trust-policy.json`:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

This says: "The EC2 service is allowed to assume this role."

#### Step 2: Create the Role

```bash
aws iam create-role \
  --role-name EC2-S3-Access \
  --assume-role-policy-document file://trust-policy.json \
  --description "Allows EC2 instances to read from S3"
```

#### Step 3: Attach Permission Policy

```bash
aws iam attach-role-policy \
  --role-name EC2-S3-Access \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

#### Step 4: Create an Instance Profile

EC2 instances use **instance profiles** to assume roles. An instance profile is a container for the role.

```bash
aws iam create-instance-profile --instance-profile-name EC2-S3-Access-Profile
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-S3-Access-Profile \
  --role-name EC2-S3-Access
```

#### Step 5: Launch EC2 with the Role

```bash
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t2.micro \
  --iam-instance-profile Name=EC2-S3-Access-Profile \
  --key-name my-key
```

Now this EC2 instance can read from S3 without any access keys!

### Common Role Trust Policies

```json
// For Lambda
{ "Principal": { "Service": "lambda.amazonaws.com" } }

// For ECS Tasks
{ "Principal": { "Service": "ecs-tasks.amazonaws.com" } }

// For another AWS account (cross-account)
{ "Principal": { "AWS": "arn:aws:iam::987654321098:root" } }
```

### Managing Roles

```bash
# List all roles
aws iam list-roles --output table

# Get role details
aws iam get-role --role-name EC2-S3-Access

# List policies attached to a role
aws iam list-attached-role-policies --role-name EC2-S3-Access

# Assume a role (get temporary credentials)
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/EC2-S3-Access \
  --role-session-name my-session

# Delete a role (must detach policies and remove from instance profiles first)
aws iam detach-role-policy --role-name EC2-S3-Access --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam remove-role-from-instance-profile --instance-profile-name EC2-S3-Access-Profile --role-name EC2-S3-Access
aws iam delete-instance-profile --instance-profile-name EC2-S3-Access-Profile
aws iam delete-role --role-name EC2-S3-Access
```

---

## MFA (Multi-Factor Authentication)

**MFA** adds an extra layer of security. Even if someone steals your password, they can't log in without the second factor.

### Types of MFA in AWS

| Type                     | Description                                      |
| ------------------------ | ------------------------------------------------ |
| **Virtual MFA device**   | App on your phone (Google Authenticator, Authy)  |
| **Hardware TOTP token**  | Physical device that generates codes             |
| **FIDO security key**    | USB security key (YubiKey, etc.)                 |

### Enabling MFA for an IAM User (Console)

1. Go to **IAM** → **Users** → Select the user
2. Click **"Security credentials"** tab
3. Under "Multi-factor authentication" → **"Assign MFA device"**
4. Choose **"Authenticator app"**
5. Open Google Authenticator → Scan the QR code
6. Enter two consecutive codes from the app
7. Click **"Add MFA"**

### Enabling MFA via CLI

```bash
# Step 1: Create a virtual MFA device
aws iam create-virtual-mfa-device \
  --virtual-mfa-device-name developer1-mfa \
  --outfile qr-code.png \
  --bootstrap-method QRCodePNG

# Step 2: Enable MFA for the user (need two consecutive codes from the app)
aws iam enable-mfa-device \
  --user-name developer1 \
  --serial-number arn:aws:iam::123456789012:mfa/developer1-mfa \
  --authentication-code1 123456 \
  --authentication-code2 789012
```

### MFA Best Practices

- ✅ Enable MFA on the **root account** — this is the #1 security recommendation
- ✅ Enable MFA on **all IAM users** with console access
- ✅ Use **hardware MFA** for root account if possible
- ✅ Store backup codes securely
- ❌ Don't share MFA devices between users

---

## Access Keys Best Practices

Access keys (Access Key ID + Secret Access Key) are used for programmatic access (CLI, SDKs, APIs).

### Security Rules

| Rule                                    | Why                                          |
| --------------------------------------- | -------------------------------------------- |
| **Never commit keys to Git**            | Bots scan GitHub for exposed keys            |
| **Rotate keys regularly** (90 days)     | Limits exposure if a key is compromised      |
| **Use IAM roles instead** when possible | Roles use temporary creds, keys are permanent|
| **Delete unused keys**                  | Fewer keys = smaller attack surface          |
| **Use environment variables or profiles**| Don't hardcode keys in application code     |

### Rotating Access Keys

```bash
# Step 1: Create a new access key
aws iam create-access-key --user-name developer1

# Step 2: Update your applications to use the new key
aws configure  # Enter new key details

# Step 3: Verify the new key works
aws sts get-caller-identity

# Step 4: Deactivate the old key
aws iam update-access-key --user-name developer1 --access-key-id OLDKEYID --status Inactive

# Step 5: After confirming everything works, delete the old key
aws iam delete-access-key --user-name developer1 --access-key-id OLDKEYID
```

### Checking for Unused Keys

```bash
# Generate a credential report
aws iam generate-credential-report

# Download the credential report (CSV format)
aws iam get-credential-report --query 'Content' --output text | base64 --decode > credential-report.csv
```

This report shows:
- When each user's keys were last used
- Whether MFA is enabled
- When passwords were last rotated

---

## IAM Access Analyzer

**IAM Access Analyzer** helps you identify resources shared with external entities and validates policies.

### Key Features

- **External access findings** — Detects resources (S3 buckets, IAM roles, KMS keys) shared outside your account
- **Policy validation** — Checks your policies for errors and suggests improvements
- **Policy generation** — Generates policies based on actual CloudTrail activity

### Setting Up Access Analyzer

```bash
# Create an analyzer
aws accessanalyzer create-analyzer \
  --analyzer-name my-account-analyzer \
  --type ACCOUNT

# List findings
aws accessanalyzer list-findings --analyzer-arn <analyzer-arn>
```

---

## IAM Best Practices Summary

### The IAM Security Checklist

```
✅ 1. Enable MFA on root account
✅ 2. Create individual IAM users (never share credentials)
✅ 3. Use groups for permissions (not individual user policies)
✅ 4. Use roles for EC2, Lambda, and cross-account access
✅ 5. Grant least privilege (start with zero permissions, add as needed)
✅ 6. Use AWS managed policies when possible
✅ 7. Rotate credentials regularly
✅ 8. Never hardcode access keys in code
✅ 9. Remove unnecessary users, roles, and credentials
✅ 10. Enable CloudTrail for auditing all API calls
✅ 11. Set up IAM Access Analyzer
✅ 12. Use strong password policy
```

### Setting a Password Policy

```bash
aws iam update-account-password-policy \
  --minimum-password-length 14 \
  --require-symbols \
  --require-numbers \
  --require-uppercase-characters \
  --require-lowercase-characters \
  --allow-users-to-change-password \
  --max-password-age 90 \
  --password-reuse-prevention 12
```

---

## Hands-On Lab: Complete IAM Setup

Follow these steps to set up a proper IAM structure for a small team:

### Step 1: Create Groups

```bash
aws iam create-group --group-name Admins
aws iam create-group --group-name Developers
aws iam create-group --group-name ReadOnly
```

### Step 2: Attach Policies to Groups

```bash
aws iam attach-group-policy --group-name Admins --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam attach-group-policy --group-name Developers --policy-arn arn:aws:iam::aws:policy/PowerUserAccess
aws iam attach-group-policy --group-name ReadOnly --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

### Step 3: Create Users

```bash
aws iam create-user --user-name admin1
aws iam create-user --user-name developer1
aws iam create-user --user-name developer2
aws iam create-user --user-name viewer1
```

### Step 4: Add Users to Groups

```bash
aws iam add-user-to-group --group-name Admins --user-name admin1
aws iam add-user-to-group --group-name Developers --user-name developer1
aws iam add-user-to-group --group-name Developers --user-name developer2
aws iam add-user-to-group --group-name ReadOnly --user-name viewer1
```

### Step 5: Create Login Profiles (Console Access)

```bash
aws iam create-login-profile --user-name admin1 --password 'Admin@Temp123!' --password-reset-required
aws iam create-login-profile --user-name developer1 --password 'Dev@Temp123!' --password-reset-required
aws iam create-login-profile --user-name developer2 --password 'Dev@Temp456!' --password-reset-required
aws iam create-login-profile --user-name viewer1 --password 'View@Temp123!' --password-reset-required
```

### Step 6: Verify Setup

```bash
# List all users
aws iam list-users --output table

# List all groups
aws iam list-groups --output table

# Check group membership
aws iam get-group --group-name Developers --output table

# Check policies attached to Developers group
aws iam list-attached-group-policies --group-name Developers --output table
```

---

## Related Notes

- [[Cloud Security Fundamentals]] — General cloud security principles
- [[AWS Getting Started]] — Setting up your AWS account
- [[AWS Security]] — Advanced security services (GuardDuty, WAF, Shield)
- [[AWS Compute]] — EC2 instance roles and Lambda execution roles
- [[AWS Storage]] — S3 bucket policies and access control
