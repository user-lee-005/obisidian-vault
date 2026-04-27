# Azure Storage

> **Store files, blobs, disks, queues, and file shares in the cloud.**
> Prerequisites: [[Azure Getting Started]] (you need a resource group first!)

---

## Table of Contents

- [[#Azure Storage Overview]]
- [[#Storage Accounts]]
- [[#Redundancy Options]]
- [[#Azure Blob Storage]]
- [[#Blob Access Tiers]]
- [[#Hands-On — Working with Blobs]]
- [[#Blob Lifecycle Management]]
- [[#Blob Security — SAS Tokens]]
- [[#Azure Managed Disks]]
- [[#Azure Files]]
- [[#Azure Queue Storage]]
- [[#Azure Table Storage]]
- [[#Storage Comparison — Azure vs AWS]]
- [[#Cost Optimization Tips]]

---

## Azure Storage Overview

Azure Storage is a collection of cloud storage services for different use cases:

```
Azure Storage Account (container for all storage services)
│
├── Blob Storage      — Object storage (files, images, backups, any binary data)
├── Azure Files       — Managed file shares (SMB/NFS, mountable like a network drive)
├── Queue Storage     — Simple message queuing for decoupling components
├── Table Storage     — NoSQL key-value store (simple structured data)
└── (Managed Disks)   — Block storage for VMs (technically separate, but related)
```

### Key Characteristics

- **Durable** — Data is replicated across multiple copies (up to 6 copies across regions).
- **Secure** — Encrypted at rest by default (256-bit AES).
- **Scalable** — Virtually unlimited storage capacity.
- **Accessible** — Available via REST API, CLI, SDKs, Azure Portal.
- **Cost-effective** — Pay only for what you store and access.

---

## Storage Accounts

### What is a Storage Account?

A **Storage Account** is the top-level container for all Azure storage services. Before you
can store any data, you need to create a storage account.

Think of it like creating an account at a bank — once you have the account, you can have
savings, checking, and investment accounts under it. Similarly, a storage account can hold
blob containers, file shares, queues, and tables.

### Storage Account Rules

- **Name must be globally unique** — It's used in the URL (e.g., `mystorageaccount.blob.core.windows.net`).
- **Name must be 3-24 characters** — Only lowercase letters and numbers. No hyphens, no uppercase.
- **One account can hold all storage types** — You don't need separate accounts for blobs vs files.
- **Every resource group can have multiple storage accounts**.

### Hands-On: Create a Storage Account

```bash
# Create a storage account
az storage account create \
  --name mystorageaccount123 \
  --resource-group myResourceGroup \
  --location centralindia \
  --sku Standard_LRS \
  --kind StorageV2
```

Let's break down the parameters:

| Parameter | Meaning |
|-----------|---------|
| `--name` | Globally unique name (lowercase + numbers only) |
| `--resource-group` | Which resource group to put it in |
| `--location` | Azure region for the storage account |
| `--sku` | Redundancy option (LRS, ZRS, GRS, RA-GRS, etc.) |
| `--kind` | Account type: `StorageV2` is the recommended modern option |

### Get the Connection String

The connection string is used by applications to access the storage account:

```bash
# Get the connection string (contains the account key — keep it secret!)
az storage account show-connection-string \
  --name mystorageaccount123 \
  --resource-group myResourceGroup \
  --output tsv
```

Output:
```
DefaultEndpointsProtocol=https;AccountName=mystorageaccount123;AccountKey=xxxx...;EndpointSuffix=core.windows.net
```

> ⚠️ **Security Warning**: The connection string contains the account key, which gives **full
> access** to the storage account. Never put it in source code! Use environment variables,
> Azure Key Vault, or Managed Identities instead. See [[Azure IAM]] for details.

### Get Account Keys

```bash
# List access keys
az storage account keys list \
  --account-name mystorageaccount123 \
  --resource-group myResourceGroup \
  --output table
```

Each storage account has **two keys** (key1 and key2). This allows you to rotate keys without
downtime — switch your apps to key2, regenerate key1, then switch back.

```bash
# Regenerate a key
az storage account keys renew \
  --account-name mystorageaccount123 \
  --resource-group myResourceGroup \
  --key key1
```

### Manage Storage Accounts

```bash
# List all storage accounts in your subscription
az storage account list --output table

# Show details of a specific storage account
az storage account show --name mystorageaccount123 --resource-group myResourceGroup

# Delete a storage account (deletes ALL data inside!)
az storage account delete --name mystorageaccount123 --resource-group myResourceGroup --yes
```

---

## Redundancy Options

Azure stores multiple copies of your data to protect against failures. You choose the
redundancy level when creating a storage account.

### Redundancy Types

```
                    Within One Region              Across Two Regions
                    ─────────────────              ──────────────────
Single DC:          LRS (3 copies)                 ───
Multiple AZs:       ZRS (3 copies, 3 AZs)         ───
Geo-Redundant:      ───                            GRS (6 copies: 3 + 3)
Geo + Zone:         ───                            GZRS (6 copies: 3 AZs + 3)
Geo + Read Access:  ───                            RA-GRS (6 copies, read from secondary)
Geo + Zone + Read:  ───                            RA-GZRS (6 copies, read from secondary)
```

### Detailed Breakdown

| SKU | Copies | Locations | Durability | Use Case |
|-----|--------|-----------|------------|----------|
| **LRS** | 3 | 1 data center | 99.999999999% (11 nines) | Dev/test, non-critical data |
| **ZRS** | 3 | 3 availability zones | 99.9999999999% (12 nines) | High availability within region |
| **GRS** | 6 | 2 regions (3 + 3) | 99.99999999999999% (16 nines) | Disaster recovery |
| **RA-GRS** | 6 | 2 regions, read from secondary | 16 nines | DR + read from secondary |
| **GZRS** | 6 | 3 AZs + 3 in secondary region | 16 nines | Maximum protection |
| **RA-GZRS** | 6 | 3 AZs + 3, read from both | 16 nines | Maximum protection + availability |

### Which One to Choose?

| Scenario | Recommended |
|----------|-------------|
| Learning / Dev / Test | **LRS** (cheapest) |
| Production (single region) | **ZRS** |
| Production (multi-region, DR) | **GRS** or **RA-GRS** |
| Mission-critical (maximum) | **RA-GZRS** |

> 💡 **For learning**: Always use **LRS** (Standard_LRS). It's the cheapest and perfectly fine
> for experimentation.

---

## Azure Blob Storage

### What is Blob Storage?

**Blob Storage** is Azure's **object storage** service — the equivalent of **AWS S3**. It stores
unstructured data: files, images, videos, backups, logs, documents — basically anything.

"Blob" stands for **Binary Large Object**.

### The Hierarchy

```
Storage Account
    └── Container          ← Like a folder / S3 bucket
        ├── Blob 1         ← A file (e.g., photo.jpg)
        ├── Blob 2         ← Another file (e.g., report.pdf)
        └── folder/        ← Virtual folder (just a name prefix)
            └── Blob 3     ← e.g., folder/data.csv
```

- **Storage Account** — The top-level Azure resource.
- **Container** — A grouping of blobs (like a folder or an S3 bucket). A storage account can
  have unlimited containers.
- **Blob** — The actual data (file). A container can have unlimited blobs.

### Blob Types

| Type | Description | Use Case |
|------|------------|----------|
| **Block Blob** | Default type. Optimized for upload/download. Up to 190.7 TB. | Images, documents, media, backups — most common |
| **Append Blob** | Optimized for append operations. Cannot modify existing data. | Log files, audit trails |
| **Page Blob** | Optimized for random read/write. Fixed 512-byte pages. | VM disks (VHDs) — used internally by Azure |

> 💡 **99% of the time, you'll use Block Blobs.** The other types are for specific use cases.

### Container Access Levels

| Level | Description |
|-------|------------|
| **Private** (default) | No anonymous access. Requires authentication. |
| **Blob** | Anonymous read access to individual blobs (but can't list blobs). |
| **Container** | Anonymous read + list access to all blobs in the container. |

> ⚠️ **Security**: Always use **Private** unless you have a specific reason to make data public.
> Use SAS tokens for temporary access instead.

---

## Blob Access Tiers

Azure Blob Storage offers **access tiers** to optimize cost based on how frequently you access
your data:

| Tier | Access Frequency | Storage Cost | Access Cost | Min Retention |
|------|-----------------|--------------|-------------|---------------|
| **Hot** | Frequently accessed | Highest | Lowest | None |
| **Cool** | Infrequently accessed (30+ days) | Lower | Higher | 30 days |
| **Cold** | Rarely accessed (90+ days) | Even lower | Even higher | 90 days |
| **Archive** | Almost never accessed (180+ days) | Lowest | Highest (+ rehydration time) | 180 days |

### Key Points

- **Hot** — Default tier. Use for active data, frequently accessed files.
- **Cool** — ~50% cheaper storage than Hot. Use for backups, older data.
- **Cold** — ~70% cheaper storage than Hot. Use for compliance archives, long-term backups.
- **Archive** — ~90% cheaper storage than Hot. Data is **offline** — takes hours to access
  (rehydration required). Use for regulatory archives.

### Change a Blob's Tier

```bash
# Change a blob to Cool tier
az storage blob set-tier \
  --account-name mystorageaccount123 \
  --container-name mycontainer \
  --name myfile.txt \
  --tier Cool
```

> 💡 **Cost Optimization**: Set up lifecycle management rules to automatically move blobs
> between tiers based on age. See [[#Blob Lifecycle Management]] below.

---

## Hands-On — Working with Blobs

### Step 1: Create a Container

```bash
# Create a blob container
az storage container create \
  --name mycontainer \
  --account-name mystorageaccount123 \
  --auth-mode login
```

> The `--auth-mode login` flag uses your Azure AD credentials instead of account keys.
> This is the recommended approach. Alternatively, set the environment variable:
> `export AZURE_STORAGE_ACCOUNT=mystorageaccount123`

```bash
# List containers in a storage account
az storage container list \
  --account-name mystorageaccount123 \
  --output table
```

### Step 2: Upload Files

```bash
# Upload a single file
az storage blob upload \
  --account-name mystorageaccount123 \
  --container-name mycontainer \
  --name myfile.txt \
  --file ./myfile.txt \
  --auth-mode login

# Upload with a "folder" path (virtual directory)
az storage blob upload \
  --account-name mystorageaccount123 \
  --container-name mycontainer \
  --name backups/2024/january/backup.sql \
  --file ./backup.sql \
  --auth-mode login

# Upload all files in a directory
az storage blob upload-batch \
  --account-name mystorageaccount123 \
  --destination mycontainer \
  --source ./my-local-folder \
  --auth-mode login
```

### Step 3: List Blobs

```bash
# List all blobs in a container
az storage blob list \
  --account-name mystorageaccount123 \
  --container-name mycontainer \
  --output table \
  --auth-mode login

# List blobs with a prefix (like listing a folder)
az storage blob list \
  --account-name mystorageaccount123 \
  --container-name mycontainer \
  --prefix "backups/2024/" \
  --output table \
  --auth-mode login
```

### Step 4: Download Files

```bash
# Download a single file
az storage blob download \
  --account-name mystorageaccount123 \
  --container-name mycontainer \
  --name myfile.txt \
  --file ./downloaded-file.txt \
  --auth-mode login

# Download all blobs in a container
az storage blob download-batch \
  --account-name mystorageaccount123 \
  --source mycontainer \
  --destination ./downloaded-folder \
  --auth-mode login
```

### Step 5: Delete Blobs

```bash
# Delete a specific blob
az storage blob delete \
  --account-name mystorageaccount123 \
  --container-name mycontainer \
  --name myfile.txt \
  --auth-mode login

# Delete a container (and ALL blobs in it!)
az storage container delete \
  --name mycontainer \
  --account-name mystorageaccount123 \
  --auth-mode login
```

### Step 6: Copy Blobs

```bash
# Copy a blob within the same account
az storage blob copy start \
  --account-name mystorageaccount123 \
  --destination-container mycontainer2 \
  --destination-blob copy-of-myfile.txt \
  --source-uri "https://mystorageaccount123.blob.core.windows.net/mycontainer/myfile.txt"
```

---

## Blob Lifecycle Management

### What is Lifecycle Management?

Lifecycle management lets you create **rules** that automatically:
- Move blobs to cooler tiers as they age
- Delete blobs after a certain number of days
- Delete old blob versions or snapshots

This is crucial for cost optimization!

### Example: Auto-Tier and Auto-Delete

```bash
# Create a lifecycle management policy (JSON format)
az storage account management-policy create \
  --account-name mystorageaccount123 \
  --resource-group myResourceGroup \
  --policy '{
    "rules": [
      {
        "name": "move-to-cool",
        "type": "Lifecycle",
        "definition": {
          "actions": {
            "baseBlob": {
              "tierToCool": { "daysAfterModificationGreaterThan": 30 },
              "tierToArchive": { "daysAfterModificationGreaterThan": 180 },
              "delete": { "daysAfterModificationGreaterThan": 365 }
            }
          },
          "filters": {
            "blobTypes": ["blockBlob"],
            "prefixMatch": ["logs/", "backups/"]
          }
        }
      }
    ]
  }'
```

This policy says:
- Blobs in `logs/` and `backups/` prefixes:
  - After 30 days → Move to **Cool** tier
  - After 180 days → Move to **Archive** tier
  - After 365 days → **Delete** automatically

### View Lifecycle Policies

```bash
# View the current lifecycle policy
az storage account management-policy show \
  --account-name mystorageaccount123 \
  --resource-group myResourceGroup
```

---

## Blob Security — SAS Tokens

### What is a SAS Token?

A **Shared Access Signature (SAS)** is a token appended to a blob URL that grants **temporary,
limited access** to the blob. It's the Azure equivalent of AWS pre-signed URLs.

Use SAS tokens when:
- You want to give someone temporary access to a file without giving them your account key.
- Your web app needs to let users download files directly from storage.
- You want to grant access with specific permissions and an expiry time.

### Generate a SAS Token

```bash
# Generate a SAS token for a specific blob
az storage blob generate-sas \
  --account-name mystorageaccount123 \
  --container-name mycontainer \
  --name myfile.txt \
  --permissions r \
  --expiry 2025-12-31T23:59:59Z \
  --auth-mode login \
  --as-user \
  --output tsv
```

Parameters:
- `--permissions` — What the token allows: `r` (read), `w` (write), `d` (delete), `l` (list)
- `--expiry` — When the token expires (ISO 8601 format)

The output is a SAS token string. Append it to the blob URL:

```
https://mystorageaccount123.blob.core.windows.net/mycontainer/myfile.txt?<SAS-token>
```

Anyone with this URL can access the file until the expiry time.

### Generate a SAS Token for a Container

```bash
# SAS for an entire container (can list and read all blobs)
az storage container generate-sas \
  --account-name mystorageaccount123 \
  --name mycontainer \
  --permissions rl \
  --expiry 2025-12-31T23:59:59Z \
  --auth-mode login \
  --as-user \
  --output tsv
```

### Immutable Storage (WORM)

For compliance requirements, Azure supports **immutable blob storage**:

- **WORM** = Write Once, Read Many
- Once written, blobs **cannot be modified or deleted** for a specified retention period.
- Used for: Financial records, medical records, legal documents, regulatory compliance.

```bash
# Set an immutability policy on a container (retain for 365 days)
az storage container immutability-policy create \
  --account-name mystorageaccount123 \
  --container-name compliance-data \
  --period 365 \
  --resource-group myResourceGroup
```

---

## Azure Managed Disks

### What are Managed Disks?

**Managed Disks** are block storage volumes for Azure VMs. They're the equivalent of **AWS EBS**
(Elastic Block Store).

When you create a VM, Azure automatically creates a managed disk for the OS. You can also
attach additional data disks.

### Disk Types

| Type | IOPS (max) | Throughput | Cost | Use Case |
|------|-----------|------------|------|----------|
| **Standard HDD** | 500 | 60 MB/s | Lowest | Dev/test, infrequent access |
| **Standard SSD** | 6,000 | 750 MB/s | Low | Web servers, light production |
| **Premium SSD** | 20,000 | 900 MB/s | Medium | Production databases, high-performance |
| **Premium SSD v2** | 80,000 | 1,200 MB/s | Medium-High | High-performance, tunable IOPS |
| **Ultra Disk** | 160,000 | 4,000 MB/s | Highest | SAP HANA, top-tier databases |

### Hands-On: Manage Disks

```bash
# Create a premium SSD managed disk
az disk create \
  --resource-group myResourceGroup \
  --name myDataDisk \
  --size-gb 32 \
  --sku Premium_LRS

# List disks in a resource group
az disk list --resource-group myResourceGroup --output table

# Attach a disk to a VM
az vm disk attach \
  --resource-group myResourceGroup \
  --vm-name myVM \
  --name myDataDisk

# Detach a disk from a VM
az vm disk detach \
  --resource-group myResourceGroup \
  --vm-name myVM \
  --name myDataDisk

# Delete a disk
az disk delete --resource-group myResourceGroup --name myDataDisk --yes
```

### After Attaching a Disk to a Linux VM

The disk is attached but not usable yet — you need to format and mount it:

```bash
# SSH into the VM, then:

# Find the new disk (usually /dev/sdc)
lsblk

# Create a partition
sudo fdisk /dev/sdc
# Type: n (new), p (primary), 1 (partition 1), Enter, Enter, w (write)

# Format with ext4
sudo mkfs.ext4 /dev/sdc1

# Create mount point and mount
sudo mkdir /mnt/data
sudo mount /dev/sdc1 /mnt/data

# Make it permanent (survive reboot)
echo '/dev/sdc1 /mnt/data ext4 defaults 0 0' | sudo tee -a /etc/fstab
```

### Snapshots

Create point-in-time backups of your disks:

```bash
# Create a snapshot of a disk
az snapshot create \
  --resource-group myResourceGroup \
  --name myDiskSnapshot \
  --source myDataDisk

# List snapshots
az snapshot list --resource-group myResourceGroup --output table

# Create a new disk from a snapshot
az disk create \
  --resource-group myResourceGroup \
  --name myRestoredDisk \
  --source myDiskSnapshot
```

---

## Azure Files

### What is Azure Files?

**Azure Files** is a fully managed file share service that uses the **SMB** (Server Message Block)
or **NFS** (Network File System) protocol. It's the equivalent of **AWS EFS**.

You can mount Azure Files from:
- Windows (native SMB support)
- Linux (SMB or NFS)
- macOS (SMB)

### Use Cases

- Shared configuration files across multiple VMs.
- Lift-and-shift applications that use file shares.
- Shared storage for containers.
- Home directories.
- Development tools and utilities.

### Hands-On: Create and Use File Shares

```bash
# Create a file share (quota in GB)
az storage share-rm create \
  --storage-account mystorageaccount123 \
  --resource-group myResourceGroup \
  --name myshare \
  --quota 5

# List file shares
az storage share-rm list \
  --storage-account mystorageaccount123 \
  --resource-group myResourceGroup \
  --output table
```

### Upload and Download Files

```bash
# Upload a file to the share
az storage file upload \
  --share-name myshare \
  --source ./myfile.txt \
  --account-name mystorageaccount123

# Create a directory in the share
az storage directory create \
  --share-name myshare \
  --name "configs" \
  --account-name mystorageaccount123

# Upload to a subdirectory
az storage file upload \
  --share-name myshare \
  --source ./app.config \
  --path "configs/app.config" \
  --account-name mystorageaccount123

# List files in the share
az storage file list \
  --share-name myshare \
  --account-name mystorageaccount123 \
  --output table

# Download a file
az storage file download \
  --share-name myshare \
  --path "myfile.txt" \
  --dest "./downloaded-myfile.txt" \
  --account-name mystorageaccount123
```

### Mount on Linux

```bash
# Install CIFS utilities
sudo apt install cifs-utils

# Get the storage account key
STORAGE_KEY=$(az storage account keys list \
  --account-name mystorageaccount123 \
  --resource-group myResourceGroup \
  --query "[0].value" --output tsv)

# Create mount point
sudo mkdir -p /mnt/myshare

# Mount the file share
sudo mount -t cifs \
  //mystorageaccount123.file.core.windows.net/myshare \
  /mnt/myshare \
  -o vers=3.0,username=mystorageaccount123,password=$STORAGE_KEY,dir_mode=0777,file_mode=0777,serverino

# Make it permanent
echo "//mystorageaccount123.file.core.windows.net/myshare /mnt/myshare cifs vers=3.0,username=mystorageaccount123,password=$STORAGE_KEY,dir_mode=0777,file_mode=0777,serverino 0 0" | sudo tee -a /etc/fstab
```

### Mount on Windows

```powershell
# In PowerShell or CMD:
net use Z: \\mystorageaccount123.file.core.windows.net\myshare /u:mystorageaccount123 <storage-key>
```

Or use the Azure Portal: Storage Account → File shares → myshare → **Connect** → It generates
the mount script for you.

---

## Azure Queue Storage

### What is Queue Storage?

**Azure Queue Storage** is a simple message queuing service for **decoupling application
components**. It stores messages that can be processed asynchronously.

> 💡 **Analogy**: Think of a queue as a to-do list. One part of your application adds tasks
> (messages) to the list, and another part reads and processes them. They don't need to run
> at the same time.

### Use Cases

- Decoupling a web frontend from a backend processor.
- Handling bursts of traffic — queue messages during peak, process during off-peak.
- Order processing — accept orders immediately, process them in the background.
- Task scheduling — queue tasks for Azure Functions to process.

### Hands-On

```bash
# Create a queue
az storage queue create \
  --name myqueue \
  --account-name mystorageaccount123

# Add a message to the queue
az storage message put \
  --queue-name myqueue \
  --content "Process order #12345" \
  --account-name mystorageaccount123

# Peek at messages (view without removing)
az storage message peek \
  --queue-name myqueue \
  --account-name mystorageaccount123

# Get a message (retrieves and makes it invisible for 30 seconds)
az storage message get \
  --queue-name myqueue \
  --account-name mystorageaccount123

# Delete a message (after processing)
az storage message delete \
  --queue-name myqueue \
  --id <message-id> \
  --pop-receipt <pop-receipt> \
  --account-name mystorageaccount123

# Clear all messages from a queue
az storage message clear \
  --queue-name myqueue \
  --account-name mystorageaccount123

# Delete a queue
az storage queue delete \
  --name myqueue \
  --account-name mystorageaccount123
```

### Queue Characteristics

| Feature | Value |
|---------|-------|
| Max message size | 64 KB |
| Max messages in queue | Unlimited |
| Message TTL | Up to 7 days (default) |
| Max visibility timeout | 7 days |
| Protocol | REST API over HTTP/HTTPS |

> 💡 **Note**: For more advanced messaging needs (topics, subscriptions, dead-letter queues,
> larger messages), use **Azure Service Bus** instead of Queue Storage. Queue Storage is
> simpler and cheaper; Service Bus is more feature-rich.

---

## Azure Table Storage

### What is Table Storage?

**Azure Table Storage** is a simple **NoSQL key-value store** for semi-structured data.
It's cheap and fast for simple queries, but limited compared to a full database.

### When to Use Table Storage?

- Storing device telemetry data
- User profile or session data
- Simple lookup tables
- Any data that doesn't need complex queries, joins, or relationships

### Hands-On

```bash
# Create a table
az storage table create \
  --name mytable \
  --account-name mystorageaccount123

# Insert an entity (row)
az storage entity insert \
  --account-name mystorageaccount123 \
  --table-name mytable \
  --entity PartitionKey=Users RowKey=user001 Name="John Developer" Email="john@example.com"

# Query entities
az storage entity query \
  --account-name mystorageaccount123 \
  --table-name mytable \
  --output table

# Delete a table
az storage table delete \
  --name mytable \
  --account-name mystorageaccount123
```

> 💡 **Tip**: For most use cases, consider **Azure Cosmos DB Table API** instead. It's
> compatible with Table Storage but offers global distribution, higher throughput, and SLAs.

---

## Storage Comparison — Azure vs AWS

| Category | Azure Service | AWS Equivalent | Protocol / Type |
|----------|--------------|----------------|-----------------|
| **Object Storage** | Blob Storage | S3 | REST API, HTTP |
| **Block Storage** | Managed Disks | EBS | Block-level, attached to VMs |
| **File Storage** | Azure Files | EFS | SMB / NFS, mountable |
| **Message Queue** | Queue Storage | SQS | Simple queue |
| **NoSQL Key-Value** | Table Storage | DynamoDB (basic) | Key-value store |
| **Archive Storage** | Blob Archive tier | S3 Glacier | Offline, hours to retrieve |
| **Temp access URLs** | SAS Tokens | Pre-signed URLs | Temporary authenticated URLs |

### Detailed Comparison: Blob Storage vs S3

| Feature | Azure Blob Storage | AWS S3 |
|---------|-------------------|--------|
| **Container/Bucket** | Container (within storage account) | Bucket (globally unique) |
| **Access Tiers** | Hot, Cool, Cold, Archive | Standard, IA, Glacier, Deep Archive |
| **Versioning** | Yes (enable on container) | Yes (enable on bucket) |
| **Lifecycle Rules** | Yes | Yes |
| **Static Website** | Yes | Yes |
| **CDN Integration** | Azure CDN | CloudFront |
| **Encryption** | AES-256 by default | AES-256 by default |
| **Max Object Size** | 190.7 TB (block blob) | 5 TB |
| **Temporary Access** | SAS tokens | Pre-signed URLs |

For a complete comparison, see [[AWS vs Azure Comparison]].

---

## Cost Optimization Tips

### 1. Choose the Right Redundancy

Don't use GRS or RA-GRS for dev/test — LRS is significantly cheaper:

| Redundancy | Relative Cost |
|-----------|---------------|
| LRS | 1x (baseline) |
| ZRS | ~1.25x |
| GRS | ~2x |
| RA-GRS | ~2.5x |

### 2. Use Access Tiers Wisely

- Don't store everything in Hot tier.
- Set up lifecycle management to automatically move old data to Cool/Cold/Archive.
- Archive tier is ~90% cheaper than Hot for storage (but has rehydration costs and delays).

### 3. Clean Up Old Data

```bash
# Find large blobs
az storage blob list \
  --account-name mystorageaccount123 \
  --container-name mycontainer \
  --output table \
  --query "[].{Name:name, Size:properties.contentLength}"
```

### 4. Use Reserved Capacity

For predictable storage workloads, you can purchase **reserved capacity** at a discount:
- 1-year reservation: ~23% savings
- 3-year reservation: ~38% savings

### 5. Monitor Costs

```bash
# View storage account metrics
az monitor metrics list \
  --resource /subscriptions/<sub-id>/resourceGroups/myResourceGroup/providers/Microsoft.Storage/storageAccounts/mystorageaccount123 \
  --metric UsedCapacity \
  --output table
```

Check **Azure Cost Management** in the portal: Subscriptions → Cost analysis → Filter by
resource type = Storage Accounts.

---

## What's Next?

- 💻 [[Azure Compute]] — Use storage with your VMs and applications
- 🌐 [[Azure Networking]] — Secure access to your storage accounts
- 🗄️ [[Azure Databases]] — When you need structured data storage
- 🔐 [[Azure IAM]] — Control who can access your storage
- 💰 [[Azure Billing]] — Monitor and optimize storage costs

---

## Quick Reference Card

```bash
# ===== STORAGE ACCOUNT =====
az storage account create -n mystorageacct -g myRG -l centralindia --sku Standard_LRS --kind StorageV2
az storage account show-connection-string -n mystorageacct -g myRG
az storage account keys list -n mystorageacct -g myRG -o table
az storage account delete -n mystorageacct -g myRG --yes

# ===== BLOB STORAGE =====
az storage container create -n mycontainer --account-name mystorageacct --auth-mode login
az storage blob upload --account-name mystorageacct -c mycontainer -n myfile.txt -f ./myfile.txt --auth-mode login
az storage blob list --account-name mystorageacct -c mycontainer -o table --auth-mode login
az storage blob download --account-name mystorageacct -c mycontainer -n myfile.txt -f ./out.txt --auth-mode login
az storage blob delete --account-name mystorageacct -c mycontainer -n myfile.txt --auth-mode login
az storage blob set-tier --account-name mystorageacct -c mycontainer -n myfile.txt --tier Cool

# ===== MANAGED DISKS =====
az disk create -g myRG -n myDisk --size-gb 32 --sku Premium_LRS
az vm disk attach -g myRG --vm-name myVM -n myDisk
az vm disk detach -g myRG --vm-name myVM -n myDisk
az snapshot create -g myRG -n mySnap --source myDisk

# ===== FILE SHARES =====
az storage share-rm create --storage-account mystorageacct -g myRG -n myshare --quota 5
az storage file upload --share-name myshare --source ./myfile.txt --account-name mystorageacct
az storage file list --share-name myshare --account-name mystorageacct -o table

# ===== QUEUE STORAGE =====
az storage queue create -n myqueue --account-name mystorageacct
az storage message put --queue-name myqueue --content "Hello" --account-name mystorageacct
```

---

**Tags**: #azure #storage #blob #files #disks #queue #beginner
**Related**: [[Cloud Fundamentals]] | [[Azure Getting Started]] | [[Azure Compute]] | [[AWS Storage]]
