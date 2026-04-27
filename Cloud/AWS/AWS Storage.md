# AWS Storage

> AWS offers three main types of storage: **Object** (S3), **Block** (EBS), and **File** (EFS). Each serves different use cases — from hosting static websites to attaching virtual hard drives to sharing files across servers.

---

## Cloud Storage Types

Before diving into specific services, understand the three fundamental types of cloud storage:

### Comparison of Storage Types

| Type            | How It Works                        | Analogy                  | AWS Service |
| --------------- | ----------------------------------- | ------------------------ | ----------- |
| **Object Storage** | Stores files as objects with metadata | Google Drive / Dropbox  | **S3**      |
| **Block Storage**  | Stores data in fixed-size blocks (like a hard drive) | USB drive / SSD | **EBS**     |
| **File Storage**   | Shared file system accessible by multiple servers | Network drive (NAS) | **EFS**     |

### When to Use Each

```
Need to store images, videos, backups, logs?
  → Object Storage (S3)

Need a hard drive for your EC2 instance (OS, database)?
  → Block Storage (EBS)

Need multiple EC2 instances to share the same files?
  → File Storage (EFS)
```

---

## Amazon S3 (Simple Storage Service)

**S3** is AWS's object storage service. It's one of the oldest and most popular AWS services — nearly every AWS architecture uses S3 in some way.

### What is Object Storage?

Unlike a traditional file system with folders and hierarchy, S3 stores data as **objects** in **buckets**:

```
Traditional File System:          S3 Object Storage:
/home/                            my-bucket/
  └── user/                         ├── photos/sunset.jpg     (object)
       ├── photos/                  ├── photos/beach.jpg      (object)
       │    ├── sunset.jpg          ├── documents/report.pdf  (object)
       │    └── beach.jpg           └── logs/app.log          (object)
       └── documents/
            └── report.pdf
```

> 💡 S3 doesn't actually have folders — the "/" in keys like `photos/sunset.jpg` is just part of the object's name (key). The console shows it as folders for convenience.

### S3 Key Concepts

| Concept       | Description                                                        |
| ------------- | ------------------------------------------------------------------ |
| **Bucket**    | A container for objects. Name must be **globally unique** across ALL AWS accounts |
| **Object**    | A file + metadata. Can be up to **5 TB**                           |
| **Key**       | The unique identifier (path) of an object within a bucket          |
| **Region**    | Buckets are created in a specific region (data stays there)        |
| **Durability**| 99.999999999% (11 nines) — designed to never lose your data       |
| **Availability** | 99.99% for Standard class                                      |

### S3 Storage Classes

Choose the right storage class based on how frequently you access data and how quickly you need it:

| Storage Class              | Access Pattern          | Min Storage | Retrieval Time | Cost (GB/month)* |
| -------------------------- | ----------------------- | ----------- | -------------- | ----------------- |
| **S3 Standard**            | Frequent access         | None        | Milliseconds   | $0.023            |
| **S3 Intelligent-Tiering** | Unknown/changing        | None        | Milliseconds   | $0.023 + monitoring fee |
| **S3 Standard-IA**         | Infrequent access       | 30 days     | Milliseconds   | $0.0125           |
| **S3 One Zone-IA**         | Infrequent, non-critical| 30 days     | Milliseconds   | $0.01             |
| **S3 Glacier Instant**     | Archive, instant access | 90 days     | Milliseconds   | $0.004            |
| **S3 Glacier Flexible**    | Archive                 | 90 days     | Minutes-hours  | $0.0036           |
| **S3 Glacier Deep Archive**| Long-term archive       | 180 days    | 12-48 hours    | $0.00099          |

*Approximate prices — vary by region. Check AWS pricing page for current rates.

#### Choosing the Right Storage Class

```
How often do you access this data?

Multiple times per day → S3 Standard
  Example: Website assets, application data, frequently-read logs

Unpredictable access patterns → S3 Intelligent-Tiering
  Example: Data lakes, new applications, user-generated content

Once a month or less → S3 Standard-IA (Infrequent Access)
  Example: Backups, disaster recovery, older logs

Once a month, non-critical → S3 One Zone-IA
  Example: Re-creatable thumbnails, secondary backups

Once a quarter, need instant access → S3 Glacier Instant Retrieval
  Example: Medical images, news archives

Once a year → S3 Glacier Flexible Retrieval
  Example: Compliance archives, old backups

Rarely/never (regulatory) → S3 Glacier Deep Archive
  Example: 7-year financial records, legal archives
```

