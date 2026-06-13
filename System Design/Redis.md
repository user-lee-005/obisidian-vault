# Redis

> *"Redis is not just a cache — it's a data structure server that happens to be extremely fast. Every time you treat it as only a cache, you leave 80% of its power on the table."* — Common SDE-2 Interview Insight

---

## Phase 1: What Redis Is and Why It Exists

### What Redis Is

Redis (**Re**mote **Di**ctionary **S**erver) is an **open-source, in-memory data structure store** that functions as a:
- **Cache** — Serve frequently accessed data from RAM
- **Database** — Persist data with optional durability guarantees
- **Message Broker** — Pub/Sub and Stream-based messaging
- **Session Store** — Fast session management across distributed services
- **Rate Limiter** — Atomic counters for request throttling
- **Coordination Primitive** — Distributed locks, leader election

Redis was created by **Salvatore Sanfilippo (antirez)** in 2009 to solve a real-time web log analyzer problem: the analyzer needed to accumulate and query data faster than any relational database could handle.

### Why Redis Exists — The Gap It Fills

```
Memcached:    Fast (in-memory) + Simple (strings only) + No persistence + No replication
PostgreSQL:   Rich data + Persistence + Transactions + Slow for hot-path key-value reads
Redis:        Fast (in-memory) + Rich data structures + Optional persistence + Replication
```

**Redis fills the gap between Memcached and a relational database.**

### Performance Numbers You Must Know

| Operation | Latency | Throughput (single node) |
|---|---|---|
| Simple GET/SET | ~100 microseconds | ~100,000 ops/sec |
| Complex data structures | ~200–500 microseconds | ~50,000 ops/sec |
| PostgreSQL equivalent | 1–50 milliseconds | ~1,000–5,000 ops/sec |

> **Key Insight:** Redis is typically **100–500x faster** than a relational database for key-value reads. This is the core value proposition, and the interviewer will expect you to quantify it.

---

## Phase 2: Memory vs Disk — The Fundamental Architecture Decision

### Why Redis Is In-Memory

Redis stores all data in **RAM**. This is a deliberate design choice with profound implications.

```
RAM access:   ~100 nanoseconds
SSD access:   ~100 microseconds    (1,000x slower than RAM)
HDD access:   ~10 milliseconds     (100,000x slower than RAM)
```

**What this means in practice:**
```
SELECT * FROM carriers WHERE id = 'MAERSK';
  → PostgreSQL: page lookup → buffer pool check → possible disk I/O = 1–10ms

GET carrier:MAERSK
  → Redis: hash table lookup in RAM = ~100 microseconds
```

### The RAM Cost Tradeoff

| Redis Use Case | RAM Needed |
|---|---|
| 1 million simple strings (~100 bytes each) | ~100MB |
| 1 million JSON objects (~1KB each) | ~1GB |
| 10 million sessions (~500 bytes each) | ~5GB |
| Typical production Redis instance | 4GB – 64GB |

RAM is more expensive than disk. Redis is not a replacement for your primary database — it is a **hot-path acceleration layer**.

> ⚠️ **Common Mistake:** Treating Redis as a general-purpose database for all data. Redis excels at high-frequency, hot-path access. Rarely-accessed historical data belongs in PostgreSQL, not Redis.

---

## Phase 3: Event Loop Architecture

### Single-Threaded Command Execution

Redis processes all client commands **on a single thread**. This is the most important architectural decision to understand for interviews.

```
       ┌──────────────────────────────────────────┐
       │            Redis Main Thread              │
       │                                           │
       │  1. Accept new client connections         │
       │  2. Read incoming commands from sockets   │
       │  3. Execute commands (one at a time)      │
       │  4. Write responses back to sockets       │
       │  5. Handle timers (key expiry, AOF flush) │
       └──────────────────────────────────────────┘
                  ↑ Everything serialized here ↑
```

**Why single-threaded?**
1. **No locks needed** — Only one command executes at a time. No mutex contention on data structures.
2. **No context switching** — CPU stays focused on one task.
3. **Predictable latency** — Commands don't wait for locks held by other threads.
4. **Simple implementation** — Redis's source is dramatically simpler than a multi-threaded equivalent.

> ⚠️ **What Junior Engineers Usually Miss:** Single-threaded command execution means Redis operations are serialized. It does NOT mean your application-level business logic spanning multiple Redis commands is safe from concurrency bugs. This is covered in depth in Phase 14.

---

### The Event Loop (ae — Async Events)

Redis uses an **event-driven, non-blocking I/O** architecture built on its own event loop library called **ae**.

```
                           Redis ae Event Loop
                   ┌──────────────────────────────────┐
                   │                                   │
  Client sockets → │         aeMain()                 │
  (thousands)      │           │                       │
                   │     aeProcessEvents()             │
                   │           │                       │
                   │     ┌─────┴────────┐              │
                   │     │              │              │
                   │  I/O events    Time events        │
                   │  (read/write   (key expiry,       │
                   │   sockets)      AOF flush)        │
                   │                                   │
                   │     └──── loop forever ────┘      │
                   └──────────────────────────────────┘
```

---

### epoll — Kernel-Level Socket Multiplexing

The event loop uses **epoll** (Linux) or **kqueue** (BSD/macOS) to monitor thousands of client sockets simultaneously — without creating a thread per connection.

**Without epoll (thread-per-connection):** 10,000 clients = 10,000 threads = massive context switching overhead

**With epoll:** 10,000 clients = 1 thread watching all sockets via kernel notification

```
                        Kernel epoll
     Socket 1 (data ready)   │
     Socket 2 (idle)         │   epoll_wait() blocks until
     Socket 3 (data ready)   │   at least one socket is ready
     Socket 4 (idle)         │
     ...                     ↓
     Socket N (data ready)   Returns: [Socket 1, Socket 3, Socket N]
                             Redis processes ONLY those — no wasted CPU
```

Redis uses **edge-triggered** epoll (EPOLLET): notified once when data arrives, not repeatedly while data is available. Higher performance, but requires reading all available data in each notification.

---

### Redis 6+ I/O Threads (Threaded I/O)

Redis 6 introduced **multi-threaded I/O** — but ONLY for reading from and writing to sockets, NOT for executing commands.

```
                      Redis 6+ Architecture
        ┌────────────────────────────────────────────────┐
        │                                                │
Clients │  I/O Thread 1 ─── parse request ─────────┐   │
(many)  │  I/O Thread 2 ─── parse request ─────────┤   │
───────→│  I/O Thread 3 ─── parse request ─────────┤   │
        │                                           ↓   │
        │              Main Thread (single) ── execute  │
        │                                           │   │
        │  I/O Thread 1 ◄── write response ────────┤   │
        │  I/O Thread 2 ◄── write response ────────┘   │
        └────────────────────────────────────────────────┘
```

Command **execution** remains single-threaded. Configure with `io-threads 4` and `io-threads-do-reads yes` in `redis.conf`. Helps when the bottleneck is socket I/O bandwidth, not command processing speed.

> **What SDE-2 Candidates Should Know:** Threaded I/O helps for high-bandwidth workloads (many clients, large values). It doesn't change the single-threaded execution model. Most production bottlenecks are not I/O-bound, so the gain is often modest.

---

### Redis Internal Request Lifecycle

```
Client App                         Redis Server
    │                                    │
    │──── TCP Connect ──────────────────→│ epoll registers new socket fd
    │                                    │
    │──── "*3\r\n$3\r\nSET\r\n..."  ───→│ epoll: socket fd is readable
    │                                    │ Read bytes from socket buffer
    │                                    │ Parse RESP protocol → [SET, foo, bar]
    │                                    │ Execute: dict_set(db, "foo", "bar")
    │                                    │ Encode response: "+OK\r\n"
    │←─── "+OK\r\n" ────────────────────│ Write to socket buffer
    │                                    │ epoll: socket fd writable, flush
```

**RESP Protocol (Redis Serialization Protocol):**
```
Client sends:  *3\r\n$3\r\nSET\r\n$3\r\nfoo\r\n$3\r\nbar\r\n
               ↑ Array of 3 items

Responses:
  +OK\r\n           → Simple string (success)
  :42\r\n           → Integer
  $6\r\nfoobar\r\n  → Bulk string (length-prefixed)
  -ERR msg\r\n      → Error
  *3\r\n...         → Array (LRANGE result etc.)
```

---

## Phase 4: Data Structures — The Core Value

### Why Data Structures Matter

Every Redis data structure is a **specialized tool**. Using the wrong structure is one of the most common mistakes in interviews and in production.

```
Use Case                        Wrong Choice    Right Choice
─────────────────────────────────────────────────────────────
Count unique page views         List            HyperLogLog
Leaderboard with scores         Hash            Sorted Set
User session object             String (JSON)   Hash (partial updates)
Rate limiting counter           List            String (INCR)
Scheduled job queue             List            Sorted Set (score=timestamp)
Real-time event log             Pub/Sub         Stream
Track if user took an action    String          Set (SISMEMBER)
```

---

### String

**Definition:** A binary-safe sequence of bytes. Can hold text, integers, floats, or serialized binary (JSON, Protobuf). Maximum size: **512MB**.

**Internal Encodings:**
```
OBJECT ENCODING counter  → "int"      integers -9999 to 9999 (shared objects)
OBJECT ENCODING shortstr → "embstr"   strings ≤44 bytes (single allocation)
OBJECT ENCODING longstr  → "raw"      strings >44 bytes (separate allocation)
```

**Key Commands:**
| Command | Complexity | Example |
|---|---|---|
| `SET key value [EX n]` | O(1) | `SET session:u123 "{...}" EX 3600` |
| `GET key` | O(1) | `GET session:u123` |
| `INCR key` | O(1) | `INCR rate:user:123` |
| `INCRBY key delta` | O(1) | `INCRBY inventory:SKU-001 -5` |
| `SETNX key value` | O(1) | `SETNX lock:order:456 "uuid-abc"` |
| `SET key value NX EX n` | O(1) | Atomic lock acquisition |
| `GETDEL key` | O(1) | Atomic read + delete |
| `MGET k1 k2 k3` | O(N) | Batch read |
| `MSET k1 v1 k2 v2` | O(N) | Batch write |

**Real-World Example:**
```java
// Atomic rate limiting: max 100 requests per minute per user
String key = "rate:user:" + userId + ":" + currentMinute; // e.g. rate:user:123:2026-06-13T10:30
Long count = redisTemplate.opsForValue().increment(key);
if (count == 1) {
    redisTemplate.expire(key, Duration.ofMinutes(2)); // 2-minute safety window
}
if (count > 100) throw new RateLimitExceededException();
```

**When to use Strings:**
- Counters (INCR is atomic — critical for rate limiting, inventory, likes)
- Sessions (store serialized session JSON)
- Distributed locks (SET NX EX for atomic lock acquisition)
- Feature flags (`SET feature:dark-mode enabled`)
- Simple key-value caching

---

### Hash

**Definition:** A map of field-value pairs stored under a single Redis key. Analogous to a Java `HashMap` nested inside a Redis key.

**Internal Encodings:**
- Small hashes (≤128 fields, values ≤64 bytes): **listpack** — compact linear array, O(N) lookup but small N
- Large hashes: **hashtable** — O(1) average lookup

**Key Commands:**
| Command | Complexity | Example |
|---|---|---|
| `HSET key field value` | O(1) | `HSET user:123 name "Venki"` |
| `HGET key field` | O(1) | `HGET user:123 name` |
| `HMGET key f1 f2` | O(N) | Get multiple fields |
| `HGETALL key` | O(N) | All fields — avoid on large hashes |
| `HINCRBY key field delta` | O(1) | `HINCRBY cart:u123 total 150` |
| `HDEL key field` | O(1) | Delete a field |
| `HEXISTS key field` | O(1) | Field existence check |
| `HLEN key` | O(1) | Number of fields |

