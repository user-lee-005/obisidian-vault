# Sharding and Replication

> *"One database can only grow so tall. At some point, you have to grow wide."*

---

## Phase 1: The Fundamentals — Why Scale a Database?

### The Scaling Problem

A single database server has physical limits:

```
Single PostgreSQL server:
  CPU: 64 cores (maxed out at 10,000 queries/sec)
  RAM: 512 GB (dataset is 2 TB — doesn't fit in memory)
  Disk: 10 TB (growing 500 GB/month — running out)
  Connections: 500 max (1,000 microservice instances want to connect)
```

When your database hits these limits, you have two options:

### Vertical Scaling (Scale Up)

Buy a **bigger machine** — more CPU, RAM, disk.

```
Before: 16 cores, 64 GB RAM, 1 TB disk
After:  64 cores, 512 GB RAM, 10 TB disk
```

| Pros | Cons |
|---|---|
| Simple — no code changes | There's a hardware ceiling (you can't buy a 10,000-core machine) |
| No distributed complexity | Exponentially expensive at high end |
| ACID transactions still work normally | Still a single point of failure |

### Horizontal Scaling (Scale Out)

Distribute data across **multiple machines**.

```
Before: 1 server with 10 TB

After: 4 servers with 2.5 TB each
  Server 1: Shipments A-F
  Server 2: Shipments G-M
  Server 3: Shipments N-S
  Server 4: Shipments T-Z
```

| Pros | Cons |
|---|---|
| Theoretically unlimited scaling | Complex — distributed queries, joins across nodes |
| No single point of failure | Transactions across shards are hard |
| Cost-effective (commodity hardware) | Operational overhead (managing many nodes) |

**This chapter covers both strategies for horizontal scaling: Replication (copies) and Sharding (partitioning).**

---

## Phase 2: Replication — Making Copies

### What is Replication?

Replication is maintaining **identical copies** of data on multiple database servers. Each copy is called a **replica**.

```
              ┌─────────────────┐
  Writes ───→ │  Primary/Leader  │
              │  (full dataset)  │
              └───────┬─────────┘
                      │ replication
           ┌──────────┼──────────┐
           ↓          ↓          ↓
     ┌──────────┐ ┌──────────┐ ┌──────────┐
     │ Replica 1│ │ Replica 2│ │ Replica 3│
     │(full copy)│ │(full copy)│ │(full copy)│
     └──────────┘ └──────────┘ └──────────┘
        Reads ←       Reads ←       Reads ←
```

### Why Replicate?

| Reason | Explanation |
|---|---|
| **High Availability** | If primary dies, a replica takes over (failover) |
| **Read Scalability** | Distribute read traffic across replicas |
| **Geographic Distribution** | Replica in US serves US users with low latency; replica in Asia serves Asia |
| **Backup** | Replicas act as live backups |

---

### Replication Topology 1: Single-Leader (Primary-Replica)

**One primary** handles all writes. **Multiple replicas** receive copies and handle reads.

```
Client Write → Primary → replicates to → Replica 1
                                       → Replica 2
                                       → Replica 3

Client Read → Replica 1 (or 2 or 3)
```

**How writes are replicated:**

#### Synchronous Replication

Primary **waits** for replica(s) to confirm the write before acknowledging to the client.

```
Client → Write to Primary → Primary writes to disk
       → Primary sends to Replica 1 → Replica 1 writes to disk → ACK
       → Primary sends to Replica 2 → Replica 2 writes to disk → ACK
       → Primary acknowledges client ✅

Total latency = Primary write + slowest replica write
```

**Pros:** Strong consistency — replicas always have the latest data.
**Cons:** Higher write latency; if any replica is slow/down, writes are delayed.

#### Asynchronous Replication

Primary **immediately acknowledges** the write and replicates in the background.

```
Client → Write to Primary → Primary writes to disk → Acknowledges client ✅
       → (background) Primary sends to Replica 1
       → (background) Primary sends to Replica 2

Total latency = Primary write only (fast!)
```

**Pros:** Low write latency; replica failures don't block writes.
**Cons:** Replicas may be behind — reads from replicas return stale data (replication lag).

#### Semi-Synchronous Replication