---

### Hands-On: Create and Use an S3 Bucket

#### Creating a Bucket

```bash
# Create a bucket (name must be globally unique!)
aws s3 mb s3://my-unique-bucket-name-12345

# Create a bucket in a specific region
aws s3 mb s3://my-unique-bucket-name-12345 --region ap-south-1

# List all your buckets
aws s3 ls
```

> ⚠️ Bucket naming rules:
> - Must be globally unique across ALL AWS accounts
> - 3-63 characters long
> - Only lowercase letters, numbers, and hyphens
> - Must start with a letter or number
> - No periods (.) for best compatibility

#### Uploading Files

```bash
# Upload a single file
aws s3 cp myfile.txt s3://my-unique-bucket-name-12345/

# Upload to a "folder" (prefix)
aws s3 cp myfile.txt s3://my-unique-bucket-name-12345/documents/

# Upload with a specific storage class
aws s3 cp largefile.zip s3://my-unique-bucket-name-12345/ --storage-class STANDARD_IA

# Upload all files in a directory
aws s3 cp ./my-folder s3://my-unique-bucket-name-12345/my-folder/ --recursive

# Upload only certain file types
aws s3 cp ./website s3://my-unique-bucket-name-12345/ --recursive --exclude "*" --include "*.html" --include "*.css"
```

#### Listing Objects

```bash
# List all objects in a bucket
aws s3 ls s3://my-unique-bucket-name-12345/

# List objects in a "folder"
aws s3 ls s3://my-unique-bucket-name-12345/documents/

# List recursively (all objects including in subfolders)
aws s3 ls s3://my-unique-bucket-name-12345/ --recursive

# List with human-readable file sizes
aws s3 ls s3://my-unique-bucket-name-12345/ --recursive --human-readable --summarize
```

#### Downloading Files

```bash
# Download a single file
aws s3 cp s3://my-unique-bucket-name-12345/myfile.txt ./downloaded.txt

# Download an entire "folder"
aws s3 cp s3://my-unique-bucket-name-12345/documents/ ./local-docs/ --recursive

# Download to current directory
aws s3 cp s3://my-unique-bucket-name-12345/myfile.txt .
```

#### Syncing Directories

```bash
# Sync local directory TO S3 (upload new/changed files)
aws s3 sync ./my-website s3://my-unique-bucket-name-12345/

# Sync S3 TO local directory (download new/changed files)
aws s3 sync s3://my-unique-bucket-name-12345/ ./local-backup/

# Sync with delete (remove files in destination that don't exist in source)
aws s3 sync ./my-website s3://my-unique-bucket-name-12345/ --delete

# Dry run — see what WOULD happen without actually doing it
aws s3 sync ./my-website s3://my-unique-bucket-name-12345/ --dryrun
```

#### Moving and Deleting

```bash
# Move a file (copy + delete source)
aws s3 mv s3://my-bucket/old-name.txt s3://my-bucket/new-name.txt

# Move between buckets
aws s3 mv s3://bucket-a/file.txt s3://bucket-b/file.txt

# Delete a single file
aws s3 rm s3://my-unique-bucket-name-12345/myfile.txt

# Delete all files in a "folder"
aws s3 rm s3://my-unique-bucket-name-12345/old-logs/ --recursive

# Delete a bucket (must be empty first)
aws s3 rb s3://my-unique-bucket-name-12345

# Force delete a bucket and ALL its contents (CAREFUL!)
aws s3 rb s3://my-unique-bucket-name-12345 --force
```

---

### S3 Bucket Policies

Bucket policies control **who** can access your bucket and **what** they can do. They're JSON documents similar to IAM policies (see [[AWS IAM]]).

#### Example: Public Read Access (for a static website)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::my-website-bucket/*"
        }
    ]
}
```

#### Example: Restrict Access to Specific IP

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowFromOfficeIP",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::my-bucket",
                "arn:aws:s3:::my-bucket/*"
            ],
            "Condition": {
                "IpAddress": {
                    "aws:SourceIp": "203.0.113.0/24"
                }
            }
        }
    ]
}
```