**Real-World Example:**
```java
// User session as Hash — enables partial updates without full re-serialize
String sessionKey = "session:" + sessionId;
redisTemplate.opsForHash().put(sessionKey, "userId", userId);
redisTemplate.opsForHash().put(sessionKey, "role", role);
redisTemplate.opsForHash().put(sessionKey, "lastActive", Instant.now().toString());
redisTemplate.expire(sessionKey, Duration.ofMinutes(30));

// Update only lastActive (no full re-serialize needed)
redisTemplate.opsForHash().put(sessionKey, "lastActive", Instant.now().toString());
```

**Hash vs String for Objects:**
| | Hash | String (JSON) |
|---|---|---|
| **Update single field** | O(1) — HSET | O(N) — full re-serialize + SET |
| **Read all fields** | O(N) — HGETALL | O(1) — GET |
| **Partial read** | O(1) — HMGET | Deserialize everything |
| **When to use** | Objects with frequent partial field updates | Objects always read/written as a whole |

---

### List

**Definition:** An ordered doubly-linked list of strings. Supports O(1) push/pop from both ends. Maximum elements: 2^32 - 1.

**Internal Encodings:**
- Small lists (≤128 elements, values ≤64 bytes): **listpack**
- Large lists: **quicklist** — a linked list of listpack nodes

```
List "queue:orders":
  [HEAD] ← "order:001" ↔ "order:002" ↔ "order:003" → [TAIL]
   LPUSH adds here                           RPUSH adds here
   LPOP removes here                         RPOP removes here
```

**Key Commands:**
| Command | Complexity | Use |
|---|---|---|
| `LPUSH / RPUSH key value` | O(1) | Add to head / tail |
| `LPOP / RPOP key` | O(1) | Remove from head / tail |
| `BLPOP key timeout` | O(1) | Blocking pop (waits up to timeout seconds) |
| `LRANGE key start stop` | O(N) | Get a range of elements |
| `LLEN key` | O(1) | List length |
| `LTRIM key start stop` | O(N) | Keep only elements in range |
| `LINDEX key index` | O(N) | Get element at index — avoid on large lists |

**Real-World Example:**
```java
// Simple task queue
// Producer: add job to tail
redisTemplate.opsForList().rightPush("jobs:email", jobJson);

// Consumer: blocking pop from head (waits up to 30 seconds for a job)
String job = redisTemplate.opsForList().leftPop("jobs:email", Duration.ofSeconds(30));

// Recent activity feed — keep only last 100 items
String feedKey = "activity:user:" + userId;
redisTemplate.opsForList().leftPush(feedKey, activityJson);
redisTemplate.opsForList().trim(feedKey, 0, 99); // Atomically keep latest 100
```

> ⚠️ **Warning:** `LINDEX` and `LREM` are O(N). Never use them inside a hot path on large lists.

---

### Set

**Definition:** An unordered collection of unique strings. Membership check is O(1). Supports set algebra (union, intersection, difference).

**Internal Encodings:**
- Small integer-only sets: **intset** — sorted array of integers, O(log N)
- Small mixed sets (≤128 elements, values ≤64 bytes): **listpack**
- Large sets: **hashtable** — O(1)

**Key Commands:**
| Command | Complexity | Use |
|---|---|---|
| `SADD key member` | O(1) | Add member |
| `SREM key member` | O(1) | Remove member |
| `SISMEMBER key member` | O(1) | Membership check |
| `SCARD key` | O(1) | Cardinality (count) |
| `SMEMBERS key` | O(N) | Get all — avoid on large sets |
| `SUNION key1 key2` | O(N+M) | Union |
| `SINTER key1 key2` | O(N*M) | Intersection |
| `SDIFF key1 key2` | O(N) | Difference |
| `SRANDMEMBER key count` | O(count) | Random sampling |

**Real-World Example:**
```java
// Track unique users who viewed a product today (per-day key, auto-expires)
String key = "views:product:" + productId + ":" + LocalDate.now();
redisTemplate.opsForSet().add(key, userId);
redisTemplate.expire(key, Duration.ofDays(1));
Long uniqueViewers = redisTemplate.opsForSet().size(key);

// Atomic coupon: has this user already redeemed it?
Boolean alreadyRedeemed = redisTemplate.opsForSet().isMember("coupon:SALE20:redeemed", userId);
if (!alreadyRedeemed) {
    redisTemplate.opsForSet().add("coupon:SALE20:redeemed", userId);
    // apply discount...
}

// Products tagged with both "electronics" AND "on-sale"
Set<String> results = redisTemplate.opsForSet().intersect("tag:electronics", "tag:on-sale");
```

---

### Sorted Set (ZSet)

**Definition:** Like a Set, but each member has an associated **floating-point score**. Members are automatically sorted by score in ascending order. One of Redis's most powerful and distinctive data structures.

**Internal Encodings:**
- Small sorted sets (≤128 elements, values ≤64 bytes): **listpack**
- Large sorted sets: **skiplist + hashtable**
  - **Skiplist:** O(log N) for ordered operations (rank, range, score queries)
  - **Hashtable:** O(1) for score lookup by member name

```
Sorted Set "leaderboard:global":
  Score 9950.0  →  "user:alice"
  Score 8820.0  →  "user:bob"
  Score 7100.0  →  "user:carol"
  Score 5000.0  →  "user:dave"
       ↑ ascending order (use ZREV* commands for descending / top-N queries)
```

**Key Commands:**
| Command | Complexity | Use |
|---|---|---|
| `ZADD key score member` | O(log N) | Add / update member |
| `ZSCORE key member` | O(1) | Get score of member |
| `ZRANK key member` | O(log N) | 0-indexed rank (ascending) |
| `ZREVRANK key member` | O(log N) | 0-indexed rank (descending) |
| `ZRANGE key start stop [WITHSCORES]` | O(log N + M) | Range by rank |
| `ZRANGEBYSCORE key min max` | O(log N + M) | Range by score |
| `ZINCRBY key delta member` | O(log N) | Increment score |
| `ZCARD key` | O(1) | Cardinality |
| `ZPOPMIN / ZPOPMAX key` | O(log N) | Remove and return min/max |

**Real-World Examples:**
```java
// Leaderboard: increment score on each game action
redisTemplate.opsForZSet().incrementScore("leaderboard:game:123", userId, pointsEarned);

// Top 10 players (descending by score)
Set<ZSetOperations.TypedTuple<String>> top10 =
    redisTemplate.opsForZSet().reverseRangeWithScores("leaderboard:game:123", 0, 9);

// Delayed job queue: score = Unix timestamp for scheduled execution
double executeAt = Instant.now().plusSeconds(300).getEpochSecond();
redisTemplate.opsForZSet().add("jobs:delayed", jobId, executeAt);

// Scheduler: get all jobs ready to run (score <= current time)
Set<String> readyJobs = redisTemplate.opsForZSet()
    .rangeByScore("jobs:delayed", 0, Instant.now().getEpochSecond());
```

> **What SDE-2 Candidates Should Know:** Sorted Sets are the secret weapon for leaderboards, delayed job queues (score = timestamp), sliding-window rate limiting (score = request timestamp), and priority queues. Knowing when to reach for ZSet separates SDE-2 answers from junior answers.

> **Interviewer's Perspective:** "What's the time complexity of ZADD?" → O(log N) because the underlying structure is a **skiplist**. Follow up: "Why skiplist over a balanced BST?" → Simpler implementation, easier range queries, similar O(log N) guarantees. This shows you understand data structures, not just memorized answers.

---

### Bitmap

**Definition:** Not a separate Redis type. Strings used as bit arrays. Each offset stores 1 bit. Enables extremely memory-efficient boolean tracking.

**Key Commands:**
| Command | Complexity | Use |
|---|---|---|
| `SETBIT key offset 0/1` | O(1) | Set bit at position |
| `GETBIT key offset` | O(1) | Get bit at position |
| `BITCOUNT key [start end]` | O(N) | Count set bits |
| `BITOP AND/OR/XOR dest k1 k2` | O(N) | Bitwise operations between keys |

**Real-World Example:**
```java
// Daily Active Users: bit position = userId (integer)
// 100 million users → 100M bits = 12.5MB per day
String dauKey = "dau:" + LocalDate.now(); // "dau:2026-06-13"
redisTemplate.opsForValue().setBit(dauKey, userId, true);

// Count DAUs today
Long dau = redisTemplate.execute((RedisCallback<Long>) conn ->
    conn.bitCount(dauKey.getBytes()));

// Users active on BOTH Monday AND Tuesday
redisTemplate.execute((RedisCallback<Object>) conn -> {
    conn.bitOp(BitOperation.AND, "dau:mon-and-tue".getBytes(),
               "dau:2026-06-09".getBytes(), "dau:2026-06-10".getBytes());
    return null;
});
```

**Memory comparison:**
```
1M users tracked as Set:    ~8MB   (8 bytes overhead per integer entry)
1M users tracked as Bitmap: 125KB  (1 bit per userId)
→ Bitmap is 64x more memory-efficient for dense userId spaces
```

---

### HyperLogLog

**Definition:** A probabilistic data structure for **approximate cardinality estimation**. Answers "how many unique elements have been added?" with ~0.81% standard error, using only **12KB of memory** regardless of the actual cardinality.

**Key Commands:**
| Command | Complexity | Use |
|---|---|---|
| `PFADD key element [element ...]` | O(1) amortized | Add element(s) |
| `PFCOUNT key [key ...]` | O(1) | Estimate unique count |
| `PFMERGE dest k1 k2 ...` | O(N) | Merge multiple HLLs |

**Real-World Example:**
```java
// Count unique search queries today — could be billions of events
String hllKey = "unique:searches:" + LocalDate.now();
redisTemplate.opsForHyperLogLog().add(hllKey, searchQuery);

// Approximate count (±0.81% error — acceptable for analytics)
Long approximateCount = redisTemplate.opsForHyperLogLog().size(hllKey);

// Weekly unique: merge 7 daily HLLs
redisTemplate.opsForHyperLogLog().union("unique:searches:week:2026-W24",
    "unique:searches:2026-06-09", "unique:searches:2026-06-10", /* ... */);
```

**When to use:** Page views, unique visitors, unique search terms, distinct API callers — when cardinality is large and exact count is not required. Never use HyperLogLog if you need to retrieve or check individual elements.

---

### Stream

**Definition:** An append-only log of messages, each with a unique auto-generated ID and associated field-value pairs. Redis's native answer to Kafka-style event streaming.

Streams are covered in depth in Phase 10. Quick reference:

```bash
XADD shipment-events * event DEPARTED shipment SHP-001   # Append message
XREAD COUNT 10 STREAMS shipment-events 0-0                # Read from beginning
XGROUP CREATE shipment-events workers $ MKSTREAM          # Create consumer group
XREADGROUP GROUP workers pod-1 COUNT 5 STREAMS shipment-events >  # Read as group
XACK shipment-events workers 1686653000-0                 # Acknowledge
```

---

### Data Structure Time Complexity Summary

| Operation | String | Hash | List | Set | Sorted Set |
|---|---|---|---|---|---|
| **Add element** | SET O(1) | HSET O(1) | LPUSH O(1) | SADD O(1) | ZADD O(log N) |
| **Get by key/field** | GET O(1) | HGET O(1) | LINDEX O(N) | — | ZSCORE O(1) |
| **Delete element** | DEL O(1) | HDEL O(1) | LREM O(N) | SREM O(1) | ZREM O(log N) |
| **Membership check** | — | HEXISTS O(1) | — | SISMEMBER O(1) | ZSCORE O(1) |
| **Count** | STRLEN O(1) | HLEN O(1) | LLEN O(1) | SCARD O(1) | ZCARD O(1) |
| **Ordered range** | — | — | LRANGE O(N) | — | ZRANGE O(log N+M) |
| **Rank** | — | — | — | — | ZRANK O(log N) |

---

## Phase 5: Expiration and TTL