Primary waits for **at least one replica** to confirm, then acknowledges. Other replicas are updated asynchronously.

```
Client → Write to Primary → Primary writes to disk
       → Sends to all replicas
       → Replica 1 ACKs first → Primary acknowledges client ✅
       → Replica 2 ACKs later (async)
```

**This is the most common production setup** — balances durability and performance.

---

### Replication Lag

The delay between a write on the primary and when it appears on replicas.

```
T=0:   Write "DELIVERED" to Primary
T=0:   Primary has "DELIVERED"
T=50ms: Replica 1 has "DELIVERED" (50ms lag)
T=200ms: Replica 2 has "DELIVERED" (200ms lag)

If client reads from Replica 2 at T=100ms → gets "IN_TRANSIT" (stale!)
```

**Replication lag is the fundamental tradeoff of read replicas.** Strategies to handle it:

| Strategy | How |
|---|---|
| **Read from primary for critical reads** | Route writes and their subsequent reads to primary |
| **Read-your-writes** | After writing, read from primary (or wait for replica to catch up) |
| **Minimum lag threshold** | Only route reads to replicas with lag < 1 second |
| **Causal consistency** | Track write timestamps; only read from replicas that have caught up |

---

### Replication Topology 2: Multi-Leader (Active-Active)

**Multiple primaries** accept writes, each replicating to the others.

```
┌──────────┐     replication     ┌──────────┐
│ Primary A│ ←─────────────────→ │ Primary B│
│ (Mumbai) │                     │ (London) │
└──────────┘                     └──────────┘
  Writes ↑                         Writes ↑
  Reads  ↓                         Reads  ↓
  Asia clients                     Europe clients
```

**Why?** Geographic distribution — users in Asia write to the Mumbai primary with low latency; users in Europe write to the London primary.

**The problem: Write conflicts!**
```
User A (Mumbai):  UPDATE shipment SET carrier = "MAERSK"
User B (London):  UPDATE shipment SET carrier = "MSC"
(Same row, same time, different primaries)

When they replicate → CONFLICT! Which write wins?
```

**Conflict resolution strategies** (as covered in [[CAP Theorem]]):
- Last-Write-Wins (LWW)
- Custom application logic
- CRDTs

**Used by:** MySQL Group Replication, PostgreSQL BDR, CockroachDB, DynamoDB Global Tables.

---

### Replication Topology 3: Leaderless (Peer-to-Peer)

**No primary.** Any node can accept reads and writes. Uses quorums for consistency.

```
┌────────┐  ┌────────┐  ┌────────┐
│ Node 1 │  │ Node 2 │  │ Node 3 │
│  R/W   │  │  R/W   │  │  R/W   │
└────────┘  └────────┘  └────────┘
```

**Write:** Client sends write to **all nodes** (or W nodes). Success = W acknowledgments.
**Read:** Client reads from **R nodes** and takes the most recent value.

With **W + R > N** → guaranteed to read the latest write (as discussed in [[CAP Theorem]]).

**Used by:** Cassandra, DynamoDB, Riak.

---

### Replication Comparison

| Topology | Write Throughput | Consistency | Conflict Handling | Complexity |
|---|---|---|---|---|
| **Single-Leader** | Limited (one primary) | Strong (sync) or Eventual (async) | No conflicts | Low |
| **Multi-Leader** | High (multiple primaries) | Eventual | Must resolve conflicts | High |
| **Leaderless** | High (any node writes) | Tunable (quorum) | Must resolve conflicts | High |

---

### Failover — When the Primary Dies

```
Normal operation:
  Primary (healthy) → Replica 1, Replica 2

Primary crashes:
  1. Detection: Replicas notice primary isn't responding (heartbeat timeout)
  2. Election: Replicas elect a new primary (Replica 1 wins)
  3. Promotion: Replica 1 becomes new primary
  4. Reconfiguration: All clients and other replicas point to new primary
  5. Recovery: Old primary comes back → becomes a replica

    ┌──────────────┐           ┌──────────────┐
    │ Old Primary  │ (crashed) │ Replica 1    │ → promoted to Primary
    │              │           │ (new Primary)│
    └──────────────┘           └──────────────┘
```