#### Apply a Bucket Policy

```bash
# Apply a bucket policy from a JSON file
aws s3api put-bucket-policy --bucket my-bucket --policy file://bucket-policy.json

# View the current bucket policy
aws s3api get-bucket-policy --bucket my-bucket

# Delete the bucket policy
aws s3api delete-bucket-policy --bucket my-bucket
```

---

### S3 Versioning

**Versioning** keeps multiple versions of every object. If you accidentally delete or overwrite a file, you can recover a previous version.

```bash
# Enable versioning on a bucket
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled

# Check versioning status
aws s3api get-bucket-versioning --bucket my-bucket

# List all versions of objects
aws s3api list-object-versions --bucket my-bucket

# Download a specific version
aws s3api get-object \
  --bucket my-bucket \
  --key myfile.txt \
  --version-id "abc123version" \
  downloaded-old-version.txt

# Delete a specific version (permanent delete)
aws s3api delete-object \
  --bucket my-bucket \
  --key myfile.txt \
  --version-id "abc123version"
```

> 💡 **Important**: Deleting an object in a versioned bucket adds a "delete marker" — the object appears deleted but previous versions still exist. This uses storage space!

---

### S3 Lifecycle Rules

**Lifecycle rules** automatically transition objects between storage classes or delete them after a specified time.

#### Example: Auto-Archive Old Data

```json
{
    "Rules": [
        {
            "ID": "ArchiveOldData",
            "Status": "Enabled",
            "Filter": {
                "Prefix": "logs/"
            },
            "Transitions": [
                {
                    "Days": 30,
                    "StorageClass": "STANDARD_IA"
                },
                {
                    "Days": 90,
                    "StorageClass": "GLACIER"
                },
                {
                    "Days": 365,
                    "StorageClass": "DEEP_ARCHIVE"
                }
            ],
            "Expiration": {
                "Days": 2555
            }
        }
    ]
}
```

This rule:
- After 30 days → Move to Standard-IA (cheaper)
- After 90 days → Move to Glacier (much cheaper)
- After 365 days → Move to Deep Archive (cheapest)
- After 7 years → Delete permanently

```bash
# Apply lifecycle configuration
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration file://lifecycle.json

# View lifecycle configuration
aws s3api get-bucket-lifecycle-configuration --bucket my-bucket
```

---

### S3 Static Website Hosting

You can host a static website (HTML, CSS, JS) directly from S3 — no servers needed!

```bash
# Step 1: Enable static website hosting
aws s3 website s3://my-website-bucket \
  --index-document index.html \
  --error-document error.html

# Step 2: Upload your website files
aws s3 sync ./website s3://my-website-bucket/

# Step 3: Apply a public-read bucket policy (see Bucket Policies section above)

# Step 4: Access your website at:
# http://my-website-bucket.s3-website-ap-south-1.amazonaws.com
```

> ⚠️ S3 static websites use HTTP (not HTTPS). For HTTPS, put CloudFront in front of S3.

---

### Pre-signed URLs

**Pre-signed URLs** give temporary access to private S3 objects. Perfect for sharing files securely without making them public.

```bash
# Generate a pre-signed URL (default: 1 hour expiry)
aws s3 presign s3://my-bucket/private-file.pdf

# Generate with custom expiry (in seconds)
aws s3 presign s3://my-bucket/private-file.pdf --expires-in 3600

# The output is a long URL anyone can use to download the file (until it expires)
# https://my-bucket.s3.amazonaws.com/private-file.pdf?X-Amz-Algorithm=...&X-Amz-Expires=3600&...
```

Use cases:
- Share a report with a client temporarily
- Allow file downloads from a web application
- Provide time-limited upload access

---

### S3 Encryption

#### Server-Side Encryption (SSE)

| Type          | Key Management            | Use Case                        |
| ------------- | ------------------------- | ------------------------------- |
| **SSE-S3**    | AWS manages keys          | Default, simplest option        |
| **SSE-KMS**   | AWS KMS manages keys      | Audit key usage, key rotation   |
| **SSE-C**     | Customer provides keys    | Full control of encryption keys |

```bash
# Upload with SSE-S3 encryption (default for new buckets)
aws s3 cp myfile.txt s3://my-bucket/ --sse AES256

# Upload with SSE-KMS encryption
aws s3 cp myfile.txt s3://my-bucket/ --sse aws:kms

# Set default encryption for the entire bucket
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'
```