### Setting TTL

```bash
SET carrier:MAERSK "{...}" EX 3600      # Set with 3600 second TTL at creation
SET carrier:HAPAG "{...}" PX 3600000    # Set with TTL in milliseconds
EXPIRE carrier:MSC 7200                 # Apply TTL to existing key (seconds)
PEXPIRE carrier:MSC 7200000             # Apply TTL in milliseconds
EXPIREAT carrier:CMA 1735689600         # Expire at specific Unix timestamp
TTL carrier:MAERSK                      # Remaining TTL in seconds (-1 = no TTL, -2 = gone)
PERSIST carrier:MAERSK                  # Remove TTL — make key permanent
```

---

### Passive Expiration (Lazy)

Redis does NOT scan all keys on a timer to proactively delete expired ones. Expiration is checked **lazily** — only when a key is accessed.

```
Client:  GET carrier:MAERSK
Redis:   1. Does key exist? → Yes
         2. Does it have a TTL? → Yes
         3. Is TTL expired? → Yes
         4. Delete the key
         5. Return: (nil) to client
```

**Implication:** A key past its TTL consumes memory until it is accessed OR until the active expiration cycle catches it. In a keyspace with many never-re-accessed keys, memory can grow silently.

---

### Active Expiration (Periodic Sampling)

To prevent memory bloat from expired-but-never-accessed keys, Redis runs an **active expiration cycle** every 100ms.

**Algorithm:**
```
Every 100ms:
  1. Sample 20 random keys from the "keys with TTL" set
  2. Delete all expired keys found in the sample
  3. If > 25% of sampled keys were expired → run again immediately
  4. Otherwise → wait for next cycle

CPU budget: Active expiration is capped at 25% of event loop time
            to avoid starving client request processing
```

**Why probabilistic?** Scanning all keys every 100ms would block the event loop for seconds on a large keyspace. Random sampling is statistically sufficient.

> **What Senior Engineers Know:** Monitor expiry health with `INFO stats`:
```bash
expired_keys:1234567    # Keys deleted by active + passive expiration combined
evicted_keys:5678       # Keys deleted due to maxmemory policy (different from expiration!)
```
A large `expired_keys` rate with growing `used_memory` means active expiration is losing the race — consider shortening TTLs or reducing keyspace size.

---

## Phase 6: Persistence

### The Persistence Dilemma

Redis is in-memory. Without persistence, all data is lost on restart.

```
No persistence:
  Redis crashes at T=0 → all data lost → cold cache → thundering herd on DB restart

With persistence:
  Redis crashes at T=0 → restarts at T=60 → loads data from disk
  → Applications resume with warm (or warm-ish) cache
```

Redis offers two persistence mechanisms with different durability/performance tradeoffs.

---

### RDB — Redis Database (Snapshots)

**What it is:** A full **point-in-time binary snapshot** of all Redis data written to `dump.rdb`.

**How it works:**
```
Normal operation:   Redis writes only to memory (full speed)

Snapshot triggered:
  Redis forks a child process
       │
  ┌────┴───────────────────────────────────┐
  │ Parent process  │  Child process        │
  │ (keeps serving  │  (writes RDB to disk) │
  │  clients)       │                       │
  └─────────────────┴───────────────────────┘

Copy-on-Write (COW):
  Parent and child initially share the same memory pages
  When parent writes new data → OS gives parent a new memory page
  Child still sees the snapshot state at fork time
  → Zero-blocking snapshot at the cost of RAM
```

**Trigger configuration:**
```bash
save 900 1       # Snapshot if ≥1 key changed in 15 minutes
save 300 10      # Snapshot if ≥10 keys changed in 5 minutes
save 60 10000    # Snapshot if ≥10,000 keys changed in 1 minute

BGSAVE           # Manual non-blocking snapshot (returns immediately)
SAVE             # Manual BLOCKING snapshot — never use in production
```

**RDB tradeoffs:**
| Pros | Cons |
|---|---|
| Compact single binary file — easy to backup | Data loss = time since last snapshot (minutes) |
| Fast restart — load one file vs replay log | Fork can spike memory (COW pressure on write-heavy load) |
| Minimal performance impact during normal operation | Not suitable for near-zero data loss requirements |
| Great for disaster recovery / periodic backups | — |

---

### AOF — Append-Only File

**What it is:** Every write command is **appended to a log file** (`appendonly.aof`). On restart, Redis replays the log to reconstruct the dataset.

**How it works:**
```
Client: SET foo bar
Redis:  1. Execute command (update in-memory data)
        2. Append "SET foo bar\r\n" to AOF write buffer
        3. Based on fsync policy, flush buffer → disk
```

**fsync Policies:**
| Policy | Behavior | Data Loss Risk | Performance Impact |
|---|---|---|---|
| `always` | fsync after every write command | At most 1 command | Slowest (~1-5ms per write) |
| `everysec` | Background fsync every second | Up to 1 second | Low (background thread) |
| `no` | Let OS decide (~30s) | Up to 30 seconds | Fastest |

**Recommendation:** `everysec` — best balance of durability and performance for production.

**AOF Rewriting** (compaction):
Over time the AOF grows large (1,000 INCR commands → replaced by one SET command). Redis rewrites it:
```
BGREWRITEAOF:
  1. Fork child process
  2. Child writes minimal commands to reconstruct current state
  3. Parent continues appending new commands to old AOF + in-memory buffer
  4. Child finishes → atomic rename to new compact AOF
  5. Buffer written to new AOF → fully caught up
```

**AOF tradeoffs:**
| Pros | Cons |
|---|---|
| Much lower data loss (≤1 second with everysec) | File larger than RDB |
| Human-readable format — easier to repair | Slower restart (replay log vs load snapshot) |
| Append-only prevents corruption from partial writes | Higher disk I/O during heavy writes |

---

### Persistence Tradeoffs Table

| Aspect | RDB | AOF (everysec) | No Persistence |
|---|---|---|---|
| **Data loss on crash** | Minutes (since last snapshot) | ~1 second | 100% |
| **Restart speed** | Fast (load binary file) | Slow (replay entire log) | Instant |
| **File size** | Compact | Larger | None |
| **Runtime performance** | Minimal impact | Small background I/O | Maximum |
| **Use case** | Periodic backups, tolerable data loss | Near-zero data loss | Pure ephemeral cache |

**Production recommendation — use both:**
```bash
# redis.conf
save 900 1
save 300 10
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes    # Write RDB preamble into AOF for faster startup
```

> **What Senior Engineers Know:** With both enabled, Redis uses the AOF on restart (more complete). `aof-use-rdb-preamble yes` embeds an RDB snapshot at the start of the AOF file, dramatically reducing replay time. Monitor `rdb_last_cow_size` in `INFO persistence` — under write-heavy load, COW memory pressure can spike RAM to 2x dataset size.

---

## Phase 7: Replication

### Master-Replica Architecture

Redis supports **asynchronous replication** — one master node accepts all writes, one or more replicas receive a copy of all writes.

```
                  ┌──────────────────────┐
  Clients (W) ──→ │     Redis Master      │
                  │  (accepts all writes) │
                  └──────────┬───────────┘
                             │
              Async replication stream
                  ┌──────────┴──────────┐
                  ↓                     ↓
         ┌──────────────┐     ┌──────────────┐
         │  Replica 1   │     │  Replica 2   │
         │  (read-only) │     │  (read-only) │
         └──────────────┘     └──────────────┘
  Clients (R) ──→ ↑               ↑ ←── Clients (R)
```

**Initial Replication Sequence:**
```
1. Replica connects: REPLICAOF master-host 6379
2. Master: FULLRESYNC <run_id> <offset>
3. Master: BGSAVE snapshot → sends RDB file to replica
4. Replica: loads RDB
5. Master: sends all buffered write commands that occurred during steps 3–4
6. Ongoing: incremental replication stream (every write command)
```

**Read Scaling in Spring Boot:**
```java
@Bean
public LettuceConnectionFactory connectionFactory() {
    RedisStaticMasterReplicaConfiguration config =
        new RedisStaticMasterReplicaConfiguration("master-host", 6379);
    config.addNode("replica-1-host", 6379);
    config.addNode("replica-2-host", 6379);

    LettuceClientConfiguration clientConfig = LettuceClientConfiguration.builder()
        .readFrom(ReadFrom.REPLICA_PREFERRED) // Reads from replica, fallback to master
        .build();

    return new LettuceConnectionFactory(config, clientConfig);
}
```

> ⚠️ **What Junior Engineers Usually Miss:** Reads from replicas can return stale data due to replication lag. Never read from replicas for data where strong consistency is required — post-write reads, inventory checks, session validation.

**Monitor replication lag:**
```bash
INFO replication
# master_repl_offset:1234567
# slave_repl_offset:1234500    ← 67 bytes of lag
```

---

## Phase 8: Redis Sentinel — High Availability

### The Problem Sentinel Solves

With a plain master-replica setup, master failure requires **manual intervention** to promote a replica. Downtime = minutes to hours.

**Sentinel automates failover.**

### How Sentinel Works

```
              ┌────────────────────────────────────────────┐
              │          Sentinel Cluster (quorum)          │
              │  Sentinel-1 ←──→ Sentinel-2 ←──→ Sentinel-3│
              └──────────────────┬─────────────────────────┘
                                 │ periodic PING monitoring
              ┌──────────────────┼──────────────────────────┐
              ↓                  ↓                          ↓
      ┌──────────────┐   ┌──────────────┐         ┌──────────────┐
      │    Master    │   │  Replica 1   │         │  Replica 2   │
      └──────────────┘   └──────────────┘         └──────────────┘
```

**Failover sequence:**
```
T=0s:  Master stops responding to Sentinel PING commands
T=30s: Sentinel-1 marks master SDOWN (subjectively down)
       Asks Sentinel-2 and Sentinel-3: "Do you also see it as down?"
       Quorum (2 of 3) agrees → master declared ODOWN (objectively down)

T=35s: Sentinels elect a leader Sentinel via Raft-based election
       Leader selects best replica to promote:
         1. Replica with highest priority (slave-priority config)
         2. Among equal priority: most up-to-date (highest replication offset)
         3. Tiebreaker: lowest lexicographic runid

T=40s: Leader Sentinel sends: REPLICAOF NO ONE → Replica-1 (promotes it)
       Leader Sentinel sends: REPLICAOF new-master-ip → Replica-2
       All Sentinels update their configuration

T=45s: Applications re-query Sentinel for master address → get Replica-1
       Applications reconnect to new master
```

**Spring Boot with Sentinel:**
```yaml
spring:
  redis:
    sentinel:
      master: mymaster            # Sentinel-monitored master name
      nodes:
        - sentinel-1:26379
        - sentinel-2:26379
        - sentinel-3:26379
      password: redis-password
```

**Quorum:** Minimum Sentinels that must agree master is unreachable. Set to majority: ⌊N/2⌋ + 1. With 3 Sentinels, quorum = 2.

> **What SDE-2 Candidates Should Know:** Sentinel provides high availability for **one master's data**. It does NOT shard or scale horizontally — all data still lives on one master. For horizontal scaling, use Redis Cluster.

---

## Phase 9: Redis Cluster — Horizontal Scaling

### The Problem Cluster Solves

```
Single master limits:
  Memory:     ~64GB practical max
  Throughput: ~100K ops/sec
  Dataset:    Must fit in one machine's RAM

Redis Cluster solution:
  Shard data across N masters
  Memory:     N × 64GB
  Throughput: N × 100K ops/sec
  Automatic replication per shard
```

### Hash Slot Architecture

Redis Cluster divides the keyspace into **16,384 hash slots** (slots 0–16383).

**Key → slot mapping:**
```
slot = CRC16(key) % 16384

Examples:
  CRC16("user:12345") % 16384 = 7638    → Node B
  CRC16("order:99")   % 16384 = 1204    → Node A
  CRC16("session:abc")% 16384 = 12001   → Node C
```

