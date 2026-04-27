# Leader Election and Consensus

> *"In a distributed system, the hardest problem isn't computation — it's getting everyone to agree."*

---

## Phase 1: The Fundamentals

### Why Do Distributed Systems Need Agreement?

In a distributed system, multiple nodes must coordinate. Many operations require **exactly one node** to act as the leader or decision-maker:

```
❌ Without coordination:
  Node A: "I'm the leader, processing order ORD-001"
  Node B: "I'm the leader, processing order ORD-001"  ← SPLIT BRAIN!
  → Order processed twice → duplicate shipment, double charge

✅ With consensus:
  All nodes agree: "Node A is the leader"
  Node A processes ORD-001
  Node B: "I'm a follower, standing by"
```

### What Problems Need Consensus?

| Problem | Why Agreement Is Needed |
|---|---|
| **Leader election** | Only one database primary should accept writes |
| **Distributed locks** | Only one service instance should process a task |
| **Configuration changes** | All nodes must agree on the new config |
| **Transaction commit** | All participants in a distributed transaction agree to commit or abort |
| **Membership changes** | All nodes agree on which nodes are in the cluster |
| **Log replication** | All nodes agree on the order of operations |

### The Core Challenge: Failures

In a distributed system, at any time:
- Nodes can **crash** and restart
- Network messages can be **delayed, lost, or duplicated**
- Nodes can be **partitioned** (unable to communicate)
- Clocks can **drift** (nodes disagree on the time)

Consensus algorithms must work correctly **despite all of these failures**.

---

## Phase 2: Leader Election

### What is Leader Election?

The process of **choosing one node** from a group to act as the **leader** (primary/master). The leader coordinates work, and all other nodes (followers) defer to it.

```
Cluster: [Node A, Node B, Node C]

Election result: Node B is the leader
  Node B: accepts writes, coordinates replication
  Node A: follower, receives replicated data
  Node C: follower, receives replicated data

Node B crashes:
  Node A and Node C detect the failure
  New election: Node A becomes the new leader
```

### Why Only One Leader?

The **split-brain problem** — if two nodes both think they're the leader:

```
Network partition:
  [Node A (thinks it's leader)] ←✗→ [Node B (thinks it's leader)]
  
  Client 1 → Node A: "Update balance to $100"
  Client 2 → Node B: "Update balance to $200"
  
  Partition heals → which balance is correct? $100? $200?
  DATA CORRUPTION
```

### Leader Election Approaches

#### Approach 1: Bully Algorithm

The node with the **highest ID** (or priority) becomes the leader.

```
Nodes: A(id=1), B(id=2), C(id=3)
Current leader: C

C crashes. B detects failure.
B starts election:
  B sends "ELECTION" to all nodes with higher IDs (only C, which is down)
  No response from C → B declares itself leader
  B sends "COORDINATOR" message to all nodes
  B is the new leader

C recovers:
  C has highest ID → "bullies" B → C takes over as leader again
```

**Pros:** Simple.
**Cons:** Highest ID always wins (no fairness). Not partition-tolerant. The "bully" node causes unnecessary leadership changes.

---

#### Approach 2: Ring-Based Election

Nodes arranged in a logical ring. Election messages travel around the ring.

```
Ring: A → B → C → D → A

D detects leader (C) is down:
  D sends ELECTION(D) to A
  A receives, adds itself: ELECTION(D, A), sends to B
  B receives, adds itself: ELECTION(D, A, B), sends to C
  C is down, skip → send to D
  D receives back → highest ID in list = D → D is the new leader
  D sends COORDINATOR(D) around the ring
```

---

#### Approach 3: Consensus-Based Election (The Standard)

Use a **consensus algorithm** (Raft, Paxos) to elect a leader. This is the most robust approach.

---

## Phase 3: The Raft Consensus Algorithm

Raft is the **most widely used and easiest to understand** consensus algorithm. Designed as an understandable alternative to Paxos.

### The Core Idea

Raft ensures all nodes in a cluster **agree on the same sequence of operations** (log entries), even if some nodes crash.