---

### S3 Best Practices

```
✅ 1. Block public access by default
      aws s3api put-public-access-block --bucket my-bucket \
        --public-access-block-configuration \
        "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

✅ 2. Enable versioning on important buckets
✅ 3. Set up lifecycle policies to save costs
✅ 4. Enable server-side encryption (default for new buckets)
✅ 5. Use bucket policies and IAM for access control
✅ 6. Enable access logging for audit trails
✅ 7. Use S3 Intelligent-Tiering if access patterns are unknown
✅ 8. Tag your buckets for cost allocation
✅ 9. Use pre-signed URLs instead of making objects public
✅ 10. Monitor costs with S3 Storage Lens
```

---

## Amazon EBS (Elastic Block Store)

**EBS** provides persistent block storage volumes for EC2 instances. Think of it as a virtual hard drive that attaches to your virtual server.

### Key Characteristics

| Feature          | Description                                                     |
| ---------------- | --------------------------------------------------------------- |
| **Persistence**  | Data survives instance stop/restart                             |
| **Attachment**   | One volume → one instance at a time (mostly)                    |
| **AZ-bound**     | Volume must be in the same Availability Zone as the instance    |
| **Snapshots**    | Backup volumes to S3 (incremental, cross-AZ/region)            |
| **Encryption**   | AES-256 encryption supported                                   |
| **Resizable**    | Can increase size without downtime                              |

### EBS Volume Types

| Type             | Code    | IOPS (max)    | Throughput    | Use Case                          |
| ---------------- | ------- | ------------- | ------------- | --------------------------------- |
| **General Purpose SSD** | `gp3`  | 16,000  | 1,000 MB/s    | Boot volumes, dev/test, most workloads |
| **General Purpose SSD** | `gp2`  | 16,000  | 250 MB/s      | Legacy — use gp3 instead          |
| **Provisioned IOPS SSD**| `io2`  | 64,000  | 1,000 MB/s    | Databases needing consistent IOPS |
| **Throughput HDD**      | `st1`  | 500     | 500 MB/s      | Big data, data warehouses, logs   |
| **Cold HDD**            | `sc1`  | 250     | 250 MB/s      | Infrequently accessed data        |

> 💡 **For beginners**: Always use **gp3** — it's the best balance of price and performance.

### Hands-On: Working with EBS Volumes

```bash
# ===== Create a Volume =====
aws ec2 create-volume \
  --size 20 \
  --volume-type gp3 \
  --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=MyDataVolume}]'
# Note the VolumeId from the output (e.g., vol-0abcdef1234567890)

# ===== Attach to an Instance =====
aws ec2 attach-volume \
  --volume-id vol-0abcdef1234567890 \
  --instance-id i-0abcdef1234567890 \
  --device /dev/xvdf

# ===== On the EC2 Instance — Format and Mount =====
# SSH into the instance first, then:
# Check if the volume has a file system
sudo file -s /dev/xvdf
# If output is "data", it needs formatting:
sudo mkfs -t xfs /dev/xvdf

# Create a mount point and mount
sudo mkdir /data
sudo mount /dev/xvdf /data

# Make it persist across reboots
echo '/dev/xvdf /data xfs defaults,nofail 0 2' | sudo tee -a /etc/fstab

# ===== Verify =====
df -h   # Shows mounted volumes
lsblk   # Shows block devices
```

### EBS Snapshots

Snapshots are **incremental backups** of your EBS volumes stored in S3. Only changed blocks are saved after the first snapshot.

```bash
# Create a snapshot
aws ec2 create-snapshot \
  --volume-id vol-0abcdef1234567890 \
  --description "Daily backup $(date +%Y-%m-%d)" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=DailyBackup}]'

# List your snapshots
aws ec2 describe-snapshots --owner-ids self --output table

# Create a new volume from a snapshot (disaster recovery!)
aws ec2 create-volume \
  --snapshot-id snap-0abcdef1234567890 \
  --availability-zone ap-south-1a \
  --volume-type gp3

# Copy a snapshot to another region (cross-region DR)
aws ec2 copy-snapshot \
  --source-region ap-south-1 \
  --source-snapshot-id snap-0abcdef1234567890 \
  --destination-region us-east-1 \
  --description "DR copy"

# Delete a snapshot
aws ec2 delete-snapshot --snapshot-id snap-0abcdef1234567890
```