**Problems during failover:**
- **Data loss:** If async replication, the new primary might be missing recent writes
- **Split-brain:** If old primary comes back and doesn't know it was replaced → two primaries
- **Downtime:** Time between primary crash and new primary election = unavailability

**Tools for automated failover:**
- PostgreSQL: Patroni, pg_auto_failover
- MySQL: MySQL InnoDB Cluster, Orchestrator
- Redis: Redis Sentinel

---

## Phase 3: Sharding — Splitting Data

### What is Sharding?

Sharding (also called **partitioning**) is splitting your dataset across multiple database servers, where each server holds a **subset** of the data.

```
Before (single DB): All 10M shipments on one server

After (4 shards):
  Shard 1: Shipments with ID hash 0-24%   → 2.5M rows
  Shard 2: Shipments with ID hash 25-49%  → 2.5M rows
  Shard 3: Shipments with ID hash 50-74%  → 2.5M rows
  Shard 4: Shipments with ID hash 75-100% → 2.5M rows
```

**Replication vs Sharding:**
```
Replication: Same data copied to multiple servers (read scaling, availability)
Sharding:    Different data on different servers (write scaling, storage scaling)

In practice, you combine BOTH:
  Shard 1 (data subset A) → Primary + 2 Replicas
  Shard 2 (data subset B) → Primary + 2 Replicas
  Shard 3 (data subset C) → Primary + 2 Replicas
```

---

### Sharding Strategy 1: Range-Based Sharding

Divide data based on **ranges of the shard key**.

```
Shard key: shipment creation date

Shard 1: Jan 2024 - Mar 2024
Shard 2: Apr 2024 - Jun 2024
Shard 3: Jul 2024 - Sep 2024
Shard 4: Oct 2024 - Dec 2024
```

**Pros:**
- Simple to implement and understand
- Range queries are efficient (e.g., "all shipments in Q1" → query Shard 1 only)

**Cons:**
- **Hot spots!** — If most activity is on recent data, the latest shard gets all the traffic while older shards sit idle.

```
January: Shard 1 handles 90% of traffic (hot! 🔥)
         Shard 2, 3, 4 are nearly idle
```

**When to use:** When data access is evenly distributed across ranges (e.g., sharding by region where all regions have similar traffic).

---

### Sharding Strategy 2: Hash-Based Sharding

Apply a **hash function** to the shard key and distribute based on hash value.

```
shard_number = hash(shipment_id) % number_of_shards

hash("SHP-001") % 4 = 2 → Shard 2
hash("SHP-002") % 4 = 0 → Shard 0
hash("SHP-003") % 4 = 3 → Shard 3
hash("SHP-004") % 4 = 1 → Shard 1
```

**Pros:**
- Even distribution of data across shards (no hot spots)
- Works well with any type of key

**Cons:**
- Range queries are **impossible** — "all shipments from January" would need to query ALL shards
- Adding/removing shards requires **rehashing** and moving data

**When to use:** Key-based lookups (get shipment by ID) where range queries are not needed.

---

### Sharding Strategy 3: Consistent Hashing

An improvement over basic hash sharding — when adding/removing shards, only a **fraction** of keys need to move.

```
Hash ring (0 to 2^32):

    Shard A (position 1000)
        ↓
  0 ────●────────●────────●────────●──── 2^32
                Shard B    Shard C   Shard D
               (pos 5000) (pos 9000)(pos 13000)

Key "SHP-001" hashes to position 3000
  → Goes clockwise → lands on Shard B (next shard at 5000)

Key "SHP-002" hashes to position 11000
  → Goes clockwise → lands on Shard D (next shard at 13000)
```

**Adding a new shard:**
```
Add Shard E at position 7000:
  Only keys between Shard B (5000) and Shard E (7000) need to move
  All other keys stay where they are!

Without consistent hashing:
  hash(key) % 4 → hash(key) % 5 → MOST keys move!
```

This is the [[Hash Ring]] concept from your earlier notes.

**Used by:** Cassandra, DynamoDB, Redis Cluster.

---

### Sharding Strategy 4: Directory-Based Sharding

A **lookup table** (directory) maps each key to its shard.

```
Lookup Table:
  customer_id 1-1000    → Shard 1
  customer_id 1001-5000 → Shard 2
  customer_id 5001+     → Shard 3
  
  OR more granularly:
  customer "ACME Corp"  → Shard 1
  customer "GlobalFreight" → Shard 2
```