### Node States

Every node is in one of three states:

```
┌──────────┐  timeout  ┌───────────┐  wins election  ┌──────────┐
│ FOLLOWER │ ────────→ │ CANDIDATE │ ────────────────→│  LEADER  │
│          │ ←──────── │           │                   │          │
└──────────┘  discovers └───────────┘                  └──────────┘
              leader       ↑ loses election                │
                           └───────────────────────────────┘
                             discovers higher-term leader
```

**Follower:** Passive. Responds to requests from leader and candidates.
**Candidate:** Actively seeking to become leader (during election).
**Leader:** Handles all client requests and replicates log to followers.

### Terms

Raft divides time into **terms** (numbered sequentially). Each term begins with an election.

```
Term 1: Node A is leader → handles requests → crashes
Term 2: Election → Node B becomes leader → handles requests
Term 3: Election → Node C becomes leader → ...
```

Terms act as a **logical clock** — a node with a higher term number is "more current". If a node receives a message with a higher term, it immediately steps down.

### Leader Election in Raft

```
Step 1: Follower timeout
  Node B hasn't heard from the leader for a while (election timeout)
  Node B becomes a CANDIDATE

Step 2: Request votes
  Node B increments its term: term = 2
  Node B votes for itself
  Node B sends RequestVote RPC to all other nodes:
    "I'm Node B, term 2, my last log entry is at index 5. Vote for me?"

Step 3: Voting
  Each node votes for AT MOST ONE candidate per term
  Node A: "My last log index is 4. B's is 5 (more up-to-date). I vote YES."
  Node C: "My last log index is 5. B's is 5 (same). I vote YES."
  
Step 4: Majority wins
  Node B receives 3 votes (including self) out of 3 nodes → MAJORITY
  Node B becomes LEADER for term 2

Step 5: Heartbeats
  Node B sends periodic heartbeats (empty AppendEntries RPCs) to followers
  Followers reset their election timeouts on each heartbeat
  → Prevents unnecessary elections while leader is healthy
```

