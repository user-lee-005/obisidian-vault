# AWS Databases

> **Why do we need managed databases?**
> In the old world, if you wanted a database, you had to:
> 1. Buy a server
> 2. Install the database software (MySQL, PostgreSQL, etc.)
> 3. Configure it properly
> 4. Set up backups manually
> 5. Patch and update the software
> 6. Handle replication and failover yourself
> 7. Scale it when your app grows
>
> **That's a LOT of work.**
> AWS gives you **managed database services** — they handle all of the above.
> You just create the database, connect to it, and focus on your application.

---

## Table of Contents

- [[#Why Managed Databases]]
- [[#Amazon RDS (Relational Database Service)]]
- [[#Amazon Aurora]]
- [[#RDS Hands-On]]
- [[#Multi-AZ vs Read Replicas]]
- [[#RDS Pricing]]
- [[#Amazon DynamoDB]]
- [[#DynamoDB Hands-On]]
- [[#DynamoDB Advanced Features]]
- [[#Amazon ElastiCache]]
- [[#ElastiCache Hands-On]]
- [[#Database Comparison Table]]
- [[#Choosing the Right Database]]

---

## Why Managed Databases

Think of it like renting an apartment vs building a house:

| **Self-Managed (on EC2)** | **AWS Managed (RDS, DynamoDB, etc.)** |
|---|---|
| You install the DB software | Already installed and configured |
| You configure backups | Automated daily backups |
| You handle OS patching | AWS patches the OS for you |
| You set up replication | One-click Multi-AZ replication |
| You monitor everything | Built-in monitoring via CloudWatch |
| You handle failover manually | Automatic failover in minutes |
| You scale manually | Easy vertical/horizontal scaling |

> [!tip] Rule of Thumb
> **Always use a managed database service** unless you have a very specific reason not to.
> Installing a database on an EC2 instance is almost never the right choice in AWS.

### What "Managed" Actually Means

When AWS says "managed," they take care of:

- **Provisioning** — Hardware is allocated automatically
- **Patching** — OS and database engine updates applied during maintenance windows
- **Backups** — Automated daily snapshots, point-in-time recovery
- **Monitoring** — CloudWatch metrics out of the box
- **High Availability** — Multi-AZ deployments with automatic failover
- **Security** — Encryption at rest and in transit, IAM integration
- **Scaling** — Resize instances, add read replicas, or auto-scale storage

---

## Amazon RDS (Relational Database Service)

**What is it?** A managed service for **relational (SQL) databases**. You choose your database engine, AWS handles the rest.

### What is a Relational Database?

If you're completely new: a relational database stores data in **tables** with **rows and columns** — like a spreadsheet. Tables can be linked (related) to each other.

Example:
```
Users Table:           Orders Table:
+----+-------+        +----+---------+--------+
| id | name  |        | id | user_id | amount |
+----+-------+        +----+---------+--------+
| 1  | John  |        | 1  | 1       | 50.00  |
| 2  | Jane  |        | 2  | 1       | 30.00  |
+----+-------+        | 3  | 2       | 75.00  |
                       +----+---------+--------+
```

You query them using **SQL** (Structured Query Language):
```sql
SELECT u.name, o.amount
FROM Users u
JOIN Orders o ON u.id = o.user_id
WHERE u.name = 'John';
```

### Supported Database Engines

RDS supports **6 database engines**:

| Engine | Description | Use Case |
|---|---|---|
| **MySQL** | Most popular open-source DB | General-purpose web apps |
| **PostgreSQL** | Advanced open-source DB | Complex queries, GIS data |
| **MariaDB** | MySQL fork, community-driven | Drop-in MySQL replacement |
| **Oracle** | Enterprise commercial DB | Legacy enterprise apps |
| **SQL Server** | Microsoft's DB | .NET / Windows applications |
| **Aurora** | AWS-built, MySQL/PostgreSQL compatible | High-performance, cloud-native |

> [!note] Which engine should a beginner pick?
> Start with **MySQL** or **PostgreSQL**. They're free, widely used, and have the most learning resources. Aurora is amazing but costs more.

### Key RDS Features

1. **Automated Backups**
   - Daily snapshots of your entire database
   - Transaction logs backed up every 5 minutes
   - **Point-in-time recovery**: Restore to any second in the last 35 days
   - Stored in S3 (you don't see them, but they're there)

2. **Multi-AZ Deployment** (High Availability)
   - AWS creates a **standby replica** in a different Availability Zone
   - If the primary fails, AWS automatically switches to the standby
   - You don't change your connection string — DNS handles the switch
   - Takes about 60-120 seconds for failover

3. **Read Replicas** (Read Scaling)
   - Create up to **15 read replicas** (5 for most engines, 15 for Aurora)
   - Replicas handle read traffic → your primary handles writes
   - Can be in different regions (cross-region read replicas)
   - Replication is **asynchronous** (slight delay)

4. **Encryption**
   - At rest: AES-256 encryption using AWS KMS
   - In transit: SSL/TLS connections
   - Must be enabled at creation time (can't encrypt existing unencrypted DB)

5. **Monitoring**
   - CloudWatch metrics: CPU, memory, storage, connections, IOPS
   - Enhanced Monitoring: OS-level metrics every 1-60 seconds
   - Performance Insights: SQL-level performance analysis

---

## Amazon Aurora

Aurora deserves special attention — it's AWS's **crown jewel** database.

### What Makes Aurora Special?

- **Built by AWS** from the ground up for the cloud
- **Compatible** with MySQL and PostgreSQL (your app doesn't need to change)
- **5x faster** than standard MySQL, **3x faster** than standard PostgreSQL
- **Storage auto-scales** from 10GB up to 128TB automatically
- **6 copies** of your data across 3 Availability Zones
- **Self-healing** storage — continuously scans and repairs errors

### Aurora vs Standard RDS

| Feature | Standard RDS | Aurora |
|---|---|---|
| Storage | You allocate (max 64TB) | Auto-scales to 128TB |
| Replication | Async | Synchronous within cluster |
| Read Replicas | Up to 5 | Up to 15 |
| Failover | 60-120 seconds | ~30 seconds |
| Data Copies | 2 (Multi-AZ) | 6 across 3 AZs |
| Cost | Lower | ~20% more than RDS |

### Aurora Serverless

- **No provisioning** — Aurora scales compute up/down automatically
- Pay per second of compute used
- Great for unpredictable workloads
- Can scale to zero when not in use (Aurora Serverless v2)

> [!tip] When to use Aurora
> If you're building a production app on AWS and need a relational database, **Aurora is usually the best choice**. For learning/dev, standard RDS MySQL is cheaper.

---

## RDS Hands-On

### Create an RDS MySQL Instance (CLI)

```bash
# Step 1: Create a DB subnet group (tells RDS which subnets to use)
aws rds create-db-subnet-group \
  --db-subnet-group-name my-db-subnet-group \
  --db-subnet-group-description "Subnets for my RDS instance" \
  --subnet-ids subnet-xxx subnet-yyy

# Step 2: Create the RDS instance
aws rds create-db-instance \
  --db-instance-identifier my-database \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --engine-version 8.0 \
  --master-username admin \
  --master-user-password MyP@ssw0rd123 \
  --allocated-storage 20 \
  --no-publicly-accessible \
  --vpc-security-group-ids sg-xxx \
  --db-subnet-group-name my-db-subnet-group \
  --backup-retention-period 7 \
  --multi-az \
  --storage-encrypted

# Step 3: Wait for it to be available (takes 5-10 minutes)
aws rds wait db-instance-available --db-instance-identifier my-database

# Step 4: Check the status
aws rds describe-db-instances \
  --db-instance-identifier my-database \
  --query 'DBInstances[0].DBInstanceStatus'

# Step 5: Get the endpoint (the address you connect to)
aws rds describe-db-instances \
  --db-instance-identifier my-database \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text

# Step 6: Connect from an EC2 instance in the same VPC
mysql -h my-database.xxxx.ap-south-1.rds.amazonaws.com -u admin -p
```

> [!warning] Important
> - `--no-publicly-accessible` means only resources inside the VPC can connect.
> - The EC2 instance's security group must allow outbound traffic on port 3306.
> - The RDS security group must allow inbound traffic on port 3306 from the EC2 security group.
> - See [[AWS Networking]] for how VPCs, subnets, and security groups work.

### Create an RDS Instance (Console Steps)

1. Go to **RDS** → **Create database**
2. Choose **Standard create**
3. Engine: **MySQL**
4. Template: **Free tier** (for learning)
5. DB instance identifier: `my-database`
6. Master username: `admin`
7. Master password: set a strong password
8. Instance class: `db.t3.micro`
9. Storage: 20 GiB, General Purpose SSD (gp3)
10. Connectivity: Choose your VPC, subnet group, **No** public access
11. Click **Create database**
12. Wait 5-10 minutes for status = "Available"

### Manage Your RDS Instance

```bash
# Take a manual snapshot (backup)
aws rds create-db-snapshot \
  --db-instance-identifier my-database \
  --db-snapshot-identifier my-backup-2024

# Create a read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier my-database-replica \
  --source-db-instance-identifier my-database

# Modify instance (e.g., change instance class)
aws rds modify-db-instance \
  --db-instance-identifier my-database \
  --db-instance-class db.t3.small \
  --apply-immediately

# Delete the instance (when done learning — IMPORTANT to avoid charges!)
aws rds delete-db-instance \
  --db-instance-identifier my-database \
  --skip-final-snapshot
```

> [!danger] Clean Up!
> Always delete RDS instances when you're done learning. Even a `db.t3.micro` costs ~$12/month if left running.

---

## Multi-AZ vs Read Replicas

These are two different features that beginners often confuse:

| Feature | Multi-AZ | Read Replicas |
|---|---|---|
| **Purpose** | High Availability (HA) | Read Scaling |
| **Problem it solves** | "What if my DB server crashes?" | "My DB is slow because of too many reads" |
| **How it works** | Standby copy in another AZ | Additional read-only copies |
| **Replication** | Synchronous (real-time) | Asynchronous (slight delay) |
| **Failover** | Automatic (60-120s) | Manual promotion needed |
| **Can you read from it?** | No (standby only) | Yes (read traffic) |
| **Connection string** | Same (DNS auto-switches) | Different endpoint per replica |
| **Number** | 1 standby | Up to 5 (15 for Aurora) |
| **Cost** | 2x instance cost | Additional instance per replica |

> [!example] Analogy
> - **Multi-AZ** = Having a backup goalkeeper on the bench. If the starter gets injured, the backup jumps in immediately.
> - **Read Replicas** = Having multiple cashiers at a store. All serve customers (reads), but only one handles returns (writes).

---

## RDS Pricing

RDS charges for:

| Component | What You Pay For | Example |
|---|---|---|
| **Instance hours** | Time the DB is running | db.t3.micro: ~$0.017/hr |
| **Storage** | GB-month of allocated storage | gp3: ~$0.115/GB-month |
| **I/O requests** | Only for Aurora I/O-Optimized | Per million I/O requests |
| **Backup storage** | Storage beyond your DB size | First backup free |
| **Data transfer** | Data sent out of AWS | In = free, Out = ~$0.09/GB |
| **Multi-AZ** | Doubles the instance cost | Extra for standby |

> [!tip] Free Tier
> 750 hours/month of `db.t2.micro` (single-AZ) for 12 months.
> That's enough to run ONE database 24/7. See [[AWS Billing]] for details.

---

## Amazon DynamoDB

### What is DynamoDB?

DynamoDB is a **fully managed NoSQL database** — it stores data WITHOUT the rigid table structures of SQL databases.

### What is NoSQL?

Unlike relational databases with fixed schemas (every row has the same columns), NoSQL is flexible:

```
// SQL (relational) — every row MUST have the same columns
| id | name | email           |
|----|------|-----------------|
| 1  | John | john@email.com  |
| 2  | Jane | jane@email.com  |

// NoSQL (DynamoDB) — each item can have different attributes
{
  "UserId": "user1",
  "Name": "John",
  "Email": "john@email.com",
  "Phone": "555-1234"     ← John has a phone
}
{
  "UserId": "user2",
  "Name": "Jane",
  "Email": "jane@email.com",
  "Address": "123 Main St" ← Jane has an address instead
}
```

### DynamoDB Core Concepts

| Concept | SQL Equivalent | Description |
|---|---|---|
| **Table** | Table | Container for data |
| **Item** | Row | A single data record |
| **Attribute** | Column | A data field in an item |
| **Partition Key** | Primary Key | Unique identifier (required) |
| **Sort Key** | Composite Key part | Optional second key for ordering |

### Primary Key Design

This is the **MOST IMPORTANT** decision in DynamoDB:

**Option 1: Partition Key Only (Simple)**
```
Table: Users
Partition Key: UserId

Each UserId must be unique.
```

**Option 2: Partition Key + Sort Key (Composite)**
```
Table: Orders
Partition Key: CustomerId
Sort Key: OrderDate

Same CustomerId can have many orders, sorted by date.
You can query: "Get all orders for customer X between Jan and Mar"
```

> [!warning] DynamoDB is NOT a replacement for RDS
> DynamoDB excels at specific patterns (key-value lookups, high-scale).
> If you need complex JOINs, aggregations, or ad-hoc queries → use RDS.

### Capacity Modes

| Mode | How It Works | Best For | Cost |
|---|---|---|---|
| **On-Demand** | Pay per read/write request | Unpredictable traffic | Higher per-request |
| **Provisioned** | You set Read/Write capacity | Predictable traffic | Lower, but you guess |

- **RCU** = Read Capacity Unit = 1 strongly consistent read/sec for items up to 4KB
- **WCU** = Write Capacity Unit = 1 write/sec for items up to 1KB

> [!tip] For learning, always use **On-Demand** mode. No capacity planning needed.

---

## DynamoDB Hands-On

### Create a Table and Work with Items (CLI)

```bash
# Step 1: Create a table with a simple partition key
aws dynamodb create-table \
  --table-name Users \
  --attribute-definitions \
    AttributeName=UserId,AttributeType=S \
  --key-schema \
    AttributeName=UserId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

# Wait for table to be active
aws dynamodb wait table-exists --table-name Users

# Step 2: Put an item (add a user)
aws dynamodb put-item \
  --table-name Users \
  --item '{
    "UserId": {"S": "user1"},
    "Name": {"S": "John"},
    "Email": {"S": "john@example.com"},
    "Age": {"N": "30"}
  }'

# Step 3: Put another item (notice different attributes — that's OK!)
aws dynamodb put-item \
  --table-name Users \
  --item '{
    "UserId": {"S": "user2"},
    "Name": {"S": "Jane"},
    "Email": {"S": "jane@example.com"},
    "Phone": {"S": "555-1234"},
    "Premium": {"BOOL": true}
  }'

# Step 4: Get a specific item by key
aws dynamodb get-item \
  --table-name Users \
  --key '{"UserId": {"S": "user1"}}'

# Step 5: Scan all items (reads entire table — expensive for large tables!)
aws dynamodb scan --table-name Users

# Step 6: Update an item
aws dynamodb update-item \
  --table-name Users \
  --key '{"UserId": {"S": "user1"}}' \
  --update-expression "SET Age = :newage" \
  --expression-attribute-values '{":newage": {"N": "31"}}'

# Step 7: Delete an item
aws dynamodb delete-item \
  --table-name Users \
  --key '{"UserId": {"S": "user1"}}'

# Step 8: Delete the table (clean up!)
aws dynamodb delete-table --table-name Users
```

> [!note] Data Types in DynamoDB CLI
> - `S` = String
> - `N` = Number (always passed as string in JSON)
> - `BOOL` = Boolean
> - `L` = List
> - `M` = Map (nested object)
> - `SS` = String Set
> - `NS` = Number Set

### Create a Table with Composite Key

```bash
# Orders table: CustomerId (partition) + OrderDate (sort)
aws dynamodb create-table \
  --table-name Orders \
  --attribute-definitions \
    AttributeName=CustomerId,AttributeType=S \
    AttributeName=OrderDate,AttributeType=S \
  --key-schema \
    AttributeName=CustomerId,KeyType=HASH \
    AttributeName=OrderDate,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST

# Query all orders for a customer (efficient — uses partition key)
aws dynamodb query \
  --table-name Orders \
  --key-condition-expression "CustomerId = :cid" \
  --expression-attribute-values '{":cid": {"S": "cust1"}}'

# Query orders for a customer within a date range
aws dynamodb query \
  --table-name Orders \
  --key-condition-expression "CustomerId = :cid AND OrderDate BETWEEN :start AND :end" \
  --expression-attribute-values '{
    ":cid": {"S": "cust1"},
    ":start": {"S": "2024-01-01"},
    ":end": {"S": "2024-03-31"}
  }'
```

---

## DynamoDB Advanced Features

### DynamoDB Streams

- Captures **every change** (insert, update, delete) to your table in real-time
- Changes are stored for **24 hours**
- Commonly used to **trigger AWS Lambda** functions on data changes
- Use cases: Send email when new user signs up, sync data to another system, build audit logs

### Global Tables

- **Multi-region replication** — your table is automatically replicated across AWS regions
- Provides **low-latency reads** from the nearest region
- **Active-active** — you can read AND write from any region
- Great for globally distributed applications

### DynamoDB Accelerator (DAX)

- **In-memory cache** specifically for DynamoDB
- Reduces read latency from **milliseconds to microseconds**
- Drop-in replacement — just change your endpoint
- Use when: Read-heavy workloads, same items read repeatedly

### When to Use DynamoDB

✅ **Good for:**
- High-scale key-value lookups (millions of requests/sec)
- Session stores, shopping carts, user profiles
- IoT data, gaming leaderboards, ad tech
- Serverless applications with Lambda
- When you know your access patterns ahead of time

❌ **Not good for:**
- Complex JOINs across multiple tables
- Ad-hoc analytics queries
- Small datasets where RDS is simpler
- When you don't know how you'll query the data

---

## Amazon ElastiCache

### What is ElastiCache?

ElastiCache is a **managed in-memory data store**. It keeps data in **RAM** instead of disk, making it **blazing fast** (sub-millisecond response times).

### Why Do You Need a Cache?

```
Without cache:
User Request → App Server → Database (slow disk read) → Response (100ms)

With cache:
User Request → App Server → Cache (fast RAM read) → Response (1ms)
                              ↓ (cache miss)
                           Database → Store in cache → Response
```

### Redis vs Memcached

| Feature | Redis | Memcached |
|---|---|---|
| **Data structures** | Strings, lists, sets, hashes, sorted sets | Strings only |
| **Persistence** | Yes (snapshots, AOF) | No |
| **Replication** | Yes (read replicas) | No |
| **Clustering** | Yes (up to 500 nodes) | Yes (simpler) |
| **Pub/Sub** | Yes | No |
| **Lua scripting** | Yes | No |
| **Multi-threaded** | No (single-threaded) | Yes |
| **Use when** | You need advanced features | Simple key-value caching |

> [!tip] **Pick Redis** in most cases. It's more feature-rich and versatile.

### Use Cases

1. **Session Store** — Store user session data for web apps (instead of sticky sessions)
2. **Database Query Cache** — Cache frequent DB queries to reduce load on RDS
3. **Leaderboards** — Redis sorted sets are perfect for real-time rankings
4. **Real-time Analytics** — Count page views, active users, etc.
5. **Rate Limiting** — Track API call counts per user
6. **Message Queues** — Redis pub/sub for simple messaging

---

## ElastiCache Hands-On

### Create a Redis Cluster (CLI)

```bash
# Step 1: Create a cache subnet group
aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name my-cache-subnet \
  --cache-subnet-group-description "Subnets for ElastiCache" \
  --subnet-ids subnet-xxx subnet-yyy

# Step 2: Create a Redis cluster
aws elasticache create-cache-cluster \
  --cache-cluster-id my-redis \
  --engine redis \
  --cache-node-type cache.t3.micro \
  --num-cache-nodes 1 \
  --cache-subnet-group-name my-cache-subnet \
  --security-group-ids sg-xxx

# Step 3: Check status
aws elasticache describe-cache-clusters \
  --cache-cluster-id my-redis \
  --query 'CacheClusters[0].CacheClusterStatus'

# Step 4: Get the endpoint
aws elasticache describe-cache-clusters \
  --cache-cluster-id my-redis \
  --show-cache-node-info \
  --query 'CacheClusters[0].CacheNodes[0].Endpoint'

# Step 5: Connect from EC2 (install redis-cli first)
redis-cli -h my-redis.xxxx.cache.amazonaws.com -p 6379

# Step 6: Basic Redis commands once connected
SET user:1:name "John"
GET user:1:name
EXPIRE user:1:name 3600   # TTL = 1 hour
TTL user:1:name            # Check remaining TTL
DEL user:1:name            # Delete

# Step 7: Clean up!
aws elasticache delete-cache-cluster --cache-cluster-id my-redis
```

> [!warning] ElastiCache is NOT accessible from the internet
> It runs inside your VPC only. You must connect from an EC2 instance or Lambda in the same VPC.
> See [[AWS Networking]] for VPC setup.

---

## Database Comparison Table

| Feature | RDS / Aurora | DynamoDB | ElastiCache |
|---|---|---|---|
| **Type** | Relational (SQL) | NoSQL (key-value/document) | In-memory cache |
| **Data Model** | Tables with fixed schema | Flexible schema (JSON-like) | Key-value pairs |
| **Query Language** | SQL | DynamoDB API / PartiQL | Redis/Memcached commands |
| **Scaling** | Vertical + Read Replicas | Horizontal (auto) | Vertical + clustering |
| **Max Storage** | 64TB (RDS) / 128TB (Aurora) | Unlimited | RAM-limited |
| **Latency** | Low milliseconds | Single-digit milliseconds | Sub-millisecond |
| **Transactions** | Full ACID | Limited ACID | Limited |
| **Use Case** | Complex queries, JOINs | High-scale lookups | Caching, sessions |
| **Pricing** | Instance + storage | Per request or capacity | Instance-based |
| **Free Tier** | 750 hrs db.t2.micro | 25GB + 25 WCU/RCU | None |
| **Managed** | Mostly (you choose engine) | Fully (serverless option) | Mostly |

---

## Choosing the Right Database

Use this decision tree:

```
Do you need a database?
├── Is your data structured with relationships (JOINs)?
│   ├── YES → Do you need high performance?
│   │   ├── YES → Amazon Aurora
│   │   └── NO → Amazon RDS (MySQL/PostgreSQL)
│   └── NO → Continue below
├── Is it key-value or document data?
│   ├── YES → Do you need massive scale?
│   │   ├── YES → DynamoDB
│   │   └── NO → Could use either RDS or DynamoDB
│   └── NO → Continue below
├── Do you need a caching layer?
│   └── YES → ElastiCache (Redis)
├── Do you need full-text search?
│   └── YES → Amazon OpenSearch Service
├── Do you need a graph database?
│   └── YES → Amazon Neptune
├── Do you need a time-series database?
│   └── YES → Amazon Timestream
├── Do you need a ledger (immutable records)?
│   └── YES → Amazon QLDB
└── Not sure → Start with RDS (MySQL). You can always migrate later.
```

### Quick Reference

| Requirement | Service |
|---|---|
| Structured data with JOINs | RDS or Aurora |
| High-throughput key-value | DynamoDB |
| Caching / session store | ElastiCache (Redis) |
| Full-text search | OpenSearch Service |
| Graph relationships | Neptune |
| Time-series data | Timestream |
| Immutable audit log | QLDB |
| Data warehouse (analytics) | Redshift |

> [!note] Other Database Services (Brief Mentions)
> - **Amazon Redshift** — Data warehouse for analytics. Column-oriented, SQL-based, petabyte-scale. Use for business intelligence. Not covered in detail here.
> - **Amazon OpenSearch** (formerly Elasticsearch) — Full-text search, log analytics. Use for search bars, log analysis.
> - **Amazon Neptune** — Graph database. Use for social networks, recommendation engines, fraud detection.
> - **Amazon DocumentDB** — MongoDB-compatible. Use if you're already using MongoDB.
> - **Amazon Keyspaces** — Cassandra-compatible. Use for wide-column workloads.

---

## Summary

| What You Learned | Key Takeaway |
|---|---|
| Managed databases | Let AWS handle the heavy lifting |
| RDS | Managed SQL — MySQL, PostgreSQL, Aurora |
| Aurora | AWS-built, 5x faster MySQL/PostgreSQL |
| Multi-AZ | High availability (auto-failover) |
| Read Replicas | Scale read traffic |
| DynamoDB | Managed NoSQL for high-scale key-value |
| ElastiCache | In-memory caching with Redis or Memcached |
| Choosing a DB | Match your data model to the right service |

---

## Cross-References

- [[Cloud Fundamentals]] — What is cloud computing and why use it
- [[AWS Getting Started]] — Setting up your AWS account and first steps
- [[Azure Databases]] — How Azure handles databases (for comparison)
- [[AWS Networking]] — VPC, subnets, and security groups (needed for DB connectivity)
- [[AWS Billing]] — Database pricing and free tier details