**Pros:** Maximum flexibility — can move specific customers to specific shards (useful for large tenants).
**Cons:** The directory itself becomes a single point of failure and bottleneck.

**When to use:** Multi-tenant SaaS where large customers need isolated shards.

---

### Sharding Strategy 5: Geographic Sharding

Distribute data based on **geographic region**.

```
Shard "Asia":    All shipments originating in Asia → DB in Mumbai
Shard "Europe":  All shipments originating in Europe → DB in Frankfurt
Shard "Americas": All shipments originating in Americas → DB in Virginia
```

**Pros:** Low latency for regional users; data residency compliance (GDPR).
**Cons:** Cross-region queries (shipment from Asia to Europe) may need to touch multiple shards.

**Logistics example:** A global freight platform shards by origin region. The Mumbai team creating bookings for Asia-origin shipments gets low-latency writes to the local shard.

---

### Sharding Strategy Comparison

| Strategy | Distribution | Range Queries | Resharding Cost | Hot Spots |
|---|---|---|---|---|
| **Range** | Uneven (possible) | ✅ Efficient | Low | High risk |
| **Hash** | Even | ❌ All shards | High (rehash all) | Low |
| **Consistent Hash** | Even | ❌ All shards | Low (minimal moves) | Low |
| **Directory** | Flexible | Depends | Medium (update table) | Controllable |
| **Geographic** | By region | Within region | Medium | Possible (popular regions) |

---

## Phase 4: Choosing a Shard Key

The shard key is the **most important decision** in sharding. A bad shard key leads to hot spots, unbalanced shards, and cross-shard queries.

### Criteria for a Good Shard Key

| Criteria | Why |
|---|---|
| **High cardinality** | Many unique values → data spreads across many shards |
| **Even distribution** | All shards get roughly equal data and traffic |
| **Query alignment** | Most queries include the shard key → avoid cross-shard queries |
| **Immutable** | The key value doesn't change (changing it requires moving data) |

### Examples

| Domain | Good Shard Key | Bad Shard Key | Why |
|---|---|---|---|
| Shipments | `shipment_id` | `status` | Status has only 5 values → uneven distribution |
| E-commerce | `customer_id` | `product_category` | Category = few values; customer = many values |
| Multi-tenant SaaS | `tenant_id` | `created_date` | Tenant aligns with query patterns |
| Chat messages | `conversation_id` | `message_id` | Most queries fetch by conversation |
| GPS tracking | `vehicle_id` | `timestamp` | Most queries fetch by vehicle |

### The Shard Key Trap — Large Tenants

```
Shard key: tenant_id
  Tenant "SmallCo":   1,000 shipments
  Tenant "MegaCorp": 5,000,000 shipments

Result: The shard holding MegaCorp is 5000x larger than SmallCo's shard → HOT SPOT
```

**Solutions:**
1. **Compound shard key:** `tenant_id + shipment_id_prefix` → spreads large tenants across shards
2. **Dedicated shard:** Give MegaCorp its own shard (directory-based)
3. **Shard splitting:** Automatically split shards that grow too large

---

## Phase 5: Cross-Shard Operations — The Hard Parts

### Cross-Shard Queries

When a query doesn't include the shard key, it must **scatter** to all shards and **gather** results.

```
Shard key: shipment_id

Query: SELECT * FROM shipments WHERE customer_id = 'CUST-123'
→ customer_id is NOT the shard key
→ Must query ALL shards → merge results → slow!

Query: SELECT * FROM shipments WHERE shipment_id = 'SHP-001'
→ shipment_id IS the shard key
→ Query ONE shard → fast!
```

**This is called "scatter-gather" and should be minimized.**

**Solutions:**
1. **Choose the right shard key** that aligns with your most common queries
2. **Denormalize** — duplicate data so queries can be served from one shard
3. **Build a search index** (Elasticsearch) that can query across all shards
4. **Secondary index** — maintain a global index that maps non-shard-key values to shard locations

---

### Cross-Shard Joins

Joins across shards are extremely expensive:

