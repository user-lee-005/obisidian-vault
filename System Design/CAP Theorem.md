# CAP Theorem

> *"You can have it all — just not at the same time."*

---

## Phase 1: The Fundamentals

### What is the CAP Theorem?

The CAP Theorem (also known as **Brewer's Theorem**, proposed by Eric Brewer in 2000) states that in a **distributed data system**, you can only guarantee **two out of three** properties simultaneously:

- **C** — Consistency
- **A** — Availability
- **P** — Partition Tolerance

```
        Consistency
           /\
          /  \
         / CP \
        /______\
       /\      /\
      /  \ CA /  \
     / AP \  /    \
    /______\/______\
Availability    Partition Tolerance
```

### The Three Properties Explained

#### C — Consistency

**Every read receives the most recent write or an error.**

All nodes in the system see the **same data at the same time**. If you write to Node A, an immediate read from Node B returns the updated value.

```
Write: shipment.status = "DELIVERED" → Node A

Consistent system:
  Read from Node A → "DELIVERED" ✅
  Read from Node B → "DELIVERED" ✅ (same value, even though write went to Node A)

Inconsistent system:
  Read from Node A → "DELIVERED" ✅
  Read from Node B → "IN_TRANSIT" ❌ (stale! hasn't synced yet)
```

**Analogy:** Imagine a logistics company with offices in Mumbai and Chennai. When a shipment status is updated in Mumbai, consistency means the Chennai office **immediately** sees the same status.

---

#### A — Availability

**Every request receives a (non-error) response — without guarantee that it contains the most recent write.**

The system is always responsive. Every request gets a response, even if some nodes are down or unreachable.

```
Available system:
  Node A is down. Node B is slow. Node C is healthy.
  Request → Node C → Response ✅ (might be slightly stale, but you get a response)

Unavailable system:
  Node A is down.
  Request → "503 Service Unavailable" ❌ (system refuses to answer)
```

**Analogy:** The Chennai office phone always gets answered. The person answering may not have the latest shipment status, but they'll always give you *some* answer rather than hanging up.

---

#### P — Partition Tolerance

**The system continues to operate despite network partitions (communication breakdowns between nodes).**

A **network partition** is when nodes can't communicate with each other — the network between them is broken.

```
Normal operation:
  Node A ←──network──→ Node B ←──network──→ Node C
  (All nodes can communicate)

Network partition:
  Node A ←──network──→ Node B    ✗    Node C
  (Node C is isolated — can't talk to A or B)
```

**Analogy:** The internet connection between the Mumbai and Chennai offices goes down. Partition tolerance means both offices continue to operate independently, even though they can't sync.

---

### Why Can't We Have All Three?

In a distributed system, **network partitions are inevitable** (hardware fails, cables get cut, data centers lose connectivity). So **P is not optional** — you must always handle partitions.

The real choice is: **when a partition occurs**, do you choose **Consistency or Availability?**

```
Network partition happens between Node A and Node B:

Option 1: Choose Consistency (CP)
  → Stop accepting writes until partition is resolved
  → Return errors for reads that can't be verified as up-to-date
  → System is consistent but partially unavailable

Option 2: Choose Availability (AP)
  → Both nodes continue accepting reads and writes independently
  → When partition heals, resolve conflicts (which write wins?)
  → System is available but may return stale data
```

---

## Phase 2: CP, AP, and CA Systems

### CP — Consistency + Partition Tolerance

**When a partition occurs, the system blocks or returns errors rather than risk serving stale data.**

```
Partition occurs between Node A and Node B:

Client → Node A: "What's the status of SHP-001?"
Node A: "I can't reach Node B to verify. Returning error." ❌

Client → Node A: "Update SHP-001 to DELIVERED"
Node A: "I can't replicate to Node B. Refusing write." ❌

System is CONSISTENT (no stale reads) but NOT AVAILABLE (returns errors)
```

**Real-world examples:**

| System | Why CP |
|---|---|
| **MongoDB** (with majority reads) | Writes require majority acknowledgment; during partition, minority nodes reject writes |
| **HBase** | Relies on ZooKeeper for consensus; if quorum is lost, regions become unavailable |
| **Redis Cluster** (default) | If a master and its replica are on different sides of a partition, the minority side refuses writes |
| **ZooKeeper** | Provides strong consistency; minority partition can't elect a leader → refuses requests |
| **Etcd** | Raft consensus; minority partition stops serving |

**When to choose CP:**
- Financial transactions — showing wrong balance is worse than temporary downtime
- Inventory systems — over-selling is more costly than briefly blocking orders
- Leader election — split-brain is catastrophic
- Configuration management — stale config can cause system-wide failures

**Logistics example:** A container allocation system. If two booking services on different sides of a partition both think a container is available, they could **double-book it**. CP prevents this by blocking bookings during the partition.

---

### AP — Availability + Partition Tolerance

**When a partition occurs, the system continues serving requests, even if some responses may be stale.**

```
Partition occurs between Node A and Node B:

Client → Node A: "What's the status of SHP-001?"
Node A: "Last I know, it's IN_TRANSIT" ✅ (might be stale if Node B has an update)

Client → Node B: "Update SHP-001 to DELIVERED"
Node B: "Done!" ✅ (but Node A doesn't know about this yet)

System is AVAILABLE (always responds) but NOT CONSISTENT (nodes disagree)
```

When the partition heals, the system must **reconcile conflicts**:
```
Partition heals:
  Node A has: SHP-001 = "IN_TRANSIT" (old)
  Node B has: SHP-001 = "DELIVERED" (new)
  Conflict resolution: Last-write-wins → "DELIVERED" propagated to Node A
```

**Real-world examples:**

| System | Why AP |
|---|---|
| **Cassandra** | Tunable consistency; can serve reads/writes from any node during partition |
| **DynamoDB** | Always writable; uses vector clocks for conflict resolution |
| **CouchDB** | Designed for offline-first; syncs when connectivity returns |
| **DNS** | Serves cached (potentially stale) records even during outages |
| **CDN caches** | Serve stale content rather than failing |

**When to choose AP:**
- Social media feeds — slightly stale timeline is better than no timeline
- Product catalogs — showing yesterday's price is better than showing nothing
- DNS — stale DNS record is better than no resolution
- Search indices — slightly outdated results are acceptable

**Logistics example:** A shipment tracking portal. During a network partition, it's better to show the **last known status** ("In Transit, last updated 5 min ago") than to show an error page. The customer gets useful information even if it's slightly behind.

---

### CA — Consistency + Availability (Without Partition Tolerance)

**Theoretically:** The system is always consistent and always available — but can't handle network partitions.

**In practice:** This only works on a **single node** or systems where network partitions are "impossible" (single-machine databases).

```
Single PostgreSQL instance:
  ✅ Consistent: one copy of data, always up-to-date
  ✅ Available: responds to every request
  ❌ Not partition tolerant: if the single node goes down, everything is down
```

**Real-world examples:**

| System | Why CA |
|---|---|
| **Single-node PostgreSQL** | One node = no partition possible, but no fault tolerance |
| **Single-node MySQL** | Same as above |
| **Traditional RDBMS** (single instance) | ACID guarantees on a single machine |

> ⚠️ **Important:** CA is largely theoretical for distributed systems. In any real distributed system, partitions WILL happen, so you must choose between CP and AP. CA only exists when you're running a single database instance.

---

### The Spectrum, Not a Binary

In reality, CAP is not a strict "pick 2" choice. Modern systems operate on a **spectrum**:

```
Strong Consistency ◄──────────────────────────────► High Availability
     (CP)                                              (AP)

     ┌─────┬──────┬────────┬─────────┬──────┬─────────┐
     │     │      │        │         │      │         │
   ZooKeeper MongoDB  MySQL   Cassandra  DynamoDB  DNS
   (strict)  (tunable)  (Cluster)  (tunable)  (eventual) (stale OK)
```

Many modern databases let you **tune the consistency level per query**:

```
Cassandra:
  SELECT * FROM shipments WHERE id = 'SHP-001'
  CONSISTENCY ONE;      // Fast, AP — read from any one node
  
  SELECT * FROM shipments WHERE id = 'SHP-001'
  CONSISTENCY QUORUM;   // Slower, more consistent — read from majority
  
  SELECT * FROM shipments WHERE id = 'SHP-001'
  CONSISTENCY ALL;      // Slowest, CP — read from every node
```

---

## Phase 3: Consistency Models

Understanding the different levels of consistency is essential for making practical decisions.

### Strong Consistency (Linearizability)

**Every read sees the most recent write.** All nodes agree on the order of operations.

```
T=0: Client A writes shipment.status = "DELIVERED" to Node 1
T=1: Client B reads shipment.status from Node 2
Result: "DELIVERED" ✅ (guaranteed)
```

**How it works:** The write doesn't return success until ALL replicas are updated.
**Cost:** High latency (must wait for all nodes), reduced availability during partitions.
**Used by:** ZooKeeper, etcd, single-node databases.

---

### Sequential Consistency

All nodes see operations in the **same order**, but the order may not match real-time wall clock.

```
Client A: write X = 1, write X = 2
Client B: read X → might see 1 first, then 2 (correct order)
          But there may be a delay before seeing the latest.
```

---

### Causal Consistency

**Causally related operations** are seen in the same order by all nodes. Unrelated operations can be seen in any order.

```
Client A: posts "Shipment picked up" (event 1)
Client A: posts "Shipment in transit" (event 2, caused by event 1)

Causal consistency guarantees:
  Every node sees event 1 before event 2 (causal relationship preserved)
  
But unrelated events from different clients may appear in different orders on different nodes.
```

---

### Eventual Consistency

**If no new writes occur, all nodes will eventually converge** to the same value. But there's no guarantee on how long "eventually" takes.

```
T=0: Client writes shipment.status = "DELIVERED" to Node 1
T=1: Client reads from Node 2 → "IN_TRANSIT" ❌ (stale, but temporary)
T=5: Client reads from Node 2 → "IN_TRANSIT" ❌ (still propagating)
T=10: Client reads from Node 2 → "DELIVERED" ✅ (eventually consistent)
```

**Used by:** DynamoDB, Cassandra (with ONE consistency), DNS, CDN caches.

---

### Read-Your-Writes Consistency

**A client always sees its own writes.** Other clients may see stale data.

```
Client A: writes shipment.status = "DELIVERED"
Client A: reads shipment.status → "DELIVERED" ✅ (sees own write)
Client B: reads shipment.status → "IN_TRANSIT" (might be stale) — acceptable
```

**Implementation:** Route reads to the same node that handled the write, or use session stickiness.

This is what we discussed in the [[Caching]] note — ensuring users see their own updates immediately.

---

### Comparison

| Model | Guarantee | Latency | Availability |
|---|---|---|---|
| **Strong** | Always latest value | Highest | Lowest |
| **Sequential** | Same order everywhere | High | Low |
| **Causal** | Causally related = ordered | Medium | Medium |
| **Eventual** | Will converge... eventually | Lowest | Highest |
| **Read-your-writes** | See your own writes | Low | High |

---

## Phase 4: The PACELC Theorem — Extending CAP

### CAP's Limitation

CAP only describes behavior **during a partition**. But partitions are rare. What about the 99.9% of the time when everything is working fine?

### PACELC

Proposed by Daniel Abadi in 2012:

> **If there's a Partition (P), choose between Availability (A) and Consistency (C). Else (E), when operating normally, choose between Latency (L) and Consistency (C).**

```
if (partition) {
    choose: Availability OR Consistency
} else {
    choose: Latency OR Consistency
}
```

### PACELC Classification

| System | During Partition (PAC) | Normal Operation (ELC) | Classification |
|---|---|---|---|
| **DynamoDB** | Choose Availability | Choose Latency | **PA/EL** |
| **Cassandra** | Choose Availability | Choose Latency | **PA/EL** |
| **MongoDB** | Choose Consistency | Choose Consistency | **PC/EC** |
| **HBase** | Choose Consistency | Choose Consistency | **PC/EC** |
| **MySQL Cluster** | Choose Availability | Choose Consistency | **PA/EC** |
| **PNUTS (Yahoo)** | Choose Consistency | Choose Latency | **PC/EL** |

**What this means practically:**

- **PA/EL (Cassandra, DynamoDB):** Always optimizes for speed. During normal operation, serves fastest response (possibly stale). During partition, stays available.
- **PC/EC (MongoDB, HBase):** Always optimizes for correctness. During normal operation, waits for replicas. During partition, blocks requests.

---

## Phase 5: Conflict Resolution

When an AP system operates during a partition, **both sides may accept writes** that conflict. When the partition heals, how do you resolve conflicts?

### Strategy 1: Last-Write-Wins (LWW)

The write with the **most recent timestamp** wins. The other is discarded.

```
Partition occurs:
  Node A: UPDATE shipment SET carrier = "MAERSK" at T=100
  Node B: UPDATE shipment SET carrier = "MSC"    at T=105

Partition heals:
  T=105 > T=100 → "MSC" wins. "MAERSK" is discarded.
```

**Problem:** Clock synchronization. What if Node A's clock is 10 seconds ahead? The "wrong" write could win.

**Used by:** Cassandra, DynamoDB.

---

### Strategy 2: Vector Clocks

Each node maintains a **version vector** to track causality across nodes.

```
Write 1: Node A writes v1 → vector clock: {A:1}
Write 2: Node B writes v2 → vector clock: {B:1}

These are concurrent (neither caused the other).
System detects conflict → application resolves it.

Write 3: Node A writes v3 after seeing v1 → vector clock: {A:2}
This is causally after v1 → no conflict, just an update.
```

**Used by:** DynamoDB (simplified as version numbers), Riak.

---

### Strategy 3: CRDTs (Conflict-free Replicated Data Types)

Data structures that **automatically merge** without conflicts, regardless of order.

```
G-Counter (Grow-only counter):
  Node A: counter_A = 5
  Node B: counter_B = 3
  
  Merge: total = max(counter_A) + max(counter_B) = 5 + 3 = 8
  No conflict possible!
  
OR-Set (Observed-Remove Set):
  Node A: adds "MAERSK" to carrier set
  Node B: adds "MSC" to carrier set
  
  Merge: carrier set = {"MAERSK", "MSC"}
  No conflict — both additions are preserved.
```

**Used by:** Redis (CRDT-based replication), Riak, collaborative editors.

---

### Strategy 4: Application-Level Resolution

Let the **application decide** how to merge conflicts, using business logic.

```java
public Shipment resolveConflict(Shipment versionA, Shipment versionB) {
    // Business rule: later status wins in the shipment lifecycle
    // CREATED < PICKED_UP < IN_TRANSIT < DELIVERED
    if (statusOrder(versionA.getStatus()) > statusOrder(versionB.getStatus())) {
        return versionA;
    }
    return versionB;
}
```

**Logistics example:** If Node A says "IN_TRANSIT" and Node B says "DELIVERED", business logic dictates "DELIVERED" wins because it's later in the lifecycle — regardless of timestamps.

---

## Phase 6: Quorum — Balancing Consistency and Availability

### What is a Quorum?

A quorum is the **minimum number of nodes** that must agree for an operation to succeed.

### The Formula

For a system with **N replicas**:
- **W** = number of nodes that must acknowledge a **write**
- **R** = number of nodes that must respond to a **read**

```
Strong consistency: W + R > N

Example with N=3 replicas:
  W=2, R=2: 2+2=4 > 3 ✅ (strong consistency)
  W=1, R=3: 1+3=4 > 3 ✅ (strong consistency, fast writes, slow reads)
  W=3, R=1: 3+1=4 > 3 ✅ (strong consistency, slow writes, fast reads)
  W=1, R=1: 1+1=2 ≤ 3 ❌ (eventual consistency — may read stale data)
```

**Why W + R > N guarantees consistency:** There's always **at least one node** that has both the latest write AND participates in the read → the read will see the latest value.

```
N=3, W=2, R=2:

Write "DELIVERED" acknowledged by Node A and Node B (W=2)

Read from Node B and Node C (R=2):
  Node B has "DELIVERED" ✅ (was part of write quorum)
  Node C has "IN_TRANSIT" (stale)
  
  System returns the newest: "DELIVERED" ✅
```

### Tuning for Your Use Case

| Configuration | Write Speed | Read Speed | Consistency | Use Case |
|---|---|---|---|---|
| W=1, R=N | Fastest writes | Slowest reads | Strong | Write-heavy, rare reads |
| W=N, R=1 | Slowest writes | Fastest reads | Strong | Read-heavy, rare writes |
| W=⌈(N+1)/2⌉, R=⌈(N+1)/2⌉ | Balanced | Balanced | Strong | General purpose |
| W=1, R=1 | Fastest | Fastest | Eventual | Maximum throughput, staleness OK |

### Cassandra Quorum Example

```sql
-- Write with quorum (majority must acknowledge)
INSERT INTO shipments (id, status) VALUES ('SHP-001', 'DELIVERED')
USING CONSISTENCY QUORUM;

-- Read with quorum (majority must respond)
SELECT * FROM shipments WHERE id = 'SHP-001'
CONSISTENCY QUORUM;

-- Fast but potentially stale read
SELECT * FROM shipments WHERE id = 'SHP-001'
CONSISTENCY ONE;
```

---

## Phase 7: Real-World Architecture Decisions

### Decision Framework

```
Question 1: Can my users tolerate stale data?
  YES → Lean toward AP (availability)
  NO  → Lean toward CP (consistency)

Question 2: Can my users tolerate downtime?
  YES → CP is acceptable
  NO  → AP is necessary

Question 3: How bad is a wrong answer vs no answer?
  Wrong answer is worse  → CP
  No answer is worse     → AP
```

### Logistics-Specific Decisions

| System | Choice | Reasoning |
|---|---|---|
| **Shipment tracking portal** | AP | Showing "last known status" is better than showing nothing; customers expect best-effort |
| **Container booking system** | CP | Double-booking a container costs $10,000+; blocking is better than inconsistency |
| **Freight rate calculator** | CP | Quoting a wrong rate can lead to financial loss; better to fail than give wrong price |
| **Shipment event log** | AP | Events can be reconciled later; losing availability means missing events entirely |
| **Carrier master data** | AP | Carrier names rarely change; stale data is harmless; always being available to look up is critical |
| **Invoice generation** | CP | Billing must be accurate; duplicate or incorrect invoices cause financial and trust issues |
| **Notification service** | AP | Sending a slightly delayed notification is better than failing to send any notification |
| **Customs declaration** | CP | Legal compliance requires accuracy; stale or incorrect data can result in fines |

---

## Phase 8: CAP in Practice — Database Selection

### How Popular Databases Handle CAP

| Database | Default CAP | Tunable? | How |
|---|---|---|---|
| **PostgreSQL** (single) | CA | No | Single node = no partitions |
| **PostgreSQL** (streaming replication) | CP | Partially | Synchronous replication = CP, async = AP |
| **MySQL** (Galera Cluster) | CP | No | All nodes must agree on writes |
| **MongoDB** | CP | Yes | `readConcern` and `writeConcern` settings |
| **Cassandra** | AP | Yes | Consistency level per query (ONE to ALL) |
| **DynamoDB** | AP | Partially | Eventually consistent by default, strongly consistent reads available |
| **Redis Cluster** | CP | Partially | Writes to master only; during partition, minority side is unavailable |
| **CockroachDB** | CP | No | Designed for strong consistency across regions |
| **Elasticsearch** | AP | No | Optimized for availability and search speed |
| **Neo4j** (Causal Cluster) | CP | Yes | Causal consistency by default |

### MongoDB Example

```javascript
// CP mode: wait for majority to acknowledge write
db.shipments.insertOne(
  { shipmentId: "SHP-001", status: "DELIVERED" },
  { writeConcern: { w: "majority", wtimeout: 5000 } }
);

// Read with majority read concern (see only majority-committed data)
db.shipments.find({ shipmentId: "SHP-001" })
  .readConcern("majority");

// AP mode: write to primary only, don't wait for replication
db.shipments.insertOne(
  { shipmentId: "SHP-001", status: "DELIVERED" },
  { writeConcern: { w: 1 } }
);
```

---

## Phase 9: Common Misconceptions

### Misconception 1: "CAP means pick 2 out of 3"

**Reality:** Since partitions are inevitable in distributed systems, you always have P. The real choice is C or A **during a partition**. When there's no partition, you can have both C and A.

### Misconception 2: "My database is CA"

**Reality:** A truly CA system can't handle network failures. If your "CA" database is a single-node PostgreSQL, it's not distributed. If it's distributed, it must handle partitions → it's either CP or AP.

### Misconception 3: "AP means data will be wrong"

**Reality:** AP means data **might be stale temporarily**, not that it's wrong forever. Eventually consistent systems converge to the correct state. The window of staleness can be milliseconds to seconds.

### Misconception 4: "CP means always slow"

**Reality:** CP adds latency only for **writes** (waiting for replicas). Reads can be fast if you read from the leader. The cost is **availability during partitions**, not latency during normal operation.

### Misconception 5: "You choose CP or AP for the entire system"

**Reality:** Different parts of your system can make different choices:
```
Booking system:      CP (can't double-book)
Tracking portal:     AP (stale status is OK)
Rate calculator:     CP (wrong rate = financial loss)
Notification system: AP (delayed notification > no notification)
```

---

## Phase 10: When CAP Doesn't Apply

### Single-Node Systems

If your database runs on a single server, there's no partition to worry about. CAP is irrelevant.

### Systems with a Single Leader

A system with one write leader and read replicas doesn't fully fit the CAP model:
- Writes go to leader → consistent
- Reads from replicas → may be stale (eventual consistency)
- If leader fails → availability or consistency depends on failover strategy

### Non-Distributed Systems

CAP applies to **distributed data stores**. It doesn't apply to:
- Local caches (single machine)
- File systems (single machine)
- In-memory data structures (single JVM)

---

## Key Takeaways

1. **CAP Theorem:** In a distributed system, during a network partition, choose Consistency or Availability
2. **Partition tolerance is mandatory** — networks WILL fail; the choice is always C vs A during partition
3. **CP systems** (MongoDB, HBase, ZooKeeper) sacrifice availability for correctness — block during partition
4. **AP systems** (Cassandra, DynamoDB, DNS) sacrifice consistency for availability — serve stale data during partition
5. **PACELC extends CAP** — during normal operation, choose between Latency and Consistency
6. **Consistency is a spectrum** — strong, sequential, causal, eventual, read-your-writes
7. **Quorum (W + R > N)** lets you tune consistency per query
8. **Conflict resolution** (LWW, vector clocks, CRDTs) is essential for AP systems
9. **Different parts of your system can make different CAP choices** — it's not one-size-fits-all
10. **For logistics:** booking/billing → CP; tracking/notifications → AP

---

**Next up:** [[Sharding and Replication]]