### Modifying EBS Volumes

```bash
# Increase volume size (no downtime required)
aws ec2 modify-volume \
  --volume-id vol-0abcdef1234567890 \
  --size 50

# After modification, extend the file system on the instance:
# For XFS:
sudo xfs_growfs /data
# For EXT4:
sudo resize2fs /dev/xvdf

# Change volume type (e.g., gp2 to gp3)
aws ec2 modify-volume \
  --volume-id vol-0abcdef1234567890 \
  --volume-type gp3

# Check modification progress
aws ec2 describe-volumes-modifications --volume-ids vol-0abcdef1234567890
```

### Clean Up EBS Resources

```bash
# Detach a volume from an instance
aws ec2 detach-volume --volume-id vol-0abcdef1234567890

# Delete a volume (must be detached first)
aws ec2 delete-volume --volume-id vol-0abcdef1234567890
```

> ⚠️ **Cost Warning**: Unattached EBS volumes still cost money! Regularly audit and delete volumes you no longer need.

---

## Amazon EFS (Elastic File System)

**EFS** is a fully-managed, scalable file storage service. Unlike EBS (one instance at a time), EFS can be mounted by **multiple EC2 instances simultaneously**.

### Key Characteristics

| Feature          | Description                                                     |
| ---------------- | --------------------------------------------------------------- |
| **Protocol**     | NFS v4.1 (Network File System)                                 |
| **Shared Access**| Multiple EC2 instances can mount the same file system           |
| **Scalability**  | Automatically grows and shrinks as you add/remove files         |
| **Durability**   | Data replicated across multiple AZs                             |
| **Performance**  | Two modes: General Purpose (default) and Max I/O                |
| **Cost**         | Pay for what you store (no pre-provisioning)                    |

### When to Use EFS

| Scenario                                       | Use EFS? |
| ---------------------------------------------- | -------- |
| Multiple servers need to share configuration   | ✅ Yes   |
| Web servers sharing uploaded content           | ✅ Yes   |
| Machine learning training data shared across instances | ✅ Yes   |
| Database storage for MySQL/PostgreSQL          | ❌ Use EBS |
| Archiving old files cheaply                    | ❌ Use S3 |
| Boot volume for EC2                            | ❌ Use EBS |

### EFS Storage Classes

| Storage Class      | Use Case                      | Cost (GB/month)* |
| ------------------ | ----------------------------- | ----------------- |
| **Standard**       | Frequently accessed files     | $0.30             |
| **Standard-IA**    | Infrequently accessed files   | $0.025            |
| **One Zone**       | Frequently accessed, one AZ   | $0.16             |
| **One Zone-IA**    | Infrequent access, one AZ     | $0.0133           |

*Approximate US East prices.

### Hands-On: Create and Use EFS

```bash
# ===== Step 1: Create the File System =====
aws efs create-file-system \
  --creation-token my-efs-token \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --tags Key=Name,Value=MySharedStorage
# Note the FileSystemId (e.g., fs-0abcdef1234567890)

# ===== Step 2: Create Mount Targets (one per AZ) =====
# Your EC2 instances connect through these
aws efs create-mount-target \
  --file-system-id fs-0abcdef1234567890 \
  --subnet-id subnet-xxx \
  --security-groups sg-xxx

# ===== Step 3: Mount on EC2 Instances =====
# SSH into your EC2 instance, then:

# Install the EFS mount helper (Amazon Linux)
sudo yum install -y amazon-efs-utils

# Create a mount point
sudo mkdir /mnt/efs

# Mount using the EFS helper (recommended)
sudo mount -t efs fs-0abcdef1234567890:/ /mnt/efs

# OR mount using standard NFS
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 \
  fs-0abcdef1234567890.efs.ap-south-1.amazonaws.com:/ /mnt/efs

# ===== Step 4: Make the Mount Persistent =====
echo 'fs-0abcdef1234567890:/ /mnt/efs efs defaults,_netdev 0 0' | sudo tee -a /etc/fstab

# ===== Step 5: Test It =====
# On Instance 1:
echo "Hello from Instance 1" | sudo tee /mnt/efs/test.txt

# On Instance 2 (mounted to the same EFS):
cat /mnt/efs/test.txt
# Output: Hello from Instance 1
# Both instances see the same files!
```