```
Shard 1 has: shipment SHP-001 (customer_id = CUST-A)
Shard 3 has: customer CUST-A

SELECT s.*, c.name 
FROM shipments s JOIN customers c ON s.customer_id = c.id
WHERE s.id = 'SHP-001'

→ Must fetch from Shard 1 (shipment) AND Shard 3 (customer)
→ Join happens at the application layer → slow, complex
```

**Solutions:**
1. **Co-locate related data** — shard customers and their shipments on the same shard (same shard key)
2. **Denormalize** — store customer name directly in the shipments table
3. **Application-level joins** — query both shards, join in application code
4. **Avoid cross-shard joins** in the design phase — restructure your data model

---

### Cross-Shard Transactions

ACID transactions across shards are the **hardest problem** in distributed databases.

```
Transfer $100 from Account A (Shard 1) to Account B (Shard 3):
  Shard 1: Debit Account A by $100
  Shard 3: Credit Account B by $100

  If Shard 3 fails after Shard 1 succeeds → $100 disappeared!
```

**Solutions:**

#### Two-Phase Commit (2PC)

A coordinator asks all shards to prepare, then commit:

```
Phase 1 (Prepare):
  Coordinator → Shard 1: "Can you debit $100?" → Shard 1: "YES, prepared"
  Coordinator → Shard 3: "Can you credit $100?" → Shard 3: "YES, prepared"

Phase 2 (Commit):
  Coordinator → Shard 1: "COMMIT" → Done ✅
  Coordinator → Shard 3: "COMMIT" → Done ✅

If either shard says "NO" in Phase 1:
  Coordinator → ALL shards: "ABORT" → Rollback
```

**Pros:** Strong consistency.
**Cons:** Slow (multiple round trips), coordinator is SPOF, locks held during entire process.

#### Saga Pattern

As covered in [[Microservices Patterns]] — each shard operation has a compensating transaction:

```
Step 1: Shard 1 debit $100 → success
Step 2: Shard 3 credit $100 → FAILS
Compensate: Shard 1 re-credit $100 (undo step 1)
```

**Pros:** No distributed locks, higher availability.
**Cons:** Eventual consistency, complex compensation logic.

---

## Phase 6: Resharding — Adding More Shards

### The Problem

Your 4 shards are full and you need to add 2 more. This means **moving data** from existing shards to new ones.

```
Before: 4 shards, each with 25% of data
After:  6 shards, each should have ~16.7% of data

Data that needs to move: ~33% of all data needs to shift to new shards
```

### Resharding Strategies

#### 1. Consistent Hashing (Minimal Data Movement)

Only data between the new shard's position and its neighbor needs to move:
```
Add Shard 5 between Shard 2 and Shard 3:
  Only ~16% of data moves (from Shard 3 to Shard 5)
  Other shards: untouched!
```

**Used by:** Cassandra, DynamoDB (automatic), Redis Cluster.

#### 2. Double-and-Split

Double the number of shards by splitting each shard in half:

```
Before: Shard 1 (hash 0-49%), Shard 2 (hash 50-99%)
After:  Shard 1a (0-24%), Shard 1b (25-49%), Shard 2a (50-74%), Shard 2b (75-99%)

Each new shard copies half its parent's data.
```

#### 3. Online Resharding (Zero Downtime)

```
1. Create new shard
2. Start copying data from source shard → new shard (background)
3. While copying, dual-write: new writes go to BOTH old and new shard
4. Once copy is complete, update routing to include new shard
5. Stop writing to old shard for moved keys
6. Clean up moved data from old shard
```

**This is complex but avoids downtime.**

### Auto-Sharding vs Manual Sharding

| Approach | Examples | Pros | Cons |
|---|---|---|---|
| **Auto-sharding** | Cassandra, DynamoDB, CockroachDB, MongoDB | Automatic rebalancing, less ops | Less control, unexpected data movement |
| **Manual sharding** | Application-level sharding (any DB) | Full control, predictable | More operational effort, error-prone |

---

## Phase 7: Sharding in Practice

### MongoDB Sharding

```javascript
// Enable sharding on a database
sh.enableSharding("logistics");

// Shard the shipments collection on shipment_id (hashed)
sh.shardCollection("logistics.shipments", { "shipmentId": "hashed" });

// Shard the bookings collection on customer_id (ranged)
sh.shardCollection("logistics.bookings", { "customerId": 1 });
```