**Slot distribution (3-master, 3-replica cluster):**
```
  ┌──────────────────────────────────────────────────────────────┐
  │  Node A (Master)   Slots 0–5460       ←→  Node D (Replica)  │
  │  Node B (Master)   Slots 5461–10922   ←→  Node E (Replica)  │
  │  Node C (Master)   Slots 10923–16383  ←→  Node F (Replica)  │
  └──────────────────────────────────────────────────────────────┘
```

**Client routing:**
```
Client: GET user:12345
  1. Compute: CRC16("user:12345") % 16384 = 7638
  2. Slot 7638 is on Node B → send to Node B → response

Wrong node (sent to Node A):
  Node A: MOVED 7638 node-b:6379
  Client: redirects to Node B, caches slot→node mapping
```

**Hash tags — force keys to same slot:**
```bash
# Problem: multi-key operations fail if keys span different slots
MGET user:123 order:456   # CROSSSLOT error if on different nodes

# Solution: only the part inside {} is hashed
MGET {user:123}:profile {user:123}:orders   # Both hash on "user:123" → same slot
```

**Resharding (adding/removing nodes):**
```
Before: A [0-5460], B [5461-10922], C [10923-16383]
Add D:  A [0-4095], B [4096-8191], C [8192-12287], D [12288-16383]

During migration:
  Clients get ASK redirects (temporary, unlike MOVED which is permanent)
  Keys in migrating slot are locked briefly during transfer
  O(N keys in migrating slots) — plan for large-dataset resharding time
```

**Cluster limitations:**
| Limitation | Impact |
|---|---|
| Multi-key ops across slots | `MGET`, `MSET`, `SUNION` fail across slots |
| Transactions | `MULTI/EXEC` only works if all keys are in the same slot |
| Lua scripts | All keys accessed must be in the same slot |
| Pub/Sub | Subscriptions are local to the node receiving the SUBSCRIBE |

> **What Senior Engineers Know:** Resharding is O(N) in migrated keys. For very large datasets, this can take hours. Capacity-plan ahead to minimize how often you need to reshard. Use hash tags to co-locate related keys from day one.

---

## Phase 10: Pub/Sub

### How Pub/Sub Works

Redis Pub/Sub is a **fire-and-forget broadcast** system. A publisher sends a message to a channel; all subscribers to that channel receive it instantly.

```
                   Publisher
                       │ PUBLISH shipment-events "SHP-001:DEPARTED"
                       ↓
              ┌────────────────────┐
              │  Redis Pub/Sub     │
              │  Channel:          │
              │  "shipment-events" │
              └──────────┬─────────┘
                         │ broadcast to all subscribers
         ┌───────────────┼────────────────┐
         ↓               ↓                ↓
   Tracking Svc    Notification Svc   Analytics Svc
```

**Commands:**
```bash
SUBSCRIBE shipment-events                      # Subscribe to a channel
PSUBSCRIBE shipment-*                          # Pattern subscribe (glob)
PUBLISH shipment-events "SHP-001:DEPARTED"     # Publish message
UNSUBSCRIBE shipment-events                    # Unsubscribe
PUBSUB CHANNELS shipment-*                     # List active matching channels
PUBSUB NUMSUB shipment-events                  # Subscriber count
```

**Spring Boot Pub/Sub:**
```java
// Publisher
redisTemplate.convertAndSend("cache:invalidate", "carrier:MAERSK");

// Subscriber configuration
@Bean
public RedisMessageListenerContainer listenerContainer(
        RedisConnectionFactory factory, MessageListenerAdapter adapter) {
    RedisMessageListenerContainer container = new RedisMessageListenerContainer();
    container.setConnectionFactory(factory);
    container.addMessageListener(adapter, new PatternTopic("cache:*"));
    return container;
}

@Bean
public MessageListenerAdapter listenerAdapter(CacheInvalidationHandler handler) {
    return new MessageListenerAdapter(handler, "handleMessage");
}
```

**Critical limitations — why Pub/Sub is NOT a message queue:**
| Limitation | Implication |
|---|---|
| **No persistence** | Message sent to channel with no subscribers → lost forever |
| **No acknowledgment** | No way to know if subscriber received the message |
| **No replay** | Late subscriber cannot receive past messages |
| **No consumer groups** | Cannot distribute messages among a worker pool |
| **No backpressure** | Slow subscriber gets overwhelmed with no protection |

> **Appropriate uses for Pub/Sub:** Cache invalidation events (missing one just causes a stale cache — acceptable), live dashboard updates (next update arrives soon), ephemeral chat notifications. For reliable delivery, use Redis Streams or Kafka.

---

## Phase 11: Redis Streams

### What Makes Streams Different From Pub/Sub and Lists

| Feature | List (as queue) | Pub/Sub | Stream |
|---|---|---|---|
| **Persistence** | Yes | No | Yes |
| **Replay history** | No | No | Yes (from any offset) |
| **Consumer groups** | No | No | Yes |
| **Delivery guarantee** | At-most-once | At-most-once | At-least-once (with ACK) |
| **Fan-out** | No | Yes (all) | Yes (via multiple groups) |
| **Load balancing** | No | No | Yes (within group) |
| **Pending entries (retry)** | No | No | Yes (XPENDING + XCLAIM) |

### Stream Message Structure

```
Stream "shipment-events":
  ID                    Fields
  1686653000000-0  →    event_type=DEPARTED  shipment=SHP-001  carrier=MAERSK
  1686653001000-0  →    event_type=ARRIVED   shipment=SHP-002  carrier=HAPAG
  1686653002000-0  →    event_type=DELIVERED shipment=SHP-001
         ↑
  Auto-generated ID: Unix-ms-timestamp + sequence number
```

**Key Commands:**
```bash
# Producer: append message (* = auto-generate ID)
XADD shipment-events * event_type DEPARTED shipment_id SHP-001 carrier MAERSK

# Consumer (simple, no group): read from beginning
XREAD COUNT 10 STREAMS shipment-events 0

# Consumer (simple): read only NEW messages from now
XREAD BLOCK 5000 COUNT 10 STREAMS shipment-events $

# Create consumer group ($ = only new messages from this point)
XGROUP CREATE shipment-events tracking-workers $ MKSTREAM

# Read as consumer group (> = new messages not yet delivered to this group)
XREADGROUP GROUP tracking-workers worker-pod-1 COUNT 5 BLOCK 5000 STREAMS shipment-events >

# Acknowledge successful processing
XACK shipment-events tracking-workers 1686653000000-0

# Check pending (delivered but not yet ACKed)
XPENDING shipment-events tracking-workers - + 10

# Claim timed-out pending messages from crashed consumer
XCLAIM shipment-events tracking-workers worker-pod-2 60000 1686653000000-0
```

**Consumer group mechanics:**
```
Stream: shipment-events
  msg-1: SHP-001 DEPARTED
  msg-2: SHP-002 ARRIVED
  msg-3: SHP-003 DEPARTED
  msg-4: SHP-001 DELIVERED

Consumer Group: tracking-workers (PEL = Pending Entry List per consumer)
  worker-pod-1 receives: msg-1, msg-3   (pending ACK in worker-pod-1's PEL)
  worker-pod-2 receives: msg-2, msg-4   (pending ACK in worker-pod-2's PEL)

Each message delivered to exactly ONE consumer in the group.
Message stays in PEL until ACKed — survives consumer crashes.
Another consumer can XCLAIM messages stuck in PEL > timeout.
```

**Spring Boot Streams:**
```java
// Producer
redisTemplate.opsForStream().add("shipment-events",
    Map.of("event_type", "DEPARTED", "shipment_id", "SHP-001"));

// Consumer (using StreamMessageListenerContainer)
@Bean
public StreamMessageListenerContainer<String, MapRecord<String, String, String>> streamContainer(
        RedisConnectionFactory factory) {
    StreamMessageListenerContainer.StreamMessageListenerContainerOptions<String, MapRecord<String, String, String>> options =
        StreamMessageListenerContainer.StreamMessageListenerContainerOptions.builder()
            .pollTimeout(Duration.ofSeconds(1))
            .build();
    return StreamMessageListenerContainer.create(factory, options);
}
```

---

## Phase 12: Transactions and Atomicity

### MULTI/EXEC — Redis Transactions

Redis transactions group commands into an **atomic execution block** — all commands run sequentially without interleaving from other clients.

```bash
MULTI                              # Begin transaction — queue subsequent commands
SET inventory:SKU-001 50
DECRBY balance:user:123 100
INCR orders:count
EXEC                               # Execute all queued commands atomically
# Returns array of results from each command
```

**What MULTI/EXEC guarantees:**
- All queued commands execute **sequentially** — no other client's commands interleave
- Commands are sent as a single unit

**What MULTI/EXEC does NOT guarantee:**
- No rollback on partial failure — if one command errors, others still execute
- Not equivalent to SQL ACID transactions

```bash
MULTI
SET mykey "hello"
INCR mykey               # Will fail — INCR on non-integer
SET mykey2 "world"
EXEC

Results:
  1) OK      ← SET succeeded
  2) ERR...  ← INCR failed (type error) — does NOT abort the transaction
  3) OK      ← SET succeeded despite #2 failing
```

> ⚠️ **Common Mistake:** Expecting Redis transactions to roll back on error like SQL. They don't. The error in step 2 does not undo step 1. Redis transactions are about **serialized execution**, not all-or-nothing semantics.

> **Interviewer's Perspective:** "Are Redis transactions ACID?" → Atomic (all commands execute) and Isolated (no interleaving) in a limited sense. Not Consistent (no rollback) and not Durable (depends on persistence config). A nuanced answer shows depth.

---

### WATCH — Optimistic Locking

`WATCH` enables **optimistic locking**: if a watched key changes between the WATCH and EXEC, the transaction is aborted (EXEC returns nil).

```bash
WATCH inventory:SKU-001            # Watch this key for changes

current = GET inventory:SKU-001    # Read current value: 10

MULTI
  DECRBY inventory:SKU-001 3      # Attempt decrement
EXEC
# → If inventory:SKU-001 was modified between WATCH and EXEC: returns nil (aborted)
# → If not modified: returns [OK] with decrement applied
```

**WATCH-based optimistic inventory reservation in Java:**
```java
public boolean reserveInventory(String sku, int quantity) {
    List<Object> result = redisTemplate.execute(new SessionCallback<>() {
        @Override
        public List<Object> execute(RedisOperations ops) {
            ops.watch("inventory:" + sku);

            Integer current = (Integer) ops.opsForValue().get("inventory:" + sku);
            if (current == null || current < quantity) {
                ops.unwatch();
                return null;
            }

            ops.multi();
            ops.opsForValue().decrement("inventory:" + sku, quantity);
            return ops.exec(); // null if watched key changed
        }
    });
    return result != null; // non-null = transaction committed
}

// Retry loop for contention
public boolean tryReserve(String sku, int qty) throws InterruptedException {
    for (int i = 0; i < 5; i++) {
        if (reserveInventory(sku, qty)) return true;
        Thread.sleep(10 * (i + 1)); // exponential backoff
    }
    return false;
}
```

---

### Lua Scripts — True Atomicity

Lua scripts execute **entirely atomically** in Redis. No other command can execute between lines of a Lua script. This solves the check-then-act problem without optimistic retry loops.

```lua
-- Atomic inventory check-and-decrement
-- KEYS[1] = inventory key, ARGV[1] = quantity requested
local current = tonumber(redis.call('GET', KEYS[1]))
if current == nil then
    return redis.error_reply('Key does not exist')
end
if current < tonumber(ARGV[1]) then
    return 0  -- Insufficient stock
end
redis.call('DECRBY', KEYS[1], ARGV[1])
return 1  -- Reservation successful
```