### Managing EFS

```bash
# List all file systems
aws efs describe-file-systems --output table

# Describe mount targets for a file system
aws efs describe-mount-targets --file-system-id fs-0abcdef1234567890

# Delete a file system (must delete mount targets first)
aws efs delete-mount-target --mount-target-id fsmt-xxx
aws efs delete-file-system --file-system-id fs-0abcdef1234567890
```

---

## Storage Comparison Table

| Feature              | S3 (Object)              | EBS (Block)              | EFS (File)               |
| -------------------- | ------------------------ | ------------------------ | ------------------------ |
| **Type**             | Object storage           | Block storage            | File storage             |
| **Access**           | HTTP/HTTPS API           | Attached to one EC2      | NFS — multiple EC2       |
| **Max Size**         | Unlimited (5TB per object)| 64 TB per volume        | Unlimited (auto-scales)  |
| **Durability**       | 99.999999999% (11 nines) | 99.999%                 | 99.999999999% (11 nines) |
| **Availability**     | 99.99%                   | 99.999%                 | 99.99%                   |
| **Performance**      | High throughput          | Low-latency IOPS        | Scalable throughput      |
| **AZ Scope**         | Regional                 | Single AZ               | Multi-AZ                 |
| **Shared Access**    | Yes (via API)            | No (one instance)       | Yes (multiple instances) |
| **Pricing Model**    | Per GB stored + requests | Per GB provisioned      | Per GB stored            |
| **Backup**           | Versioning, replication  | Snapshots to S3         | AWS Backup               |
| **Use Cases**        | Static assets, backups, data lakes | Boot volumes, databases | Shared config, CMS, ML data |
| **Best Analogy**     | Google Drive             | USB Hard Drive          | Network Shared Drive     |

### Decision Guide

```
"Where should I put my data?"

Is it a file that many servers need to access at once?
  → EFS

Is it a disk that one EC2 instance needs (OS, database, application)?
  → EBS

Is it an object (image, video, log, backup, static website)?
  → S3

Am I not sure?
  → Start with S3 (most versatile, cheapest)
```

---

## AWS Storage Gateway (Bonus)

**Storage Gateway** is a hybrid service that bridges on-premises storage with AWS cloud storage. Useful for companies migrating to the cloud gradually.

| Gateway Type      | Use Case                                      |
| ----------------- | --------------------------------------------- |
| **S3 File Gateway** | Access S3 objects as files via NFS/SMB       |
| **FSx File Gateway** | Access FSx for Windows from on-premises     |
| **Volume Gateway**   | iSCSI block storage backed by S3/EBS       |
| **Tape Gateway**     | Replace physical tape backups with S3/Glacier|

---

## Hands-On: Clean Up Storage Resources

Always clean up resources when you're done experimenting:

```bash
# ===== S3 =====
# Empty and delete a bucket
aws s3 rm s3://my-unique-bucket-name-12345 --recursive
aws s3 rb s3://my-unique-bucket-name-12345

# ===== EBS =====
# Detach and delete a volume
aws ec2 detach-volume --volume-id vol-xxx
aws ec2 delete-volume --volume-id vol-xxx

# Delete old snapshots
aws ec2 delete-snapshot --snapshot-id snap-xxx

# ===== EFS =====
# Delete mount targets first, then file system
aws efs delete-mount-target --mount-target-id fsmt-xxx
aws efs delete-file-system --file-system-id fs-xxx
```

---

## What's Next?

- [[AWS Compute]] — Use storage with EC2 instances and Lambda
- [[AWS Networking]] — Network configuration for accessing storage
- [[AWS Databases]] — RDS, DynamoDB and other managed data stores
- [[AWS Getting Started]] — Back to basics

---

## Related Notes

- [[Cloud Fundamentals]] — How storage fits into cloud architecture
- [[AWS Getting Started]] — Account setup and CLI configuration
- [[AWS Compute]] — EC2 instances that use EBS and EFS
- [[Azure Storage]] — Compare with Microsoft Azure's storage services
- [[AWS IAM]] — Controlling access to S3 buckets and EBS volumes