```
MongoDB Sharded Cluster:
  ┌─────────┐
  │  mongos  │  ← Query router (clients connect here)
  │ (router) │
  └────┬─────┘
       │
  ┌────┴─────┐
  │  Config   │  ← Stores shard metadata (which shard has which data)
  │  Servers  │
  └────┬─────┘
       │
  ┌────┼────────────┐
  ↓    ↓             ↓
Shard 1  Shard 2   Shard 3   ← Each is a replica set (primary + replicas)
```

---

### PostgreSQL Sharding (Citus)

```sql
-- Create a distributed table (sharded by shipment_id)
SELECT create_distributed_table('shipments', 'shipment_id');

-- Queries with shipment_id → routed to one shard
SELECT * FROM shipments WHERE shipment_id = 'SHP-001';
-- → hits ONE shard ✅

-- Queries without shard key → scatter-gather
SELECT COUNT(*) FROM shipments WHERE status = 'DELIVERED';
-- → hits ALL shards, results merged ⚠️
```

---

### Cassandra Partitioning

Cassandra uses consistent hashing natively:

```sql
CREATE TABLE shipments (
    shipment_id TEXT,
    event_time TIMESTAMP,
    status TEXT,
    carrier TEXT,
    PRIMARY KEY (shipment_id, event_time)
);
-- shipment_id = partition key (determines which node stores the data)
-- event_time = clustering key (determines sort order within the partition)

-- This query is efficient (hits one partition):
SELECT * FROM shipments WHERE shipment_id = 'SHP-001';

-- This query is INEFFICIENT (full table scan):
SELECT * FROM shipments WHERE carrier = 'MAERSK';
```

---

### Application-Level Sharding (Spring Boot)

When your database doesn't support native sharding, you implement it in the application:

```java
@Component
public class ShardRouter {
    
    private final List<DataSource> shards;  // 4 shard DataSources
    
    public DataSource getShardForShipment(String shipmentId) {
        int shardIndex = Math.abs(shipmentId.hashCode()) % shards.size();
        return shards.get(shardIndex);
    }
}

@Service
public class ShipmentRepository {
    
    private final ShardRouter router;
    
    public Shipment findById(String shipmentId) {
        DataSource shard = router.getShardForShipment(shipmentId);
        // Query ONLY this shard
        return new JdbcTemplate(shard)
            .queryForObject("SELECT * FROM shipments WHERE id = ?", 
                            shipmentMapper, shipmentId);
    }
    
    public List<Shipment> findByCustomer(String customerId) {
        // Customer ID is NOT the shard key → must query ALL shards
        return shards.stream()
            .flatMap(shard -> new JdbcTemplate(shard)
                .query("SELECT * FROM shipments WHERE customer_id = ?", 
                       shipmentMapper, customerId)
                .stream())
            .collect(Collectors.toList());
    }
}
```

---

## Phase 8: Replication + Sharding Combined

In production, you almost always use **both** — sharding for write scaling and replication for read scaling and availability.

```
                      ┌─────────────────────────────────────────┐
                      │            Sharded + Replicated          │
                      │                                         │
                      │  Shard 1 (shipments A-M)                │
                      │    Primary ←→ Replica 1 ←→ Replica 2    │
                      │                                         │
                      │  Shard 2 (shipments N-Z)                │
                      │    Primary ←→ Replica 1 ←→ Replica 2    │
                      │                                         │
                      │  Shard 3 (bookings)                     │
                      │    Primary ←→ Replica 1 ←→ Replica 2    │
                      └─────────────────────────────────────────┘

Write path: Router → correct shard → Primary
Read path:  Router → correct shard → any Replica (or Primary)
```

**This gives you:**
- **Write scaling** — writes distributed across shard primaries
- **Read scaling** — reads distributed across replicas within each shard
- **High availability** — if a primary fails, its replica takes over
- **Storage scaling** — total storage = sum of all shards

---

## Phase 9: When Sharding is an Overhead

### Scenario 1: Data Fits on One Server

If your dataset is 50 GB and a single server has 256 GB RAM → everything fits in memory. Sharding adds complexity for no benefit.