**Key election rules:**
1. Each node votes for **at most one candidate** per term
2. Candidate must have a **log at least as up-to-date** as the voter's log
3. Candidate needs votes from a **majority** of nodes
4. **Randomized election timeouts** prevent split votes (each node's timeout is different)

### Split Vote Prevention

```
Without randomized timeouts:
  Node A timeout at T=150ms → becomes candidate
  Node B timeout at T=150ms → becomes candidate
  Each votes for themselves → tie → no leader elected → try again → tie again...

With randomized timeouts:
  Node A timeout at T=150ms → becomes candidate → gets majority votes → LEADER
  Node B timeout at T=200ms → before it times out, receives heartbeat from A → stays FOLLOWER
```

### Log Replication in Raft

Once a leader is elected, it replicates operations to followers:

```
Client: "Set shipment SHP-001 status to DELIVERED"

Step 1: Leader appends to its log
  Leader log: [..., {index: 6, term: 2, cmd: "SET SHP-001 DELIVERED"}]

Step 2: Leader sends AppendEntries to all followers
  "Here's entry at index 6. Append it to your logs."

Step 3: Followers append and acknowledge
  Node A: appended → ACK
  Node C: appended → ACK

Step 4: Leader sees majority (2/3) acknowledged → COMMITS the entry
  Leader applies the command to its state machine
  Leader responds to client: "Success ✅"

Step 5: Leader notifies followers that entry is committed
  Followers apply the committed entry to their state machines
```

**Commit rule:** An entry is committed only when it's replicated to a **majority** of nodes. This guarantees that any future leader will have this entry.

```
5-node cluster: entry committed when stored on 3+ nodes
  Even if 2 nodes crash, the entry is safe on the remaining 3
  Any new leader must come from these 3 (majority) → new leader has the entry
```

---

## Phase 4: Paxos

### What is Paxos?

Paxos (by Leslie Lamport, 1989) is the **original** consensus algorithm. It's theoretically elegant but notoriously **difficult to understand and implement**.

> *"The dirty little secret of the distributed systems community is that Paxos is really hard to understand."* — Diego Ongaro (Raft creator)

### Basic Paxos (Single-Value Consensus)

Agrees on **a single value**. Three roles:

| Role | What They Do |
|---|---|
| **Proposer** | Proposes a value to be agreed upon |
| **Acceptor** | Votes on proposals |
| **Learner** | Learns the decided value |

```
Phase 1: PREPARE
  Proposer sends Prepare(n) to all Acceptors
    "I want to propose with proposal number n"
  
  Acceptors respond:
    "I promise not to accept any proposal with number < n"
    "Here's the highest-numbered proposal I've already accepted (if any)"

Phase 2: ACCEPT
  If Proposer receives promises from majority:
    Proposer sends Accept(n, value) to all Acceptors
      (If any acceptor already accepted a value, proposer must use THAT value)
    
  Acceptors:
    If they haven't promised to a higher proposal → Accept
    Notify Learners of the accepted value

Phase 3: LEARN
  Learner receives accepted value from majority of Acceptors → DECIDED
```

### Multi-Paxos

Runs multiple rounds of Paxos for a sequence of values (like a replicated log). A stable leader skips Phase 1 for subsequent rounds.

### Paxos vs Raft

| Aspect | Paxos | Raft |
|---|---|---|
| **Year** | 1989 | 2014 |
| **Understandability** | Very difficult | Designed for clarity |
| **Leader** | Optional (can be leaderless) | Required (strong leader) |
| **Log management** | Complex (gaps allowed) | Simple (no gaps, sequential) |
| **Membership changes** | Complex | Built-in (joint consensus) |
| **Implementations** | Google Chubby, Apache Zab | etcd, CockroachDB, TiKV |
| **Production use** | Proven at Google scale | Widely adopted |

> **Recommendation for study:** Understand Raft thoroughly. Know that Paxos exists and provides the theoretical foundation. In interviews, explain Raft — it's more practical and interviewers appreciate clarity.

---

## Phase 5: Consensus in Practice

### ZooKeeper (Zab Protocol)

ZooKeeper uses the **Zab (ZooKeeper Atomic Broadcast)** protocol — similar to Paxos.

```
ZooKeeper ensemble (3 or 5 nodes):
  1 Leader + 2 or 4 Followers

Leader:
  - Handles all write requests
  - Broadcasts writes to followers using Zab
  - Write committed when majority acknowledges

Followers:
  - Handle read requests locally
  - Forward write requests to leader
  - Participate in leader election on failure
```

**Use cases:**
- **Configuration management** — store and distribute config across all services
- **Leader election** — services use ZooKeeper to elect a leader
- **Distributed locks** — ensure only one process holds a lock
- **Service discovery** — register and discover services

```java
// Leader election using Apache Curator (ZooKeeper client)
LeaderSelector selector = new LeaderSelector(client, "/leader-election", 
    new LeaderSelectorListenerAdapter() {
        @Override
        public void takeLeadership(CuratorFramework client) {
            log.info("I am now the leader!");
            // Do leader work...
            // Method blocks while this node is leader
            // Returns when leadership is lost
        }
    });
selector.autoRequeue();  // Re-enter election when leadership lost
selector.start();
```

---

### etcd (Raft)

etcd is a distributed key-value store that uses **Raft** for consensus. It's the backbone of Kubernetes.

```
etcd cluster (3 or 5 nodes):
  Raft consensus for every write
  
Kubernetes uses etcd to store:
  - Cluster state (pods, services, deployments)
  - Configuration (ConfigMaps, Secrets)
  - Leader election for controllers

# Write to etcd
etcdctl put /config/rate-limit "100"

# Read from etcd
etcdctl get /config/rate-limit
```

**etcd vs ZooKeeper:**

| Aspect | etcd | ZooKeeper |
|---|---|---|
| **Algorithm** | Raft | Zab (Paxos-like) |
| **Language** | Go | Java |
| **API** | gRPC + HTTP/JSON | Custom TCP protocol |
| **Watch** | Efficient watch on key ranges | Watch on znodes |
| **Kubernetes** | Default store | Historically used (now etcd) |
| **Complexity** | Simpler to operate | More complex |

---

### Redis Sentinel (Leader Election for Redis)

```
Redis Sentinel monitors Redis master-replica setup:

Normal:
  [Master] ←→ [Replica 1] ←→ [Replica 2]
  [Sentinel 1] [Sentinel 2] [Sentinel 3] (monitoring)

Master fails:
  Sentinels detect failure (quorum vote)
  Sentinels elect a new master from replicas
  Sentinels redirect clients to new master

Sentinel uses Raft-like voting:
  1. Sentinel 1 detects master down → starts election
  2. Sentinel 1 asks Sentinel 2, 3: "Master is down, I want to do failover. Agree?"
  3. Majority agrees → Sentinel 1 promotes Replica 1 to master
```

---

## Phase 6: Distributed Locks

A practical application of consensus — ensuring **only one process** can perform an operation at a time.

### Why Distributed Locks?

```
Without lock:
  Worker A: reads shipment SHP-001, status = "PENDING"
  Worker B: reads shipment SHP-001, status = "PENDING"
  Worker A: processes SHP-001, sets status = "PROCESSING"
  Worker B: processes SHP-001, sets status = "PROCESSING"  ← DUPLICATE PROCESSING!

With lock:
  Worker A: acquires lock "shipment:SHP-001" → SUCCESS
  Worker B: tries to acquire lock "shipment:SHP-001" → FAILED (already held)
  Worker A: processes SHP-001 → releases lock
  Worker B: acquires lock → processes next shipment
```

### Redis Distributed Lock (Redlock)

```java
// Using Redisson (Redis distributed lock library)
RLock lock = redissonClient.getLock("shipment-processing:" + shipmentId);

try {
    // Try to acquire lock, wait up to 10 sec, auto-release after 30 sec
    if (lock.tryLock(10, 30, TimeUnit.SECONDS)) {
        try {
            processShipment(shipmentId);
        } finally {
            lock.unlock();
        }
    } else {
        log.info("Could not acquire lock for {}, another worker is processing it", shipmentId);
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

### Lock Safety Requirements

| Requirement | Meaning |
|---|---|
| **Mutual exclusion** | At most one client holds the lock at any time |
| **Deadlock freedom** | Even if a client crashes while holding the lock, it eventually releases (TTL) |
| **Fault tolerance** | Lock works even if some Redis/ZooKeeper nodes fail |

### ⚠️ The Fencing Token Problem

```
Worker A acquires lock → starts processing → GC pause (30 sec) → lock expires
Worker B acquires lock → starts processing
Worker A resumes → STILL THINKS it holds the lock → writes stale data!
```

**Solution: Fencing tokens**
```
Worker A acquires lock, gets fencing token #33
Worker B acquires lock (after A's expired), gets fencing token #34

Worker A tries to write with token #33 → Storage rejects (seen token #34 already)
Worker B writes with token #34 → Storage accepts ✅
```

---

## Phase 7: Cluster Size — Why Odd Numbers?

### The Quorum Requirement

Consensus requires a **majority** (quorum) to agree. The quorum must be > N/2.

| Cluster Size (N) | Quorum | Can Tolerate Failures |
|---|---|---|
| 1 | 1 | 0 (no fault tolerance) |
| 2 | 2 | 0 (both must agree — useless!) |
| **3** | 2 | **1 failure** |
| 4 | 3 | 1 failure (same as 3 — wasted node!) |
| **5** | 3 | **2 failures** |
| 6 | 4 | 2 failures (same as 5 — wasted node!) |
| **7** | 4 | **3 failures** |

**Notice:** Even-numbered clusters tolerate the same failures as N-1. The extra node adds cost but no benefit.

> **Rule: Always use odd-numbered clusters** — 3 for most systems, 5 for critical systems, 7 for extreme HA.

### Common Production Setups

| System | Typical Cluster Size | Why |
|---|---|---|
| **etcd (Kubernetes)** | 3 or 5 | 5 for production, tolerates 2 failures |
| **ZooKeeper** | 3 or 5 | Odd numbers required |
| **Kafka** (controller quorum) | 3 | KRaft mode controller nodes |
| **MongoDB** (replica set) | 3 | 1 primary + 2 secondaries |
| **Redis Sentinel** | 3 | Quorum for failover decisions |

---

## Phase 8: When Consensus is an Overhead

### Scenario 1: Single-Node Systems

If you're running one database instance, there's nothing to agree on. Adding consensus adds latency for no benefit.

---

### Scenario 2: Eventual Consistency is Acceptable

If your system can tolerate temporary disagreement (AP systems from [[CAP Theorem]]), you don't need consensus. Use gossip protocols or CRDTs instead.

```
Social media "like" count:
  Node A: 1,005 likes
  Node B: 1,003 likes
  Eventually converges to 1,005 — no consensus needed
```

---

### Scenario 3: High Write Throughput

Consensus requires multiple network round-trips per write. For millions of writes/sec, the overhead is significant.

```
Raft consensus per write:
  Client → Leader → send to followers → wait for majority ACK → respond
  Latency: ~5-20ms per write (network round trips)

Direct write (no consensus):
  Client → Server → write to disk → respond
  Latency: ~1ms
```

For high-throughput event streams, use Kafka (which handles its own internal consensus) rather than building consensus into your application.

---

### Scenario 4: Stateless Services

If your services are stateless (all state in the database), there's no need for leader election among service instances. Use a load balancer instead.

---

## Phase 9: Real-World Case Studies

### Case 1: Logistics — Scheduled Job Leader Election

**Problem:** A "generate daily invoices" job runs across 5 service instances. If all 5 run it simultaneously, customers get 5 invoices.

**Solution:**
```java
// Using Spring Integration with Redis lock
@Scheduled(cron = "0 0 6 * * *")  // 6 AM daily
public void generateDailyInvoices() {
    RLock lock = redissonClient.getLock("daily-invoice-generation");
    
    if (lock.tryLock(0, 60, TimeUnit.MINUTES)) {
        try {
            log.info("This instance won the lock. Generating invoices...");
            invoiceService.generateAll();
        } finally {
            lock.unlock();
        }
    } else {
        log.info("Another instance is generating invoices. Skipping.");
    }
}
```

**Result:** Exactly one instance processes invoices. Others skip gracefully.

---

### Case 2: Logistics — Database Failover

**Problem:** PostgreSQL primary crashes. The platform must failover to a replica within seconds.

**Solution:**
```
Patroni (PostgreSQL HA) uses etcd for consensus:

Normal:
  Primary (Node A) → registered as leader in etcd
  Replica (Node B) → follows Node A
  Replica (Node C) → follows Node A

Failure:
  Node A crashes → Patroni detects (missed heartbeat)
  Patroni on Node B and C → consult etcd → elect new leader
  etcd consensus: Node B has most recent data → Node B promoted to primary
  Application connection pool automatically reconnects to Node B

Failover time: ~10-30 seconds
```

---

## Key Takeaways

1. **Consensus = getting distributed nodes to agree** on a value, leader, or sequence of operations
2. **Raft is the go-to algorithm** — understand leader election, log replication, and commit rules
3. **Paxos is the theoretical foundation** — know it exists, prefer Raft for practical understanding
4. **Always use odd-numbered clusters** — 3 for standard, 5 for critical systems
5. **ZooKeeper (Zab) and etcd (Raft)** are the two main consensus infrastructure tools
6. **Distributed locks** ensure mutual exclusion — use Redis (Redisson) or ZooKeeper
7. **Fencing tokens** prevent stale writes from crashed lock holders
8. **Split-brain is the #1 danger** — consensus prevents two nodes from both thinking they're leader
9. **Don't use consensus when you don't need it** — stateless services, eventual consistency, and single-node systems don't benefit

---

**See also:** [[CAP Theorem]] (consistency tradeoffs), [[Sharding and Replication]] (replication + failover), [[Microservices Patterns]] (distributed coordination)