```java
DefaultRedisScript<Long> script = new DefaultRedisScript<>();
script.setScriptText(luaScript);
script.setResultType(Long.class);

Long result = redisTemplate.execute(script,
    Collections.singletonList("inventory:SKU-001"),
    String.valueOf(quantity));

boolean reserved = Long.valueOf(1).equals(result);
```

**Script caching with SHA (production best practice):**
```java
// Load script once at startup — Redis caches it by SHA1 hash
String sha = redisTemplate.execute((RedisCallback<String>) conn ->
    conn.scriptLoad(luaScript.getBytes()));

// Subsequent calls use SHA — no script retransmission overhead
Long result = redisTemplate.execute((RedisCallback<Long>) conn ->
    conn.evalSha(sha, ReturnType.INTEGER, 1,
        "inventory:SKU-001".getBytes(), quantity.toString().getBytes()));
```

**WATCH vs Lua Script tradeoffs:**
| | WATCH + MULTI/EXEC | Lua Script |
|---|---|---|
| **Atomicity** | Optimistic (may abort and retry) | Guaranteed single execution |
| **Round trips** | Multiple (WATCH + MULTI + commands + EXEC) | Single (EVAL or EVALSHA) |
| **Read-then-branch** | Not possible inside MULTI | Fully supported in Lua |
| **Complexity** | Simpler, no Lua knowledge required | Requires Lua knowledge |
| **High contention** | Many retries degrade performance | Preferred |
| **Use case** | Low contention, simple operations | Complex logic, high concurrency |

---

## Phase 13: Concurrency and Race Conditions

### Why Single-Threaded Does Not Eliminate Business Concurrency

This is the most important advanced misconception about Redis, and the most common gap between junior and SDE-2 answers.

**The misconception:** "Redis is single-threaded, so there are no race conditions."

**The reality:** Each Redis command is atomic. But when your application logic requires **reading a value, making a decision, then writing back** — those are three separate commands. Between your READ and your WRITE, another application thread on another pod can perform the same sequence.

```
Business logic (NOT atomic as a whole):
  Step 1: Read inventory count from Redis
  Step 2: Check if count >= requested quantity  <- application code, not Redis
  Step 3: Write decremented count to Redis

Timeline with 2 concurrent pods:
  T=1  Pod A: GET inventory:SKU-001 -> 5
  T=2  Pod B: GET inventory:SKU-001 -> 5     <- SAME value read
  T=3  Pod A: 5 >= 3? YES
  T=4  Pod B: 5 >= 3? YES
  T=5  Pod A: SET inventory:SKU-001 2        <- correct
  T=6  Pod B: SET inventory:SKU-001 2        <- ALSO writes 2! Oversold!
```

**Redis executed every command correctly.** The race condition is entirely in the application logic between Redis commands.

**Solutions (in order of preference):**
1. **Lua script** — make the entire check-then-act a single atomic operation
2. **WATCH + MULTI/EXEC** — optimistic lock, retry on conflict
3. **Atomic commands** — `DECRBY` and detect negative result
4. **Distributed lock** — serializes all pods through one lock (heaviest option)

```lua
-- Lua: atomic check-and-decrement -- no race condition possible
local current = tonumber(redis.call('GET', KEYS[1]))
if current >= tonumber(ARGV[1]) then
    redis.call('DECRBY', KEYS[1], ARGV[1])
    return 1  -- success
else
    return 0  -- insufficient
end
```

> **Interviewer's Perspective:** "Is Redis thread-safe?" → Individual Redis commands are atomic. Business logic spanning multiple commands is NOT automatically thread-safe. This distinction separates SDE-2 answers from junior answers.

> **What Junior Engineers Usually Miss:** They treat Redis's single-threaded nature as a guarantee that their entire application flow is race-condition-free. The single thread only serializes Redis commands — not the application logic between them running across multiple pods.

> **What SDE-2 Candidates Should Know:** Any Read-Check-Write business operation must be made atomic via Lua, WATCH, or a distributed lock. The choice depends on contention level: Lua for high contention (no retries), WATCH for low contention (occasional retries acceptable), locks for operations that take significant time.

> **What Senior Engineers Know:** The right default is Lua scripts for complex atomic operations. WATCH is appropriate only for low-contention scenarios. Distributed locks introduce the most overhead and should be the last resort. For idempotent operations, consider `SET key value NX` + a unique request ID instead.

---

## Phase 14: Distributed Locking

### Why Distributed Locks Are Needed

In a microservices system with multiple pods, you need **mutual exclusion** across processes for:
- Only one pod should process a payment at a time (idempotency guarantee)
- Only one scheduler instance should run a cron job at a time
- Only one process should regenerate an expensive cached value (stampede prevention)

### Simple Redis Distributed Lock

```bash
SET lock:order:12345  "pod-3-uuid-abc123"  NX  EX  30
#   key               unique owner ID      NX  TTL safety net
```

```java
public boolean acquireLock(String resource, String ownerId, Duration ttl) {
    Boolean acquired = redisTemplate.opsForValue()
        .setIfAbsent("lock:" + resource, ownerId, ttl);
    return Boolean.TRUE.equals(acquired);
}

// Release MUST be atomic — check owner then delete in one operation
private static final String RELEASE_SCRIPT =
    "if redis.call('get', KEYS[1]) == ARGV[1] then\n" +
    "  return redis.call('del', KEYS[1])\n" +
    "else\n" +
    "  return 0\n" +
    "end";

public boolean releaseLock(String resource, String ownerId) {
    DefaultRedisScript<Long> script = new DefaultRedisScript<>(RELEASE_SCRIPT, Long.class);
    Long result = redisTemplate.execute(script,
        Collections.singletonList("lock:" + resource), ownerId);
    return Long.valueOf(1).equals(result);
}
```

**Why the release must be atomic Lua (not GET + DEL):**
```
Without Lua:
  Pod A: GET lock:order:12345 -> "uuid-A" -> mine
  Pod A pauses (GC for 35 seconds)
  Lock TTL (30s) expires
  Pod B: acquires lock with "uuid-B"
  Pod A resumes: DEL lock:order:12345  <- deletes Pod B's lock! DISASTER

With Lua (atomic GET + conditional DEL):
  No window between the check and delete — this race cannot happen.
```

**Lock renewal for long-running operations:**
```java
public void processWithLock(String orderId) {
    String ownerId = UUID.randomUUID().toString();
    boolean locked = acquireLock("order:" + orderId, ownerId, Duration.ofSeconds(30));
    if (!locked) throw new ResourceBusyException("Order being processed elsewhere");

    ScheduledFuture<?> renewal = scheduler.scheduleAtFixedRate(
        () -> redisTemplate.expire("lock:order:" + orderId, Duration.ofSeconds(30)),
        15, 15, TimeUnit.SECONDS); // Renew every 15s; TTL never expires mid-work

    try {
        performOrderProcessing(orderId);
    } finally {
        renewal.cancel(true);
        releaseLock("order:" + orderId, ownerId);
    }
}
```

---

### The Redlock Algorithm

**Problem with simple locking + HA Redis:** If the master dies after `SET NX` but before replicating, the promoted replica has no lock record → two pods hold the lock simultaneously.

**Redlock** (by antirez): acquire lock on majority of N independent Redis masters (no replication between them).

```
Setup: 5 independent Redis nodes (no master-replica relationship)
Quorum: 3 (majority of 5)

Acquire on all 5 simultaneously:
  Node1: OK   Node2: OK   Node3: OK   Node4: timeout   Node5: fail
  Quorum of 3 achieved.

lock_validity = TTL - elapsed_acquisition_time - clock_drift_safety_margin
Valid if: quorum >= 3 AND lock_validity > 0

Release: Send Lua delete to ALL 5 nodes.
```

**The Kleppmann critique:**
1. **Process pauses** (GC, OS scheduling) can cause lock holder to act after lock_validity expires
2. **Clock skew** between nodes invalidates timing assumptions
3. **No fencing token** — nothing prevents a stale lock holder from causing damage

**Fencing token pattern (correct for strict safety):**
```
Lock service issues monotonically increasing token on each lock acquisition.
Storage layer rejects writes with stale tokens (token < current max seen).
Client A (stale, token=42) cannot overwrite Client B's work (token=43).
```

**Decision matrix:**
| Scenario | Recommendation |
|---|---|
| Best-effort coordination (dedup cron) | Simple SET NX EX + UUID Lua release |
| HA Redis, tolerable brief double-processing | Redlock (document its limits) |
| Zero data corruption tolerance | ZooKeeper/etcd + fencing tokens |
| Spring Boot applications | `RedisLockRegistry` (Spring Integration) |

> **What Senior Engineers Know:** For most application-level coordination (preventing duplicate cron jobs, deduplicating idempotent requests), simple `SET NX EX` is entirely sufficient. Redlock complexity is only justified when partial locks cause data corruption and no fencing token mechanism exists.

---

## Phase 15: Cache Patterns (Redis-Specific Implementation)

See [[Caching]] for full strategy treatment. Redis-specific details:

### Cache-Aside with Spring Boot

```java
@Cacheable(value = "carriers", key = "#carrierId", unless = "#result == null")
public Carrier getCarrier(String carrierId) {
    return carrierRepository.findById(carrierId).orElse(null);
}

@CacheEvict(value = "carriers", key = "#carrierId")
@Transactional
public void updateCarrier(String carrierId, Carrier updated) {
    carrierRepository.save(updated);
    // Eviction fires AFTER DB write via Spring AOP
}

@CacheEvict(value = "carriers", allEntries = true)
public void importAllCarriers(List<Carrier> carriers) {
    carrierRepository.saveAll(carriers);
}
```

**Self-invocation trap:**
```java
@Service
public class CarrierService {
    // BROKEN: self-invocation bypasses Spring AOP proxy — caching doesn't happen
    public CarrierSummary getSummary(String id) {
        return toSummary(getCarrier(id));  // proxy bypassed!
    }

    @Cacheable("carriers")
    public Carrier getCarrier(String id) { ... }
    // Fix: extract getCarrier to a separate @Service bean
}
```

### TTL Configuration Per Cache

```java
@Bean
public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
    RedisCacheConfiguration base = RedisCacheConfiguration.defaultCacheConfig()
        .serializeValuesWith(SerializationPair.fromSerializer(
            new GenericJackson2JsonRedisSerializer()));

    return RedisCacheManager.builder(factory)
        .cacheDefaults(base.entryTtl(Duration.ofMinutes(30)))
        .withInitialCacheConfigurations(Map.of(
            "carriers",  base.entryTtl(Duration.ofHours(6)),
            "ports",     base.entryTtl(Duration.ofHours(24)),
            "shipments", base.entryTtl(Duration.ofMinutes(2)),
            "rates",     base.entryTtl(Duration.ofMinutes(15))
        ))
        .build();
}
```

---

## Phase 16: Cache Failure Scenarios

### Cache Stampede (Thundering Herd)

**Problem:** Hot key expires → all concurrent requests miss → all hit DB simultaneously.

```
"leaderboard:global" expires at T=0
T=0.001 through T=0.500: 2,000 concurrent requests all query DB for same data
                         -> DB CPU spikes to 100% -> cascading failure
```

**Solution 1: Distributed Mutex Lock**
```java
public Leaderboard getLeaderboard() {
    Leaderboard cached = (Leaderboard) redis.opsForValue().get("leaderboard:global");
    if (cached != null) return cached;

    String lockOwner = UUID.randomUUID().toString();
    boolean isLoader = Boolean.TRUE.equals(
        redis.opsForValue().setIfAbsent("lock:leaderboard", lockOwner, Duration.ofSeconds(5)));

    if (isLoader) {
        try {
            Leaderboard fresh = db.computeLeaderboard();
            redis.opsForValue().set("leaderboard:global", fresh, Duration.ofMinutes(5));
            return fresh;
        } finally {
            releaseLock("lock:leaderboard", lockOwner);
        }
    }
    Thread.sleep(100); // Wait for the loading pod to finish
    Object retried = redis.opsForValue().get("leaderboard:global");
    return retried != null ? (Leaderboard) retried : getLeaderboard();
}
```

