# Azure Databases

> **Managed database services** — Focus on your application logic while Azure handles
> backups, patching, high availability, scaling, and disaster recovery.

---

## Table of Contents

- [[#Why Managed Databases?]]
- [[#Azure SQL Database]]
- [[#Azure Cosmos DB]]
- [[#Azure Cache for Redis]]
- [[#Azure Database for MySQL / PostgreSQL]]
- [[#Database Comparison Table]]
- [[#Choosing the Right Database]]

---

## Why Managed Databases?

When you run a database yourself (on a VM or on-premise), **you** are responsible for:

- Installing and configuring the database engine
- Applying security patches and version upgrades
- Setting up backups and testing restores
- Configuring high availability (replicas, failover)
- Monitoring performance and tuning queries
- Scaling up/down when load changes
- Managing storage, IOPS, and networking

> **Managed databases remove all of that.**
> Azure takes care of the infrastructure so you can focus on your app.

### What Azure Manages for You

| Concern               | Self-hosted (VM)        | Managed (Azure)                |
|------------------------|-------------------------|--------------------------------|
| OS Patching            | You                     | Azure                          |
| DB Engine Updates      | You                     | Azure (with maintenance window)|
| Backups                | You (scripts, cron)     | Automatic, configurable        |
| High Availability      | You (replication setup) | Built-in                       |
| Scaling                | Manual (resize VM)      | One click / auto-scale         |
| Security               | You (firewall, TLS)     | Built-in + configurable        |
| Monitoring             | You (install tools)     | Built-in (Azure Monitor)       |

### Key Takeaway

> If your business is not "running databases", use a managed service.
> Save your engineering time for features that matter to your users.

---

## Azure SQL Database

**Azure SQL Database** is a fully managed relational database service built on
**Microsoft SQL Server**. It gives you the power of SQL Server without the
overhead of managing the engine, OS, or hardware.

### Deployment Options

Azure SQL offers **three deployment options** — pick based on your needs:

#### 1. Single Database

- **One isolated database** with its own dedicated resources.
- Billed per database (DTU or vCore model).
- Best for: New cloud-native apps, most typical workloads.
- Think of it as one database = one billing unit.

#### 2. Elastic Pool

- **A pool of shared resources** for multiple databases.
- Databases in the pool share CPU, memory, and storage.
- Best for: SaaS apps where many tenant databases have unpredictable usage.
- Cost-efficient when databases have varying, staggered peak usage.

> **Example**: You have 20 tenant databases. Each needs up to 100 DTUs at peak,
> but they peak at different times. Instead of paying 20 × 100 = 2000 DTUs,
> you buy a pool of 400 DTUs and share them.

#### 3. Managed Instance

- **Full SQL Server compatibility** in the cloud.
- Supports features not available in Single DB: SQL Agent, cross-database queries,
  CLR, Service Broker, linked servers.
- Best for: **Migrating on-premise SQL Server** to Azure with minimal changes.
- Deployed inside your own VNet for network isolation.

### Service Tiers

Azure SQL uses two pricing models:

#### DTU-Based (simpler)

| Tier      | DTUs    | Storage   | Best For              |
|-----------|---------|----------|-----------------------|
| Basic     | 5       | 2 GB     | Dev/test, light usage |
| Standard  | 10-3000 | 250 GB   | Most production apps  |
| Premium   | 125-4000| 1 TB     | High IO, in-memory    |

> **DTU (Database Transaction Unit)** = a blended measure of CPU, memory, and IO.
> You don't choose individual specs — you pick a DTU level.

#### vCore-Based (more control)

| Tier              | vCores | Features                           |
|-------------------|--------|------------------------------------|
| General Purpose   | 2-80   | Standard workloads, zone redundant |
| Business Critical | 2-128  | High IO, built-in read replica     |
| Hyperscale        | 2-128  | Rapid scale-out, up to 100 TB     |

> **vCore model** lets you independently choose compute and storage.
> Best when you need fine-grained control or already have SQL Server licenses
> (use **Azure Hybrid Benefit** to save up to 55%).

### Hands-on: Create an Azure SQL Database

> Prerequisites: Azure CLI installed, logged in (`az login`), resource group created.

```bash
# Step 1: Create a logical SQL server
# This is NOT a VM — it's a logical endpoint that hosts your databases
az sql server create \
  --name myserver123 \
  --resource-group myResourceGroup \
  --location centralindia \
  --admin-user sqladmin \
  --admin-password 'MyP@ssw0rd123!'
```

```bash
# Step 2: Configure firewall — allow YOUR IP address
# By default, NO external access is allowed (secure by default)
az sql server firewall-rule create \
  --resource-group myResourceGroup \
  --server myserver123 \
  --name AllowMyIP \
  --start-ip-address <your-ip> \
  --end-ip-address <your-ip>
```

```bash
# Step 3: Allow Azure services to connect
# This lets other Azure services (App Service, Functions) reach your DB
az sql server firewall-rule create \
  --resource-group myResourceGroup \
  --server myserver123 \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0
```

```bash
# Step 4: Create the database
# S0 = Standard tier, 10 DTUs — good for learning/dev
az sql db create \
  --resource-group myResourceGroup \
  --server myserver123 \
  --name myDatabase \
  --service-objective S0
```

```bash
# Step 5: Get the connection string (for your application)
az sql db show-connection-string \
  --server myserver123 \
  --name myDatabase \
  --client sqlcmd
```

```bash
# Step 6: Connect to the database
# From Azure Cloud Shell or a machine with sqlcmd installed:
# sqlcmd -S myserver123.database.windows.net -d myDatabase -U sqladmin -P 'MyP@ssw0rd123!'

# Once connected, try:
# SELECT @@VERSION;
# CREATE TABLE Users (Id INT PRIMARY KEY, Name NVARCHAR(50));
# INSERT INTO Users VALUES (1, 'Alice');
# SELECT * FROM Users;
```

```bash
# Cleanup when done
az sql db delete --resource-group myResourceGroup --server myserver123 --name myDatabase --yes
az sql server delete --resource-group myResourceGroup --name myserver123 --yes
```

### Key Azure SQL Features

| Feature                | Description                                                |
|------------------------|------------------------------------------------------------|
| **Auto-tuning**        | Azure automatically identifies and applies index/query improvements |
| **Geo-replication**    | Replicate your database to another Azure region for disaster recovery |
| **Point-in-time restore** | Restore to any point in the last 7-35 days (depending on tier) |
| **Long-term retention** | Keep backups for up to 10 years (compliance, auditing) |
| **Transparent Data Encryption (TDE)** | Data encrypted at rest by default |
| **Advanced Threat Protection** | Detects anomalous activities (SQL injection, brute force) |
| **Elastic Query**      | Query across multiple databases                            |
| **Read Replicas**      | Offload read traffic to replicas (Business Critical tier)  |

> **Connection string format** for applications:
> `Server=tcp:myserver123.database.windows.net,1433;Database=myDatabase;User ID=sqladmin;Password=...;Encrypt=true;`

---

## Azure Cosmos DB

**Azure Cosmos DB** is a **globally distributed, multi-model NoSQL database**
designed for planet-scale applications.

> Think of it as: "I need my data everywhere, fast, and I don't want to manage
> replication myself."

### APIs Supported

Cosmos DB is unique — it supports **multiple database APIs** on the same engine:

| API             | Data Model   | Use Case                          | Compatible With   |
|-----------------|-------------|-----------------------------------|-------------------|
| **NoSQL**       | Document/JSON| Default. Best for new apps        | Native Cosmos SDK |
| **MongoDB**     | Document/BSON| Migrate existing MongoDB apps     | MongoDB drivers   |
| **Cassandra**   | Wide-column  | Migrate Cassandra workloads       | Cassandra drivers |
| **Gremlin**     | Graph        | Social networks, recommendations  | Apache TinkerPop  |
| **Table**       | Key-value    | Migrate from Azure Table Storage  | Table Storage SDK |

> **Recommendation for new projects**: Use the **NoSQL API** (also called the
> Core/SQL API). It's the most feature-rich and best optimized.

### Key Concepts — The Cosmos DB Hierarchy

```
Cosmos DB Account
  └── Database
        └── Container (≈ table / collection)
              └── Items (≈ rows / documents)
```

- **Account**: Top-level resource. You choose the API when creating it.
- **Database**: Logical namespace. Can share throughput across containers.
- **Container**: Where your data lives. Requires a **partition key**.
- **Items**: Individual JSON documents (NoSQL API) or rows.

### Partition Key — Critical Choice!

> The partition key determines **how your data is distributed** across physical
> partitions. A bad partition key = hot partitions = poor performance.

**Good partition keys**:
- High cardinality (many unique values): `userId`, `orderId`, `tenantId`

**Bad partition keys**:
- Low cardinality: `country` (only ~200 values), `status` (only a few values)
- This causes hot partitions where one partition gets all the load.

### Consistency Levels

Cosmos DB offers **5 consistency levels** — a spectrum from strongest to most relaxed:

| Level               | Guarantee                                      | Latency | Cost   |
|---------------------|-------------------------------------------------|---------|--------|
| **Strong**          | Reads always see the latest write               | Highest | Highest|
| **Bounded Staleness** | Reads lag behind writes by at most K versions or T time | High    | High   |
| **Session**         | Within a session, reads see your own writes     | Medium  | Medium |
| **Consistent Prefix** | Reads never see out-of-order writes            | Low     | Low    |
| **Eventual**        | Reads may see older data, but eventually catch up| Lowest  | Lowest |

> **Default**: Session consistency. This is the sweet spot for most applications.
> It guarantees "read your own writes" which is what users expect.

### Request Units (RU/s) — How Cosmos DB Pricing Works

> **1 RU = the cost of reading a single 1KB item by its ID and partition key.**

Everything in Cosmos DB is measured in Request Units:

| Operation                          | Approximate Cost |
|-----------------------------------|-----------------|
| Read 1KB item by ID               | 1 RU            |
| Write 1KB item                    | ~5 RUs          |
| Query returning 1KB of results    | ~3 RUs          |
| Cross-partition query              | Higher RUs      |

- You provision throughput in RU/s (e.g., 400 RU/s minimum).
- If you exceed your RU/s, requests get **throttled** (HTTP 429).
- You can use **autoscale** to automatically scale between min and max RU/s.
- **Serverless mode**: Pay per RU consumed (no provisioning). Good for dev/test.

### Hands-on: Create a Cosmos DB Account

```bash
# Step 1: Create a Cosmos DB account (NoSQL API)
# This takes 5-10 minutes — Cosmos DB sets up global infrastructure
az cosmosdb create \
  --name mycosmosdb123 \
  --resource-group myResourceGroup \
  --default-consistency-level Session \
  --locations regionName=centralindia
```

```bash
# Step 2: Create a database
az cosmosdb sql database create \
  --account-name mycosmosdb123 \
  --resource-group myResourceGroup \
  --name myDB
```

```bash
# Step 3: Create a container with a partition key
# /id is fine for learning; in production, choose based on access patterns
az cosmosdb sql container create \
  --account-name mycosmosdb123 \
  --resource-group myResourceGroup \
  --database-name myDB \
  --name myContainer \
  --partition-key-path /id \
  --throughput 400
```

```bash
# Step 4: Get connection keys (you'll need these for your app)
az cosmosdb keys list \
  --name mycosmosdb123 \
  --resource-group myResourceGroup
```

```bash
# Step 5: Get the connection endpoint
az cosmosdb show \
  --name mycosmosdb123 \
  --resource-group myResourceGroup \
  --query documentEndpoint
```

```bash
# Cleanup
az cosmosdb delete --name mycosmosdb123 --resource-group myResourceGroup --yes
```

### When to Use Cosmos DB

✅ **Use Cosmos DB when you need**:
- Global distribution (replicate data across regions with a click)
- Single-digit millisecond latency at any scale
- Multi-model support (document, graph, key-value, column-family)
- Massive write throughput (IoT, gaming, e-commerce)
- 99.999% availability SLA (with multi-region writes)

❌ **Don't use Cosmos DB when**:
- You need complex joins and transactions (use Azure SQL instead)
- Your data fits a strict relational model
- Budget is very tight (Cosmos DB can be expensive at scale)

### Cosmos DB vs AWS DynamoDB

| Feature              | Cosmos DB                      | DynamoDB                     |
|----------------------|--------------------------------|------------------------------|
| **Data Models**      | Document, Graph, Key-Value, Column, Table | Key-Value, Document    |
| **APIs**             | NoSQL, MongoDB, Cassandra, Gremlin, Table | DynamoDB API only      |
| **Consistency**      | 5 levels (Strong → Eventual)   | 2 levels (Strong, Eventual)  |
| **Global Distribution** | Multi-region writes built-in | Global Tables (eventually consistent) |
| **Pricing**          | RU/s based                     | RCU/WCU based                |
| **Query Language**   | SQL-like syntax                | PartiQL or API-based         |
| **Serverless**       | Yes                            | Yes (on-demand mode)         |
| **SLA**              | 99.999% (multi-region)         | 99.999% (global tables)      |

> Both are excellent NoSQL services. Cosmos DB wins on flexibility (5 APIs, 5
> consistency levels). DynamoDB wins on simplicity and AWS ecosystem integration.

---

## Azure Cache for Redis

**Azure Cache for Redis** is a fully managed in-memory data store based on
**Redis** — the world's most popular caching solution.

> **In-memory = fast.** Redis stores data in RAM, so reads/writes are measured
> in microseconds, not milliseconds.

### Tiers

| Tier       | Features                                         | Best For           |
|------------|--------------------------------------------------|--------------------|
| **Basic**  | Single node, no SLA, no replication              | Dev/test only      |
| **Standard** | Two nodes (primary + replica), SLA, failover   | Production caching |
| **Premium** | Clustering, persistence, VNet, geo-replication  | Enterprise, HA     |
| **Enterprise** | Redis Enterprise modules (RediSearch, etc.)  | Advanced use cases |

### Common Use Cases

1. **Application caching** — Cache database query results, API responses.
   Reduce load on your database by 10-100x.
2. **Session store** — Store user sessions in Redis instead of the app server.
   Enables stateless apps (important for scaling).
3. **Real-time analytics** — Leaderboards, counters, rate limiting.
4. **Message broker** — Pub/Sub messaging between microservices.
5. **Queue** — Use Redis Lists as a lightweight job queue.

### Hands-on: Create Azure Cache for Redis

```bash
# Step 1: Create a Redis cache
# Basic tier, C0 size (250 MB) — good for learning
az redis create \
  --name myredis123 \
  --resource-group myResourceGroup \
  --location centralindia \
  --sku Basic \
  --vm-size c0
```

> ⚠️ Redis creation takes 15-20 minutes. Be patient!

```bash
# Step 2: Get the hostname
az redis show \
  --name myredis123 \
  --resource-group myResourceGroup \
  --query hostName
# Output: "myredis123.redis.cache.windows.net"
```

```bash
# Step 3: Get the access keys
az redis list-keys \
  --name myredis123 \
  --resource-group myResourceGroup
# Output includes primaryKey and secondaryKey
```

```bash
# Connection string format for your app:
# myredis123.redis.cache.windows.net:6380,password=<primaryKey>,ssl=True,abortConnect=False

# Cleanup
az redis delete --name myredis123 --resource-group myResourceGroup --yes
```

### Redis Best Practices

- **Set expiration** on cached items (TTL). Don't let the cache grow unbounded.
- **Use connection pooling** in your app. Don't open a new connection per request.
- **Avoid large keys** — keep values under 100 KB for best performance.
- **Use the Standard tier or above** for production (Basic has no SLA).

---

## Azure Database for MySQL / PostgreSQL

Azure offers **fully managed** versions of the two most popular open-source
databases: **MySQL** and **PostgreSQL**.

> These are ideal when your team already knows MySQL/PostgreSQL and you don't
> want to change your database engine.

### Deployment Options

Both MySQL and PostgreSQL offer a **Flexible Server** deployment:

- **Flexible Server** (recommended):
  - Zone-redundant HA
  - Configurable maintenance windows
  - Burstable, General Purpose, and Memory Optimized tiers
  - Start/Stop capability (save costs on dev/test)
  - Custom server parameters

### Hands-on: Create a MySQL Flexible Server

```bash
# Create a MySQL flexible server
az mysql flexible-server create \
  --resource-group myResourceGroup \
  --name mymysql123 \
  --admin-user mysqladmin \
  --admin-password 'MyP@ssw0rd123!' \
  --sku-name Standard_B1ms \
  --tier Burstable
```

```bash
# Connect to MySQL
az mysql flexible-server connect \
  --name mymysql123 \
  --admin-user mysqladmin \
  --admin-password 'MyP@ssw0rd123!' \
  --interactive
```

```bash
# Create a PostgreSQL flexible server
az postgres flexible-server create \
  --resource-group myResourceGroup \
  --name mypostgres123 \
  --admin-user pgadmin \
  --admin-password 'MyP@ssw0rd123!' \
  --sku-name Standard_B1ms \
  --tier Burstable
```

```bash
# Cleanup
az mysql flexible-server delete --resource-group myResourceGroup --name mymysql123 --yes
az postgres flexible-server delete --resource-group myResourceGroup --name mypostgres123 --yes
```

### When to Choose MySQL vs PostgreSQL

| Feature            | MySQL                          | PostgreSQL                     |
|--------------------|--------------------------------|--------------------------------|
| **Strengths**      | Simple, fast reads, widespread | Advanced features, JSONB, CTE  |
| **Best For**       | Web apps, CMS, e-commerce      | Analytics, GIS, complex queries|
| **JSON Support**   | Basic                          | Advanced (JSONB, indexing)     |
| **Replication**    | Read replicas                  | Read replicas, logical replication |
| **Community**      | Very large                     | Growing rapidly                |

---

## Database Comparison Table

| Feature           | Azure SQL Database          | Cosmos DB                    | Azure Cache for Redis | MySQL / PostgreSQL         |
|-------------------|-----------------------------|------------------------------|-----------------------|----------------------------|
| **Type**          | Relational (SQL Server)     | NoSQL (multi-model)          | In-memory cache       | Relational (open-source)   |
| **Data Model**    | Tables, rows, joins         | Documents, graphs, key-value | Key-value, hashes     | Tables, rows, joins        |
| **Query Language**| T-SQL                       | SQL-like (NoSQL API)         | Redis commands        | MySQL SQL / PostgreSQL SQL |
| **Scaling**       | Vertical + read replicas    | Horizontal (partitioning)    | Clustering (Premium)  | Vertical + read replicas   |
| **Consistency**   | ACID transactions           | 5 configurable levels        | Single-node consistent| ACID transactions          |
| **Global Distribution** | Geo-replication       | Built-in multi-region writes | Geo-replication (Premium) | Read replicas (same region) |
| **Pricing Model** | DTU or vCore                | Request Units (RU/s)         | Cache size + tier     | vCore + storage            |
| **Best For**      | Structured data, transactions| Global, flexible schema      | Caching, sessions     | Open-source compatibility  |
| **SLA**           | 99.99%                      | 99.999% (multi-region)       | 99.9% (Standard+)    | 99.99%                     |
| **Serverless**    | Yes                         | Yes                          | No                    | No (but Burstable tier)    |

---

## Choosing the Right Database

Use this decision tree:

```
Start: What kind of data do you have?
│
├── Structured data with relationships (tables, foreign keys)?
│   ├── Need SQL Server compatibility? → Azure SQL Database
│   ├── Need open-source (MySQL)? → Azure Database for MySQL
│   └── Need advanced SQL (PostGIS, JSONB)? → Azure Database for PostgreSQL
│
├── Semi-structured / flexible schema (JSON, documents)?
│   ├── Need global distribution? → Cosmos DB
│   ├── Need graph queries? → Cosmos DB (Gremlin API)
│   └── Existing MongoDB app? → Cosmos DB (MongoDB API)
│
├── Need fast caching / session store?
│   └── Azure Cache for Redis
│
└── Not sure?
    └── Start with Azure SQL Database (most versatile for typical apps)
```

### Quick Decision Guide

| Scenario                                    | Recommended Service              |
|---------------------------------------------|----------------------------------|
| E-commerce product catalog                  | Azure SQL or Cosmos DB           |
| User session management                     | Azure Cache for Redis            |
| IoT telemetry data (millions of writes/sec) | Cosmos DB                        |
| Content management system                   | Azure Database for MySQL         |
| Social network (graph relationships)        | Cosmos DB (Gremlin API)          |
| Data warehouse / analytics                  | Azure Synapse Analytics          |
| Migrating on-prem SQL Server                | Azure SQL Managed Instance       |
| Migrating on-prem MongoDB                   | Cosmos DB (MongoDB API)          |
| Caching API responses                       | Azure Cache for Redis            |
| Real-time leaderboards                      | Azure Cache for Redis            |

---

## Summary

| Service                       | One-Line Summary                                     |
|-------------------------------|------------------------------------------------------|
| **Azure SQL Database**        | Managed SQL Server for relational workloads           |
| **Azure Cosmos DB**           | Global NoSQL database with multiple APIs              |
| **Azure Cache for Redis**     | In-memory cache for speed                             |
| **Azure Database for MySQL**  | Managed MySQL for open-source compatibility           |
| **Azure Database for PostgreSQL** | Managed PostgreSQL for advanced SQL needs         |

> 🎯 **Beginner tip**: Start with **Azure SQL Database** (Single Database, S0 tier).
> It's the most forgiving, has the most tutorials, and handles most typical
> application workloads. Add Redis for caching once you understand the basics.

---

## Related Notes

- [[Cloud Fundamentals]] — Core cloud concepts (IaaS, PaaS, SaaS)
- [[Azure Getting Started]] — Setting up your Azure account and CLI
- [[AWS Databases]] — Compare with AWS database services (RDS, DynamoDB, ElastiCache)
- [[Azure Compute]] — VMs, App Service, and other compute options
- [[Azure Networking]] — VNets, NSGs, and how databases connect to your apps
- [[Azure Billing]] — Understand costs before provisioning databases