**Solution:** Vertical scaling + read replicas.

---

### Scenario 2: Read-Heavy, Write-Light Workload

If writes are 100/sec (easily handled by one server) but reads are 50,000/sec:

```
Sharding solves: write scaling, storage scaling
You need: read scaling

Solution: Single primary + 5 read replicas
  → Handles 50,000 reads/sec with zero sharding complexity
```

---

### Scenario 3: Many Cross-Shard Queries

If 80% of your queries don't include the shard key → 80% of queries scatter to ALL shards → worse performance than a single server.

```
Before sharding: 1 server, query takes 5ms
After sharding (4 shards): scatter to 4 servers, merge, takes 20ms + overhead

Sharding made things SLOWER.
```

---

### Scenario 4: Small Team, Early Stage

Sharding requires:
- Shard routing logic
- Cross-shard query handling
- Resharding procedures
- Monitoring per-shard health

For a 3-person team, this operational overhead is crippling.

---

### The Sharding Overhead Checklist

- [ ] Has vertical scaling reached its limit?
- [ ] Does the write volume exceed one server's capacity?
- [ ] Is the dataset too large for one server's storage?
- [ ] Are most queries shard-key-aligned (avoid scatter-gather)?
- [ ] Do you have the operational maturity to manage multiple shards?

> **Rule of thumb:** Don't shard until you absolutely must. Try vertical scaling → read replicas → caching → query optimization → THEN sharding.

---

## Phase 10: Real-World Case Studies

### Case 1: Logistics — Shipment Data Sharding

**Problem:** A global freight platform stores 500M shipment records. Single PostgreSQL instance struggles with write throughput and storage.

**Solution:**
```
Shard key: shipment_id (hashed)
Shards: 8 PostgreSQL instances
Replication: Each shard has 1 primary + 2 replicas

Write path: Application → hash(shipment_id) % 8 → shard primary
Read path: Application → hash(shipment_id) % 8 → shard replica

Cross-shard query (search by customer):
  → Elasticsearch index for cross-shard searches
  → Returns shipment_ids → fetch from correct shards
```

**Result:** Write throughput increased 6x. Storage distributed across 8 servers.

---

### Case 2: Logistics — Multi-Tenant Sharding

**Problem:** A logistics SaaS serves 200 customers. One customer (MegaCorp) generates 60% of all data.

**Solution:**
```
Directory-based sharding:
  MegaCorp → Dedicated shard (Shard 1, beefy hardware)
  Top 10 customers → Shared shard (Shard 2)
  Remaining 189 customers → Shared shard (Shard 3)

Lookup table:
  tenant_id → shard_id mapping stored in Redis
```

**Result:** MegaCorp gets dedicated resources without affecting other customers.

---

### Case 3: Logistics — Event Store Sharding

**Problem:** A shipment event store receives 100,000 events/sec. Single database can't keep up.

**Solution:**
```
Technology: Cassandra (natively sharded via consistent hashing)
Partition key: shipment_id
Clustering key: event_timestamp

All events for one shipment → same partition → ordered by time
10 Cassandra nodes with replication factor 3

Write path: Any node → routes to correct partition → replicates to 2 peers
Read path: SELECT * FROM events WHERE shipment_id = 'SHP-001'
           → hits ONE partition → returns ordered events
```

**Result:** 100,000 writes/sec sustained across the cluster. Full event history per shipment in <5ms.

---

## Key Takeaways

1. **Replication** = copies of the same data for availability and read scaling
2. **Sharding** = splitting data across servers for write scaling and storage scaling
3. **Single-leader replication** is the most common — sync for consistency, async for speed
4. **Replication lag** is the fundamental tradeoff of read replicas
5. **Shard key is the most important decision** — high cardinality, even distribution, query-aligned
6. **Hash-based sharding** avoids hot spots but kills range queries
7. **Consistent hashing** minimizes data movement when adding/removing shards
8. **Cross-shard operations** (queries, joins, transactions) are expensive — design to avoid them
9. **Combine replication + sharding** in production for full scaling
10. **Don't shard early** — try vertical scaling → read replicas → caching → query optimization first

---

**Previous:** [[CAP Theorem]]
**See also:** [[Databases]], [[Hash Ring]], [[Caching]]