**Solution 2: Probabilistic Early Refresh (XFetch)**
```java
Long ttl = redis.getExpire(key, TimeUnit.SECONDS);
Carrier cached = (Carrier) redis.opsForValue().get(key);
if (cached != null && ttl != null && ttl > 0) {
    // Probability of background refresh increases as TTL approaches zero
    if (Math.random() < Math.exp(-ttl / 60.0)) {
        CompletableFuture.runAsync(() -> refreshFromDb(key)); // Background refresh
    }
    return cached;
}
return loadAndCache(key); // Synchronous miss
```

### Cache Avalanche

**Problem:** Many keys expire simultaneously → mass DB queries.

```java
// Solution: TTL jitter spreads expirations over a window
int jitter = new Random().nextInt(600); // 0-10 minutes
redis.opsForValue().set(key, value, Duration.ofSeconds(3600 + jitter));
```

### Cache Penetration

**Problem:** Requests for non-existent data always miss → always hit DB.

**Solution 1: Null caching**
```java
Shipment result = shipmentRepository.findById(id).orElse(null);
redis.opsForValue().set("shipment:" + id,
    result != null ? result : "__NULL__",
    result != null ? Duration.ofHours(1) : Duration.ofMinutes(5));
```

**Solution 2: Bloom Filter**
```java
RBloomFilter<String> bloom = redisson.getBloomFilter("shipment-bloom");
bloom.tryInit(10_000_000L, 0.01); // 10M entries, 1% false-positive rate, ~12MB

// On shipment creation: bloom.add(shipment.getId())
// On read: if (!bloom.contains(id)) return null; // Definitely not in DB
```

```
Memory comparison:
  Bloom filter for 10M IDs at 1% FP: ~12MB
  Set of 10M string IDs:             ~800MB
  Bloom filter is 65x more memory-efficient
```

---

## Phase 17: Rate Limiting

### Fixed Window

```java
public boolean isAllowed(String userId) {
    String key = "rate:" + userId + ":"
        + LocalDateTime.now().truncatedTo(ChronoUnit.MINUTES);
    Long count = redis.opsForValue().increment(key);
    if (count == 1) redis.expire(key, Duration.ofMinutes(2));
    return count <= 100;
}
// Problem: user can send 100 at 10:30:59 and 100 at 10:31:01 = 200 in 2 seconds
```

### Sliding Window (Sorted Set + Lua)

```java
private static final String SLIDING_SCRIPT =
    "local now = tonumber(ARGV[1])\n" +
    "local win = tonumber(ARGV[2])\n" +
    "local lim = tonumber(ARGV[3])\n" +
    "redis.call('ZREMRANGEBYSCORE', KEYS[1], '-inf', now - win)\n" +
    "local count = redis.call('ZCARD', KEYS[1])\n" +
    "if count < lim then\n" +
    "  redis.call('ZADD', KEYS[1], now, now .. math.random())\n" +
    "  redis.call('EXPIRE', KEYS[1], math.ceil(win/1000) + 1)\n" +
    "  return 1\n" + // allowed
    "else return 0 end"; // denied

public boolean isAllowed(String userId, int limit, Duration window) {
    long nowMs = System.currentTimeMillis();
    Long result = redis.execute(
        new DefaultRedisScript<>(SLIDING_SCRIPT, Long.class),
        List.of("rate:sliding:" + userId),
        String.valueOf(nowMs), String.valueOf(window.toMillis()), String.valueOf(limit));
    return Long.valueOf(1).equals(result);
}
```

### Rate Limiting Strategy Comparison

| Approach | Accuracy | Burst Allowed | Memory Per User | Complexity |
|---|---|---|---|---|
| Fixed Window | Low (boundary burst) | Yes at boundary | O(1) | Low |
| Sliding Window | High | No | O(requests in window) | Medium |
| Token Bucket | High | Yes (controlled) | O(1) | Medium |
| Leaky Bucket | High | No (smoothed output) | O(1) | Medium |

---

## Phase 18: Session Management

```yaml
spring:
  session:
    store-type: redis
    timeout: 30m
    redis:
      namespace: spring:session
      flush-mode: on-save
```

```java
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
@Configuration
public class SessionConfig {}
// Spring Session auto-stores every HttpSession as a Redis Hash
```

**Redis key structure:**
```
spring:session:sessions:{sessionId}          -> Hash (all session attrs + metadata)
spring:session:sessions:expires:{sessionId}  -> TTL key (keyspace notification triggers cleanup)
spring:session:index:PRINCIPAL_NAME:{user}   -> Set of sessionIds (enables bulk revocation)
```

**Bulk revocation (password change, account compromise):**
```java
public void revokeAllUserSessions(String username) {
    Map<String, ? extends Session> sessions =
        sessionRepository.findByPrincipalName(username);
    sessions.keySet().forEach(sessionRepository::deleteById);
}
```

> **What SDE-2 Candidates Should Know:** Spring Session uses a separate `expires:` key with the actual TTL because Redis keyspace notifications fire on that key's expiry event. The listener then cleans up the main session Hash (which has no TTL itself). This two-key design is intentional and documented in Spring Session architecture.

---

## Phase 19: Inventory, Coupons, and Scheduler Coordination

### Atomic Inventory Reservation

```java
private static final String RESERVE_SCRIPT =
    "local cur = tonumber(redis.call('GET', KEYS[1]))\n" +
    "if cur == nil then return -1 end\n" +        // key doesn't exist
    "if cur < tonumber(ARGV[1]) then return 0 end\n" + // insufficient
    "redis.call('DECRBY', KEYS[1], ARGV[1])\n" +
    "return 1";                                    // success

public ReservationResult reserve(String sku, int qty) {
    DefaultRedisScript<Long> script = new DefaultRedisScript<>(RESERVE_SCRIPT, Long.class);
    Long r = redis.execute(script, List.of("inventory:" + sku), String.valueOf(qty));
    return r == 1 ? SUCCESS : r == 0 ? INSUFFICIENT : NOT_FOUND;
}

public void release(String sku, int qty) {
    redis.opsForValue().increment("inventory:" + sku, qty); // INCRBY is atomic
}
```

**Hot key at extreme scale (500K concurrent buyers, 1 SKU):**
```
Single key "inventory:SKU-001" = single slot = single node = throughput bottleneck.

Solution: Shard inventory
  inventory:SKU-001:s0 -> 100 units
  inventory:SKU-001:s1 -> 100 units
  ... (10 shards x 100 = 1000 total)

Client picks random shard. If that shard returns INSUFFICIENT, try next shard.
Periodic reconciliation job rebalances shards.
Result: 10x throughput capacity, load spread across 10 slots/nodes.
```

### Coupon Redemption (Atomic Lua)

```java
private static final String REDEEM_SCRIPT =
    "if redis.call('SISMEMBER', KEYS[2], ARGV[1]) == 1 then return -1 end\n" +
    "local rem = tonumber(redis.call('GET', KEYS[1]))\n" +
    "if rem == nil or rem <= 0 then return 0 end\n" +
    "redis.call('DECRBY', KEYS[1], 1)\n" +
    "redis.call('SADD', KEYS[2], ARGV[1])\n" +
    "return 1";
// KEYS[1]: coupon:CODE:remaining  KEYS[2]: coupon:CODE:redeemed  ARGV[1]: userId
// Returns: 1=success, 0=exhausted, -1=already redeemed
```

### Scheduler Leader Election

```java
@Scheduled(fixedDelay = 60_000)
public void runScheduledJob() {
    String podId = System.getenv("POD_NAME"); // unique Kubernetes pod identity
    boolean isLeader = Boolean.TRUE.equals(
        redis.opsForValue()
            .setIfAbsent("scheduler:daily-report", podId, Duration.ofSeconds(55)));
    if (isLeader) {
        log.info("Pod {} elected as leader, running job", podId);
        performScheduledWork();
    }
    // Non-leaders silently skip. TTL < interval ensures fair competition next cycle.
}
```

---

## Phase 20: Redis in Microservices

### Cache Invalidation via Pub/Sub

```java
// Publisher (service that owns the data)
redis.convertAndSend("cache:invalidate", "carrier:MAERSK");

// Subscriber (every service that caches carrier data)
public void handleMessage(String message) {
    String[] parts = message.split(":", 2);
    Cache cache = cacheManager.getCache(parts[0]);
    if (cache != null) cache.evict(parts[1]);
    log.debug("Cache evicted: {}", message);
}
```

### Always SCAN, Never KEYS

```java
// NEVER in production — blocks event loop
redis.keys("session:*");

// ALWAYS use SCAN — cursor-based, non-blocking
ScanOptions opts = ScanOptions.scanOptions().match("session:*").count(100).build();
List<String> keys = redis.execute((RedisCallback<List<String>>) conn -> {
    List<String> result = new ArrayList<>();
    try (Cursor<byte[]> cursor = conn.scan(opts)) {
        cursor.forEachRemaining(k -> result.add(new String(k)));
    }
    return result;
});
```

### Pipelining for Batch Writes

```java
// N round trips (slow):
for (Carrier c : carriers) redis.opsForValue().set("carrier:" + c.getId(), c);

// 1 round trip (fast):
redis.executePipelined((RedisCallback<Object>) conn -> {
    carriers.forEach(c ->
        conn.setEx(("carrier:" + c.getId()).getBytes(), 3600, serialize(c)));
    return null;
});
```

---

## Phase 21: Redis Failure Scenarios

### Redis Restart (Cold Start)

```
T=0:   Redis dies
T=0:   All pods detect connection failure -> circuit breakers open
T=0:   All reads fall through to DB -> 50x normal DB traffic -> overload risk
T=10s: Redis restarts, loads RDB+AOF -> accepting connections
T=60s: Hit ratio recovers to ~90% via cache-aside repopulation

Mitigations:
  1. Persistence (RDB+AOF) -> warm data on restart, shorter warm-up
  2. Circuit breaker (Resilience4j) -> graceful DB fallback
  3. Cache warming: @EventListener(ApplicationReadyEvent.class) -> preload hot keys
  4. Connection pool sizing: ensure DB pool handles 5x normal load during warm-up
```

```java
@CircuitBreaker(name = "redis", fallbackMethod = "getCarrierFromDb")
@Cacheable("carriers")
public Carrier getCarrier(String id) {
    return (Carrier) redis.opsForValue().get("carrier:" + id);
}

public Carrier getCarrierFromDb(String id, Exception e) {
    log.warn("Redis unavailable, falling back to DB for carrier:{}", id);
    return carrierRepository.findById(id).orElseThrow();
}

@EventListener(ApplicationReadyEvent.class)
public void warmCache() {
    carrierRepository.findAllActive().forEach(c ->
        redis.opsForValue().set("carrier:" + c.getId(), c,
            Duration.ofHours(6 + new Random().nextInt(600))));
}
```

### Sentinel Failover Timeline

```
T=0s:   Master unresponsive
T=30s:  Sentinel quorum declares ODOWN (majority agree master is down)
T=35s:  Leader Sentinel elected (Raft-based)
T=38s:  Best replica promoted: REPLICAOF NO ONE
T=40s:  Other replicas redirected to new master
T=42s:  Applications re-query Sentinel -> get new master address
T=42s:  Applications reconnect

Impact:
  Write unavailability: ~42 seconds
  Read availability: replica reads continue throughout
  Data loss: writes since last replication sync (typically milliseconds)
```

### Cluster Node Failure

```
Cluster: A+A' B+B' C+C' (master+replica pairs for each shard)

T=0s:   Node A (master, slots 0-5460) fails
T=1s:   Other cluster nodes detect via gossip
T=15s:  Cluster promotes A' to master for slots 0-5460
T=15s:  All slot 0-5460 traffic routes to A'

Impact:
  Slots 0-5460: ~15 seconds unavailability
  Slots 5461-16383: fully available throughout
  Data loss: possible last few milliseconds of writes if A->A' replication was lagging
```

### Split-Brain Prevention

```bash
# Minority partition refuses writes if it can't reach enough replicas
min-replicas-to-write 1
min-replicas-max-lag 10   # replica must be <= 10 seconds behind
# Without a connected in-sync replica, the node refuses writes
# -> No split-brain data divergence during network partition
```

---

## Phase 22: Redis Memory Management

### Memory Monitoring

```bash
INFO memory
# used_memory:             104857600   (bytes Redis believes it's using)
# used_memory_rss:         209715200   (bytes as seen by OS -- includes fragmentation)
# mem_fragmentation_ratio: 2.0         (>1.5 = high fragmentation)
# maxmemory:               4294967296  (configured hard limit)
# maxmemory_policy:        allkeys-lru
# mem_allocator:           jemalloc-5.3.0
```

**Memory overhead per structure (approximate):**
```
String:        ~56 bytes overhead + value bytes
Hash field:    ~64 bytes per field (hashtable) / ~16 bytes (listpack for small hashes)
List element:  ~32 bytes per element (quicklist)
Set member:    ~64 bytes per member (hashtable)
Sorted set:    ~128 bytes per member (skiplist + hashtable)
```

**Online defragmentation (Redis 4+):**
```bash
CONFIG SET activedefrag yes
CONFIG SET active-defrag-threshold-lower 10   # Start when frag ratio > 10%
CONFIG SET active-defrag-ignore-bytes 100mb   # Only defrag if > 100MB fragmented
```

### Eviction Policies

| Policy | Best For |
|---|---|
| `noeviction` | Primary store — size Redis so eviction never triggers |
| `allkeys-lru` | **Pure cache (recommended default)** |
| `volatile-lru` | Mix: cached data (with TTL) + permanent data (no TTL) |
| `allkeys-lfu` | Stable hot data where frequency > recency |
| `volatile-lfu` | Same as volatile-lru but frequency-based eviction |
| `volatile-ttl` | Evict soonest-to-expire cache entries first |

```bash
CONFIG SET maxmemory 4gb
CONFIG SET maxmemory-policy allkeys-lru
# Or: maxmemory-samples 10   (higher = more accurate LRU, more CPU)
```

> **What Senior Engineers Know:** Redis LRU is **approximated** — it samples `maxmemory-samples` random keys and evicts the least-recently-used among them. Redis 4+ added LFU, which uses a logarithmic Morris counter per key. Counter increments probabilistically on access and decrements for every `lfu-decay-time` minutes of inactivity. LFU outperforms LRU for workloads with stable hot keys (e.g., top 1% of carriers queried 1000x more than the rest) because it doesn't evict a once-popular key that simply hasn't been accessed in the last minute.

---

## Phase 23: Redis Security

```bash
# redis.conf hardening
bind 127.0.0.1 10.0.0.5          # Only internal interfaces; never 0.0.0.0
protected-mode yes
rename-command FLUSHALL ""        # Disable: delete all keys in all databases
rename-command CONFIG   ""        # Disable: prevent runtime config changes from clients
rename-command DEBUG    ""        # Disable: security risk
rename-command KEYS     ""        # Optional: force SCAN usage

# Redis 6+ ACL (fine-grained per-user permissions)
ACL SETUSER app-svc on >strongpass ~app:* &* +@read +@write +@string +@hash
ACL SETUSER readonly  on >pass ~* &* +@read
ACL SETUSER admin     on >adminpass ~* &* +@all
```

**TLS (Redis 6+):**
```bash
tls-port 6380
port 0                            # Disable plaintext port entirely
tls-cert-file /etc/redis/redis.crt
tls-key-file  /etc/redis/redis.key
tls-ca-cert-file /etc/redis/ca.crt
tls-auth-clients yes              # Mutual TLS: require client certificates
tls-protocols "TLSv1.2 TLSv1.3"
```

```yaml
spring:
  redis:
    username: app-svc
    password: strongpass
    ssl:
      enabled: true
```

**Data sensitivity rules:**
| Data | Cache in Redis? | Precaution |
|---|---|---|
| Session tokens | Yes | Short TTL, TLS, ACL per namespace |
| PII (names, addresses) | With caution | Encrypt values, short TTL |
| Passwords / hashes | No | DB only, with proper hashing |
| API keys / secrets | No | Vault / Secrets Manager |
| Payment card data | **Never** | PCI-DSS violation |
| Multi-tenant data | Yes | Always scope key: `tenant:{id}:entity:{id}` |

---

## Phase 24: Redis Monitoring

### Key Metrics to Track in Production

| Metric | Source | Alert Threshold |
|---|---|---|
| **Hit ratio** | `INFO stats` hits/(hits+misses) | Alert if < 85% |
| **Memory %** | `used_memory / maxmemory` | Alert at 80% |
| **Fragmentation** | `mem_fragmentation_ratio` | Alert if > 1.5 |
| **Evicted keys/s** | `INFO stats evicted_keys` rate | Alert if > 0 sustained |
| **Replication lag** | `INFO replication` master vs slave offset | Alert if > 1 second |
| **p99 latency** | `LATENCY HISTORY command` | Alert if > 2ms |
| **Slow commands** | `SLOWLOG GET 10` | Investigate any > 10ms |
| **Connected clients** | `INFO clients connected_clients` | Alert near `maxclients` |

### CLI Diagnostics

```bash
redis-cli --latency-history    # Continuous p50/p99 latency measurement
redis-cli --stat               # Real-time: ops/sec, memory, keys, connections
redis-cli --bigkeys            # Largest keys by type (uses SCAN — safe in prod)
redis-cli SLOWLOG GET 10       # Last 10 slow commands with execution time
redis-cli SLOWLOG RESET        # Clear slow log
redis-cli LATENCY HISTORY all  # Per-command latency histograms
redis-cli OBJECT ENCODING key  # Internal encoding (int/embstr/raw/listpack/etc.)
redis-cli OBJECT IDLETIME key  # Seconds since last access (LRU clock)
redis-cli OBJECT FREQ key      # Access frequency counter (LFU mode)
```

### Spring Boot Redis Metrics (Micrometer)

```yaml
management:
  endpoints:
    web:
      exposure:
        include: metrics, prometheus, health
```

```
GET /actuator/metrics/cache.gets?tag=name:carriers&tag=result:hit
GET /actuator/metrics/cache.gets?tag=name:carriers&tag=result:miss
GET /actuator/metrics/cache.evictions?tag=name:carriers
```

**Prometheus alert rules:**
```yaml
- alert: RedisHighMemoryUsage
  expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.8
  for: 5m

- alert: RedisLowHitRatio
  expr: redis_keyspace_hits_total / (redis_keyspace_hits_total + redis_keyspace_misses_total) < 0.85
  for: 10m

- alert: RedisEvictingKeys
  expr: increase(redis_evicted_keys_total[5m]) > 0
  for: 5m
```

---

## Phase 25: Real Interview Scenarios

### Scenario 1: Design an API Rate Limiter

**Interviewer:** "Design a rate limiter — 1,000 requests per minute per API key."

```
Clarify:
  Hard limit (reject at 1001) or soft (warn at 800)?
  Fixed window acceptable or sliding window required?
  Single region or global limit across regions?

Implementation (sliding window, Lua for atomicity):
  Key: rate:{apiKey}   Type: Sorted Set   Score: request timestamp ms

  On each request (single Lua round trip):
    1. ZREMRANGEBYSCORE: remove entries older than now - 60000ms
    2. ZCARD: count remaining entries in last 60 seconds
    3. If count < 1000: ZADD current timestamp -> 200 OK
    4. If count >= 1000: 429 Too Many Requests

  Response headers:
    X-RateLimit-Limit: 1000
    X-RateLimit-Remaining: max(0, 1000 - ZCARD)
    X-RateLimit-Reset: (ZSCORE first_member + 60000) / 1000

  Redis failure mode:
    Fail open (allow requests) for internal APIs — rate limiting is best-effort
    Fail closed (deny) for billing-critical or abuse-prone endpoints

  Multi-region:
    Simpler: per-region limit of 1000/min per region
    Complex: cross-region counter sync via gossip (adds ~5ms latency, complex)
    Usually the per-region approach is the right call unless globally unique limits required
```

---

### Scenario 2: Session Store for 50 Microservices

```
Data sizing:
  5M concurrent sessions x 500 bytes = 2.5GB -> fits in Redis

Architecture:
  JWT cookie (sessionId = UUID) -> any pod -> Redis Cluster

Session design:
  Key: session:{UUID}
  Type: Hash { userId, role, permissions, lastActive, createdAt, tenantId }
  TTL: sliding 30 minutes (extend EXPIRE on every request)

Reading sessions:
  ReadFrom.REPLICA_PREFERRED for read scaling
  Write session updates directly to master

Cross-service access:
  Option A: All services share same Redis cluster (simple, fast, tight coupling)
  Option B: Session service owns Redis, exposes /session/{id} REST API (decoupled)
  Option C: JWT + Redis blacklist for revocation (stateless reads + revocable)

Revocation:
  Logout:          DEL session:{id}
  Password change: findByPrincipalName(username) -> DELETE all session IDs for user
  Account lockout: same as password change, plus add userId to temporary blacklist

Security:
  sessionId = SecureRandom UUID (not predictable, not guessable)
  Redis behind private VPC, not publicly accessible
  TLS for all Redis connections
  HttpOnly + Secure + SameSite=Strict cookie flags
  ACL: each service has its own Redis user with namespace restrictions
```

---

### Scenario 3: Flash Sale — 10,000 Units, 500,000 Buyers

```
Layer 1: API Gateway token bucket rate limiting
  Reduce actual concurrent load reaching the application tier.
  E.g., 500K concurrent -> 50K concurrent after gateway shaping.

Layer 2: Atomic Lua reservation in Redis
  SET inventory:SKU-001 10000
  Lua: GET -> check >= qty -> DECRBY -> return 1 (success) / 0 (insufficient)
  Single serialized atomic operation — no overselling possible regardless of concurrency.

Layer 3: Hot key mitigation (500K concurrent on one key = single node bottleneck)
  Shard inventory:
    inventory:SKU-001:s0 through :s9 (1000 units each = 10000 total)
  Client picks random shard.
  If shard INSUFFICIENT: try next shard (round-robin).
  Result: 10x throughput capacity, load spread across 10 cluster slots.

Layer 4: Async DB write
  Lua reservation gates the sale. DB write is the audit record.
  On success: publish "order-reserved" event to Kafka.
  Order Service consumes event -> creates order record in PostgreSQL.

Layer 5: Reconciliation job
  Every 5 minutes: compare Redis inventory sum across shards vs DB confirmed orders.
  Alert on gap > threshold. Redis is source of truth for availability; DB for records.

Failure scenario:
  Redis dies mid-sale -> circuit breaker opens -> "Checkout temporarily unavailable" page
  Better user experience than silent overselling or DB-based reservation (too slow).
```

---

### Scenario 4: Real-Time Global Gaming Leaderboard

```
Data structure: Sorted Set
  Key: leaderboard:global
  Score: total points (float)   Member: userId

Core operations (all O(log N)):
  Update score:  ZINCRBY leaderboard:global {pts} {userId}
  Player rank:   ZREVRANK leaderboard:global {userId}
  Top 100:       ZREVRANGE leaderboard:global 0 99 WITHSCORES
  Nearby ranks:  ZREVRANGE leaderboard:global {rank-2} {rank+2} WITHSCORES

Memory sizing:
  10M players x ~128 bytes per member = ~1.3GB -> single Redis node feasible

Write throughput:
  ZINCRBY is O(log N). At 100K score updates/sec -> ~100K ops/sec.
  Single Redis node handles this. Above this: shard by game/season.

Multiple leaderboard dimensions:
  leaderboard:global             -> all-time overall
  leaderboard:weekly:2026-W24   -> current week (RENAME + EXPIRE at week boundary)
  leaderboard:country:IN        -> India-specific
  leaderboard:game:RPG-01       -> per-game instance

Weekly rotation:
  RENAME leaderboard:weekly:current leaderboard:weekly:2026-W23
  EXPIRE leaderboard:weekly:2026-W23 2592000  (keep 30 days for history)
  DEL leaderboard:weekly:current (start fresh)

Persistence:
  RDB every 30 seconds. Losing 30s of score updates is acceptable for a game.
  If not acceptable: leaderboard rebuilt from event log in DB on restart.
```

---

## Phase 26: Common Interview Mistakes

### Mistake 1: Calling Redis "Just a Cache"

Redis is a **data structure server** with caching, streaming, pub/sub, coordination, and rate limiting capabilities. Calling it "just a cache" signals shallow understanding to any interviewer who knows the technology.

### Mistake 2: KEYS in Production

```bash
# NEVER: blocks the event loop — can freeze Redis for seconds on large keyspaces
KEYS session:*

# ALWAYS: cursor-based, non-blocking
SCAN 0 MATCH session:* COUNT 100
```

### Mistake 3: No TTL on Cached Keys

```java
// Memory leak — key lives until eviction or manual delete:
redis.opsForValue().set("session:" + id, data);

// Correct:
redis.opsForValue().set("session:" + id, data, Duration.ofMinutes(30));
```

### Mistake 4: Reading from Replica After Write

```java
// May return stale data due to replication lag:
orderService.createOrder(order);
Order fetched = replicaRedis.get("order:" + id); // Potentially stale!

// Read-after-write must use master:
Order fetched = masterRedis.get("order:" + id);
```

### Mistake 5: Multi-Key Operations Across Cluster Slots

```bash
# CROSSSLOT error in Redis Cluster:
MGET user:123 order:456

# Fix — hash tags force same slot:
MGET {user:123}:profile {user:123}:orders
```

### Mistake 6: Not Pipelining Batch Operations

```java
// N round trips — slow:
for (String id : ids) redis.delete("session:" + id);

// 1 round trip — fast:
redis.executePipelined((RedisCallback<Object>) conn -> {
    ids.forEach(id -> conn.del(("session:" + id).getBytes()));
    return null;
});
```

### Mistake 7: Expecting MULTI/EXEC to Roll Back

MULTI/EXEC does **not** roll back on command-level errors. If one command in the block fails (e.g., wrong type), the others still execute. For all-or-nothing semantics, use a Lua script.

### Mistake 8: Using Default JDK Serializer in Spring

```java
// Default: JdkSerializationRedisSerializer — slow, Java-only, large binary blobs
// Fix: use JSON for cross-language compatibility, readability, and smaller size
template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
template.setKeySerializer(new StringRedisSerializer());
```

---

## Phase 27: FAANG SDE-2 Redis Questions

**Q1: How does INCR guarantee atomicity under concurrency?**
> Redis executes INCR on a single thread. No other command can interleave between INCR's read-modify-write steps because the event loop processes exactly one command at a time. No mutex is needed — there is no parallel thread that could race.

**Q2: What is the difference between Redis Sentinel and Redis Cluster?**
> Sentinel: provides **high availability for a single master's dataset** — automatic failover, no horizontal scaling. All data lives on one master. Cluster: provides **horizontal scaling** by sharding data across N masters using 16,384 hash slots, each master having its own replica. Cluster has built-in per-shard failure detection and promotion. Use Sentinel when dataset fits in one machine and you need HA. Use Cluster when you need to exceed one machine's memory or throughput.

**Q3: What is the time complexity of ZADD, and why?**
> O(log N), where N is the number of elements in the sorted set. The underlying structure is a skiplist — insert requires finding the correct sorted position across probabilistic skip levels, which takes O(log N). The hashtable component provides O(1) score lookup by member but does not affect insert complexity.

**Q4: What is the difference between MULTI/EXEC and Lua scripts for atomicity?**
> MULTI queues commands — no read-then-branch logic is possible within the block. If a command fails, others still execute (no rollback). Each call is a separate round trip. Lua runs as a single fully atomic unit, can read values and branch on them, and executes in one round trip. Use Lua when you need conditional logic inside the atomic operation, or when high contention makes WATCH retries too costly.

**Q5: How do you prevent cache penetration?**
> (1) **Null caching**: cache "not found" results with a short TTL (5 minutes). (2) **Bloom filter**: pre-screen IDs before cache/DB access using probabilistic membership — zero cost for IDs that definitely don't exist, 1% false-positive rate for a typical configuration. (3) **Input validation**: reject malformed IDs at the API boundary. (4) **Rate limiting**: prevent a single client from generating unbounded misses.

**Q6: When does WATCH abort a transaction, and when should you use it vs Lua?**
> EXEC returns nil if any WATCHed key was modified by any other client between WATCH and EXEC. The application detects nil and retries. Use WATCH for low-contention scenarios where conflicts are rare and retry overhead is acceptable. Use Lua scripts for high-contention scenarios — Lua is always atomic with no retries needed, at the cost of requiring Lua knowledge and a single round trip.

**Q7: What happens to unacknowledged messages in a Redis Stream consumer group?**
> They remain in the consumer's **Pending Entry List (PEL)**. They are not redelivered automatically. To recover from a crashed consumer: use `XPENDING` to find stuck messages, `XCLAIM` to reassign them to a healthy consumer. Once claimed, the new consumer processes and ACKs them. This provides at-least-once delivery semantics.

**Q8: Which eviction policy would you choose for a pure cache workload, and why?**
> `allkeys-lru`. In a pure cache, all keys are evictable — there is no "permanent" data that must never be removed. LRU aligns with temporal locality — recently accessed data is more likely to be needed again. `noeviction` causes OOM errors and write failures instead of graceful eviction. `volatile-lru` would not evict keys that don't have a TTL set, which could be a problem if not all cache keys are configured with TTL.

---

## Phase 28: Senior-Level Redis Questions

**Q1: Explain the Redis fork + copy-on-write mechanism and its production implications.**
> When Redis triggers an RDB snapshot (`BGSAVE`) or AOF rewrite, it `fork()`s a child process. The parent and child initially share the same physical memory pages via Linux copy-on-write. When the parent receives new writes after the fork, the OS gives the parent new memory pages for the modified data — the child still sees the pre-fork snapshot. This enables a zero-blocking snapshot at the cost of potentially doubling RAM usage under write-heavy load (every written page gets duplicated). Monitor `rdb_last_cow_size` in `INFO persistence`. Production rule: always keep at least 50% RAM headroom on Redis nodes that use RDB. Schedule RDB snapshots during low-write periods on large datasets.

**Q2: How does the Redis skiplist enable O(log N) rank queries?**
> Each skiplist node stores a **span** at each level — how many level-1 (bottom) nodes it skips over. A rank query traverses from the highest populated level downward, accumulating spans. When it reaches the target node, the total accumulated span equals the rank. Since traversal visits O(log N) nodes (skiplist's probabilistic height guarantee), the rank query is O(log N). This is why `ZRANK` and `ZREVRANK` are O(log N) — they traverse the skiplist and count spans, not the full element list.

**Q3: Design a Redis-based distributed cron that survives pod restarts.**
> Leader election via `SET scheduler:leader {podId} NX EX 30`. The winning pod runs the job and renews TTL every 10 seconds. If the winner crashes, TTL expires and the next `SET NX` from any pod wins. Store job state (last_run_time, last_run_status) in a Redis Hash with AOF persistence so it survives Redis restarts. On pod startup, read state to decide if the job needs immediate re-execution. Idempotent job design (using a unique run ID with conditional execution) prevents double-execution even if two pods briefly both believe they are the leader during a split-second leadership transition.

**Q4: How would you migrate 500GB of Redis data to a new cluster with zero downtime?**
> Phase 1 (dual-write, 1-2 days): Application writes to both old and new Redis. Reads still from old cluster. No user impact. Phase 2 (key migration, 2-5 days): Background process uses SCAN on old cluster + MIGRATE command to move each key to new cluster non-blockingly, key by key. New cluster catches up via dual-writes. Phase 3 (read cutover): Gradually shift reads to new cluster by feature flag or percentage — monitor hit ratio. Phase 4: Stop writes to old cluster once new cluster is stable. Phase 5: Decommission old cluster after a 1-week observation window. Total duration for 500GB: days, not hours.

**Q5: What is `lua-time-limit` and what happens when it is exceeded?**
> The `lua-time-limit` (default 5000ms) is the maximum time a Lua script can run before Redis considers it slow. When exceeded, Redis starts responding to other clients with `BUSY` errors but does **not** kill the running script — it lets it finish. `SCRIPT KILL` can abort the script only if it hasn't yet executed any write commands. If it has written data, `SCRIPT KILL` fails and recovery requires `SHUTDOWN NOSAVE` (data loss). Production implication: always benchmark Lua scripts thoroughly before deploying. Never write Lua scripts that iterate over large datasets.

**Q6: How does Redis LFU eviction differ from LRU in practice?**
> LRU evicts the key not accessed for the longest wall-clock time. LFU evicts the key with the lowest access frequency. Redis LFU uses a logarithmic Morris counter (0-255) per key — it increments probabilistically on access (probability decreases as count rises, preventing a counter from maxing out on a burst) and decrements by 1 for every `lfu-decay-time` minutes of inactivity (default 1). This solves the classic LFU "stale popular item" problem: a once-popular item that stopped being accessed decays toward zero and becomes evictable. LFU outperforms LRU for workloads with stable, consistently hot data — e.g., top 100 carrier master records accessed 10,000 times more frequently than the rest.

**Q7: Trade-offs between AOF `always` vs `everysec` fsync policies.**
> `always`: fsync after every write command. Write latency equals disk I/O latency (typically 1-5ms on SSD). At most 1 command lost on crash. `everysec`: background thread fsyncs every second. Write latency is near-RAM speed (microseconds). At most 1 second of commands lost on crash. For financial-critical systems where every write must be durable (payment records, order confirms): `always`. For production caches, session stores, rate limiters: `everysec` — the performance gain is dramatic and 1-second loss is acceptable. For pure ephemeral caches: no persistence. Most teams use `everysec` as the production default.

---

## Key Takeaways

1. **Redis is a data structure server** — reach for the right structure: String for counters/locks, Hash for partial-update objects, Sorted Set for rankings and delayed queues, HyperLogLog for approximate cardinality, Stream for reliable event logs
2. **Single-threaded execution** eliminates data-structure races but NOT multi-command business logic races — use Lua scripts or WATCH for composite atomicity
3. **Persistence**: RDB for fast restart + periodic backups, AOF (`everysec`) for ≤1 second data loss — use both in production with `aof-use-rdb-preamble yes`
4. **Sentinel = HA for one master's data; Cluster = horizontal scaling** — they solve different problems and are not interchangeable
5. **Lua scripts are preferred over WATCH** for high-contention atomic operations: single round trip, no retry loop, true atomicity, supports conditional logic
6. **Always set TTL + jitter** on every cached key — prevents memory leaks and cache avalanche
7. **Never use KEYS in production** — always SCAN with cursor; never MONITOR in production — use SLOWLOG and LATENCY HISTORY instead
8. **Hot keys are the #1 Redis scalability problem** — mitigate with L1 Caffeine cache per pod, key replication with random shard selection, or re-architecture (fan-out on write)
9. **Redlock is controversial** — simple `SET NX EX` with unique owner ID and Lua release is sufficient for most application coordination; for strict zero-double-processing guarantees use ZooKeeper/etcd with fencing tokens
10. **Monitor four metrics religiously**: hit ratio, memory usage %, replication lag, eviction rate — these four metrics tell you 80% of what you need to know about Redis health at any moment

---

**Related:** [[Caching]] | [[System Design/Index]] | [[Message Queues]]
