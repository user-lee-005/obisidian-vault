# Caching

> *"There are only two hard things in Computer Science: cache invalidation and naming things."* — Phil Karlton

---

## Phase 1: The Fundamentals

### What is Caching?

Caching is the technique of storing **frequently accessed data** in a **faster storage layer** so that future requests for that data can be served quicker — without going to the original, slower source every time.

**Analogy:** Think of a warehouse worker who ships the same 10 items hundreds of times a day. Instead of walking to the far end of the warehouse every time, they keep those 10 items on a shelf right next to the packing station. That shelf is a **cache**.

```
Without Cache:
Client → Server → Database (slow, every time)

With Cache:
Client → Server → Cache (fast, most times)
                 ↘ Database (only on cache miss)
```

### Why Do We Need Caching?

| Problem | How Caching Helps |
|---|---|
| **High Latency** | Reads from memory (~100μs) instead of disk/network (~10ms) — **100x faster** |
| **Database Overload** | Reduces repeated queries; DB handles only unique/fresh requests |
| **High Cost** | Fewer DB reads = fewer replicas needed = lower infra cost |
| **Throughput Bottleneck** | Cache can serve thousands of reads/sec per node |

**Example:** A logistics platform shows carrier details on every shipment page. Without caching, every page load queries the `carriers` table. With caching, the carrier data (which rarely changes) is served from Redis in <1ms instead of 5-10ms from PostgreSQL.

### How Caching Works — The Basic Flow

```
1. Client sends request to Server
2. Server checks the Cache:
   ├── CACHE HIT  → Data found → Return immediately (fast path)
   └── CACHE MISS → Data NOT found → Query Database → Store result in Cache → Return
```

**Cache Hit:** The requested data exists in cache → served instantly.
**Cache Miss:** Data is not in cache → fetch from source, populate cache, then return.
**Hit Ratio:** `Cache Hits / (Cache Hits + Cache Misses)` — aim for **>90%** in production.

---

## Phase 2: What Makes Data Cacheable?

Before jumping into strategies, you need to know **what should and shouldn't be cached**. This is the most important decision.

### ✅ Good Candidates for Caching

| Characteristic | Example |
|---|---|
| **Read-heavy, rarely changes** | Carrier master data, port codes, country lists |
| **Expensive to compute** | Freight rate calculations, route optimizations |
| **Frequently accessed by many users** | Dashboard aggregations, popular product listings |
| **Tolerance for slight staleness** | Shipment count summaries (ok if 30 sec behind) |

### ❌ Bad Candidates for Caching

| Characteristic | Example |
|---|---|
| **Write-heavy, rarely read** | Audit logs, event streams |
| **Read-once data** | One-time OTP codes, single-use download links |
| **Correctness-critical, real-time** | Payment processing, inventory reservation |
| **Highly personalized + low reuse** | Per-user, per-second analytics |
| **Large objects with low reuse** | File attachments, PDF reports generated once |

### The Golden Rule

> **Cache when the cost of a cache miss (fetching from source) is significantly higher than the cost of maintaining the cache, AND the data is accessed frequently enough to justify it.**

---

## Phase 3: Where Can We Cache? (The Layers)

Caching can happen at **multiple layers** of your system. Each layer serves a different purpose.

```
Client (Browser)
    ↓
CDN (Edge Servers)
    ↓
API Gateway / Reverse Proxy
    ↓
Application Layer (In-Process)
    ↓
Distributed Cache (Redis/Memcached)
    ↓
Database Layer (Buffer Pool)
```

### Layer 1: Client-Side Caching

The browser or mobile app stores responses locally to avoid making network requests entirely.

**How it works:**
- Server sends HTTP cache headers with the response
- Browser stores the response and serves it locally on subsequent requests

**Key HTTP Cache Headers:**
```
Cache-Control: max-age=3600          → Cache for 1 hour
Cache-Control: no-cache              → Always revalidate with server
Cache-Control: no-store              → Never cache (sensitive data)
ETag: "abc123"                       → Version fingerprint for conditional requests
Last-Modified: Mon, 01 Jan 2024      → Timestamp-based conditional caching
```

**Example:** A logistics dashboard's static assets (JS, CSS, images) are cached in the browser with `max-age=86400` (1 day). The carrier logo images rarely change, so the browser never re-downloads them.

---

### Layer 2: CDN Caching

A CDN (Content Delivery Network) is a geographically distributed network of servers that cache content **close to the user**.

**How it works:**
- User in Mumbai requests an image → CDN edge server in Mumbai serves it
- If not cached at edge → CDN fetches from origin server → caches at edge → serves

**Best for:** Static assets (images, CSS, JS), API responses that are the same for all users.

**Example:** A freight tracking portal used by clients across 30 countries. Product images and static UI assets are served from Cloudflare's CDN — a user in London gets content from a London edge server instead of the origin in Sydney.

---

### Layer 3: Application-Level Caching (In-Process)

Data is cached **inside the application's memory** (JVM heap in Java).

**Tools:** `HashMap`, `ConcurrentHashMap`, Google Guava Cache, **Caffeine** (recommended for Spring Boot)

**Pros:**
- Fastest possible access — no network hop
- No serialization/deserialization overhead

**Cons:**
- Cache is **local to each instance** — 3 app servers = 3 separate caches
- Data inconsistency between instances
- Consumes JVM heap memory

**Example:**
```java
// Caffeine cache in a Spring Boot app
LoadingCache<String, Carrier> carrierCache = Caffeine.newBuilder()
    .maximumSize(1000)
    .expireAfterWrite(Duration.ofMinutes(30))
    .build(carrierId -> carrierRepository.findById(carrierId));

Carrier carrier = carrierCache.get("MAERSK");  // Fetches from DB only on miss
```

**When to use:** Reference data that every request needs (country codes, carrier names, port mappings) and where slight inconsistency across instances is acceptable.

---

### Layer 4: Distributed Caching (Out-of-Process)

A **separate caching service** shared by all application instances.

**Tools:** **Redis**, Memcached, Hazelcast

**Pros:**
- Shared across all app instances — single source of truth for cached data
- Survives application restarts
- Scales independently from app servers

**Cons:**
- Network hop required (~0.5-1ms)
- Serialization/deserialization cost
- Additional infrastructure to manage

```
App Server 1 ──┐
App Server 2 ──┼──→ Redis Cluster ──→ (if miss) ──→ Database
App Server 3 ──┘
```

**Example:** Shipment tracking data cached in Redis. All 10 instances of the tracking service read from the same Redis cluster, so updating the cache once makes it available to all instances immediately.

---

### Layer 5: Database-Side Acceleration

The database itself has caching mechanisms:

| Mechanism | Description |
|---|---|
| **Buffer Pool** (InnoDB) | Frequently accessed data pages are kept in memory |
| **Result Caching** | Some DBs cache query results (Oracle) |
| **Materialized Views** | Precomputed query results stored as a table, refreshed periodically |
| **Denormalized Tables** | Read-optimized tables that duplicate data for faster queries |

> ⚠️ **Note:** MySQL's query cache was **removed in MySQL 8.0** because it caused more contention than benefit. Don't rely on it in modern systems.

---

### Multi-Level Caching (L1 + L2)

In production, you often combine layers for maximum performance:

```
Request → L1 (Caffeine, in-process, ~nanoseconds)
       → L2 (Redis, distributed, ~1ms)
       → Database (~5-50ms)
```

**Why two levels?**
- L1 handles the ultra-hot data with zero network cost
- L2 provides consistency across instances and survives restarts
- DB is the final fallback

**Example in Spring Boot:**
```java
// Check local Caffeine cache first
Carrier carrier = localCache.getIfPresent(carrierId);
if (carrier == null) {
    // Check Redis
    carrier = redisTemplate.opsForValue().get("carrier:" + carrierId);
    if (carrier == null) {
        // Query database
        carrier = carrierRepository.findById(carrierId);
        redisTemplate.opsForValue().set("carrier:" + carrierId, carrier, Duration.ofHours(1));
    }
    localCache.put(carrierId, carrier);
}
```

**Tradeoff:** More levels = faster reads BUT harder to keep consistent.

---

## Phase 4: Caching Strategies (Read/Write Patterns)

How your application reads from and writes to the cache. This is the core of cache architecture.

### Strategy 1: Cache-Aside (Lazy Loading)

The **application** is responsible for reading from and writing to the cache.

```
READ:
1. App checks cache
2. If HIT → return cached data
3. If MISS → query DB → write to cache → return

WRITE:
1. App writes to DB
2. App invalidates (deletes) the cache entry
```

**Pros:**
- Simple to implement
- Cache only contains data that's actually requested (no wasted memory)
- If cache goes down, app still works (just slower)

**Cons:**
- First request for any data is always slow (cache miss)
- Potential for stale data between write and invalidation

**Best for:** General purpose, read-heavy workloads.

**Spring Boot Example:**
```java
@Cacheable(value = "carriers", key = "#carrierId")
public Carrier getCarrier(String carrierId) {
    return carrierRepository.findById(carrierId);  // Called only on cache miss
}

@CacheEvict(value = "carriers", key = "#carrier.id")
public void updateCarrier(Carrier carrier) {
    carrierRepository.save(carrier);  // Evicts cache after DB write
}
```

---

### Strategy 2: Read-Through

Similar to Cache-Aside, but the **cache library itself** is responsible for loading data from the DB on a miss.

```
READ:
1. App asks cache for data
2. If HIT → cache returns data
3. If MISS → cache itself queries DB → stores it → returns to app

(App never talks to DB directly for reads)
```

**Pros:**
- Simpler application code — no manual cache miss handling
- Cache logic is centralized

**Cons:**
- Cache library needs to know how to query your data source
- First request is still slow

**Example:** Caffeine's `LoadingCache` is a read-through cache:
```java
LoadingCache<String, Port> portCache = Caffeine.newBuilder()
    .maximumSize(5000)
    .expireAfterWrite(Duration.ofHours(6))
    .build(portCode -> portRepository.findByCode(portCode));  // Cache loads from DB itself

Port port = portCache.get("INMAA");  // Cache handles miss transparently
```

---

### Strategy 3: Write-Through

Every write goes to **both the cache and the database synchronously**.

```
WRITE:
1. App writes to cache
2. Cache writes to DB (synchronously)
3. Acknowledge to app only after both succeed
```

**Pros:**
- Cache is always consistent with DB
- Reads after writes are always fast (data is already cached)

**Cons:**
- **Higher write latency** — every write has two operations
- Cache may store data that's never read (waste)

**Best for:** Data that is written and then immediately read back (e.g., user session data).

---

### Strategy 4: Write-Behind (Write-Back)

Writes go to the **cache first**, and the cache **asynchronously** flushes to the database later.

```
WRITE:
1. App writes to cache
2. Cache acknowledges immediately
3. Cache asynchronously batches and writes to DB (after delay or batch threshold)
```

**Pros:**
- **Very fast writes** — app doesn't wait for DB
- Batching reduces DB load (100 individual writes → 1 batch write)

**Cons:**
- **Risk of data loss** — if cache crashes before flushing, data is lost
- Complex to implement correctly
- Eventual consistency between cache and DB

**Best for:** Write-heavy workloads where slight data loss is acceptable (analytics, counters, tracking events).

**Example:** A real-time shipment tracking system receives GPS updates every second. Writing every update to the DB would overwhelm it. Instead, updates are written to Redis and flushed to the DB every 30 seconds in batches.

---

### Strategy 5: Write-Around

Writes go **directly to the database**, bypassing the cache entirely.

```
WRITE:
1. App writes directly to DB (cache is not updated)

READ (subsequent):
1. Cache miss → fetch from DB → populate cache
```

**Pros:**
- Cache isn't polluted with data that may not be read
- Simple write path

**Cons:**
- Reads after writes always result in a cache miss (higher read latency initially)

**Best for:** Data that is written frequently but read infrequently (audit logs, historical records).

---

### Strategy Comparison

| Strategy          | Read Latency | Write Latency | Consistency | Data Loss Risk | Best For               |
| ----------------- | ------------ | ------------- | ----------- | -------------- | ---------------------- |
| **Cache-Aside**   | Miss = slow  | Low           | Eventual    | None           | General purpose        |
| **Read-Through**  | Miss = slow  | Low           | Eventual    | None           | Simplified read logic  |
| **Write-Through** | Always fast  | High          | Strong      | None           | Read-after-write       |
| **Write-Behind**  | Always fast  | Very low      | Eventual    | **Yes**        | Write-heavy workloads  |
| **Write-Around**  | Miss = slow  | Low           | Eventual    | None           | Write-heavy, read-rare |

---

## Phase 5: Cache Key Design

Many production cache bugs come from **bad keys**, not bad eviction. Key design matters.

### Principles

1. **Be specific** — Include all dimensions that make the data unique
2. **Be consistent** — Follow a naming convention across the codebase
3. **Be scoped** — Include tenant/user context when needed
4. **Be versioned** — Allow key format changes without purging everything

### Naming Convention

```
{service}:{entity}:{identifier}[:version]

Examples:
  tracking:shipment:SHP-2024-001
  rating:freight-rate:MAERSK:INMAA:USLAX:v2
  auth:session:user-12345
  master:carrier:HAPAG
  master:port:INMAA
```

### Common Pitfalls

| Pitfall | Problem | Fix |
|---|---|---|
| **Too generic keys** | `cache:data:123` — what data? | Use entity names: `tracking:shipment:123` |
| **Missing tenant scope** | Multi-tenant app returns wrong data | `tenant:{tenantId}:shipment:{id}` |
| **Cardinality explosion** | Caching per-user + per-page + per-filter combos | Cap dimensions, cache shared data |
| **Unbounded key growth** | Never-expiring keys consume memory | Always set TTL |
| **No versioning** | Schema change → deserialization errors | `carrier:{id}:v2` |

### Composite Keys for Complex Queries

```
// Freight rate depends on: carrier + origin + destination + date
String cacheKey = String.format("rate:%s:%s:%s:%s", 
    carrierId, originPort, destPort, effectiveDate);
// rate:MAERSK:INMAA:USLAX:2024-01-15
```

---

## Phase 6: Cache Eviction Policies

When the cache is full, **which entry do we remove** to make room for new data?

### LRU — Least Recently Used

Evicts the entry that **hasn't been accessed for the longest time**.

```
Cache (capacity=3):  [A, B, C]
Access B →           [A, C, B]    (B moves to end)
Add D →              [C, B, D]    (A evicted — least recently used)
```

**When to use:** General purpose. Works well when recent data is more likely to be accessed again.
**Used by:** Redis (default approximated LRU), Caffeine, Memcached.

---

### LFU — Least Frequently Used

Evicts the entry with the **lowest access count**.

```
Cache:  A(5 hits), B(2 hits), C(8 hits)
Add D → B evicted (lowest frequency)
```

**When to use:** When some items are consistently popular (e.g., top 10 carriers accessed 1000x more than others).
**Caveat:** A once-popular item that's no longer relevant can stay cached forever.

---

### FIFO — First In, First Out

Evicts the **oldest entry** regardless of how often it's accessed.

```
Cache:  [A(oldest), B, C]
Add D → [B, C, D]    (A evicted — first in)
```

**When to use:** When all data has roughly equal access patterns and age correlates with relevance.

---

### TTL — Time-to-Live

Every entry has an **expiration timestamp**. After the TTL passes, the entry is automatically removed.

```java
// Redis: Set key with 30-minute TTL
redisTemplate.opsForValue().set("shipment:SHP-001", shipment, Duration.ofMinutes(30));
```

**How to choose TTL based on business tolerance:**

| Data Type | TTL | Reason |
|---|---|---|
| Port codes, country master | 24 hours | Changes very rarely |
| Carrier details | 1 hour | Changes occasionally |
| Freight rates | 15 minutes | Tariffs update, need freshness |
| Shipment tracking status | 30 seconds or NO cache | Real-time accuracy matters |
| User session | 30 minutes | Security timeout |

**Pro tip:** Add a **random jitter** to TTLs to prevent mass expiry (cache avalanche):
```java
int baseTTL = 3600; // 1 hour
int jitter = new Random().nextInt(300); // 0-5 minutes random
int finalTTL = baseTTL + jitter;
```

---

### TTI — Time-to-Idle

Entry expires if it **hasn't been accessed** within the idle duration. Resets on every access.

**Example:** Session cache with 30-min TTI — session stays alive as long as user is active, expires 30 min after last activity.

---

### Size-Based Eviction

Limit cache by **number of entries** or **total memory**:

```java
Caffeine.newBuilder()
    .maximumSize(10_000)         // Max 10,000 entries
    // OR
    .maximumWeight(100_000_000)  // Max ~100MB (with custom weigher)
    .build();
```

---

## Phase 7: Cache Invalidation (The Hard Problem)

> *When the source data changes, how do you ensure the cache doesn't serve stale data?*

This is the **hardest problem in caching** because there's no perfect solution — only tradeoffs.

### Approach 1: TTL-Based Expiry

Just let cached entries expire naturally after a set time.

```
Data updated in DB at T=0
Cache still serves old data until TTL expires at T=300 (5 min)
```

**Staleness window:** Up to the full TTL duration.
**When acceptable:** Reference data, dashboards, non-critical reads.

---

### Approach 2: Event-Based Invalidation

When data changes, **actively delete or update** the cache entry.

```java
// Service that updates a carrier
public void updateCarrier(Carrier carrier) {
    carrierRepository.save(carrier);
    redisTemplate.delete("carrier:" + carrier.getId());  // Invalidate immediately
}
```

**Pros:** Near-instant consistency.
**Cons:** Every write path must know about cache invalidation — easy to miss one.

---

### Approach 3: Pub/Sub Invalidation (Distributed Systems)

When one service updates data, it **publishes an event**. All services that cache that data **subscribe and invalidate**.

```
Service A updates carrier in DB
    → Publishes event: "carrier:MAERSK:updated" to Kafka/Redis Pub-Sub
    
Service B (caches carrier data) receives event
    → Deletes "carrier:MAERSK" from its local cache
    
Service C (also caches carrier data) receives event
    → Deletes "carrier:MAERSK" from its local cache
```

**Best for:** Microservices architecture where multiple services cache the same data.

---

### Approach 4: Versioned Keys

Instead of invalidating, **write new data under a new key version**:

```
v1: carrier:MAERSK:v1 → { name: "Maersk", ... }
v2: carrier:MAERSK:v2 → { name: "Maersk Line", ... }
```

Readers always request the latest version. Old versions expire naturally.
**Best for:** Immutable data, content versioning.

---

### The Delete vs Update Debate

| Approach | Pros | Cons |
|---|---|---|
| **Delete on write** | Simple, no stale data served twice | Next read is a cache miss |
| **Update on write** | Next read is a hit | Race condition if concurrent writes |

**Recommendation:** Prefer **delete on write** — it's simpler and avoids race conditions. The next read will repopulate the cache.

---

## Phase 8: Cache Consistency and Correctness

For a logistics system, caching incorrectly can cause **real business damage** — wrong shipment status, wrong freight rates, wrong inventory counts.

### Freshness vs Latency Tradeoff

```
Strong Consistency ◄────────────────────► Low Latency
(always correct,        (might be stale,
 higher latency)         but fast)
```

**You must decide per data type:**

| Data | Consistency Need | Cache Strategy |
|---|---|---|
| Shipment tracking status | **Strong** — customers see it | Very short TTL (30s) or no cache |
| Port codes / carrier master | **Eventual** — rarely changes | Long TTL (24h), event-based invalidation |
| Freight rate quotes | **Careful** — version by effective date | `rate:{carrier}:{route}:{effectiveDate}` |
| Inventory / available capacity | **Strong** — over-selling is costly | No cache, or write-through with short TTL |
| Dashboard aggregates | **Eventual** — slight delay ok | 5-minute TTL |

### Read-Your-Writes Consistency

**Problem:** User updates shipment status → immediately views the shipment → sees the OLD status from cache.

**Solution:**
```java
// After write, force read from DB (bypass cache) for the writing user
public Shipment getShipment(String shipmentId, boolean forceRefresh) {
    if (forceRefresh) {
        Shipment shipment = shipmentRepository.findById(shipmentId);
        cache.put(shipmentId, shipment);  // Update cache
        return shipment;
    }
    return cache.get(shipmentId);  // Normal cached read
}
```

### Stale Reads

**Problem:** Cache serves data that's already been updated in the DB.
**Mitigations:**
1. **Short TTL** for frequently changing data
2. **Event-based invalidation** on every write
3. **Version stamps** — client sends last-known version, server checks if cache is newer

---

## Phase 9: Distributed Caching in Production

### Redis Architecture

**Single Instance:**
```
App → Redis (single node)
```
Simple but single point of failure. Good for development only.

**Redis Sentinel (High Availability):**
```
App → Sentinel → Redis Master ←→ Redis Replica 1
                              ←→ Redis Replica 2
```
Sentinel monitors the master. If master dies, sentinel promotes a replica automatically.

**Redis Cluster (Horizontal Scaling):**
```
App → Redis Cluster
      ├── Shard 1 (keys A-F) → Master + Replica
      ├── Shard 2 (keys G-M) → Master + Replica
      └── Shard 3 (keys N-Z) → Master + Replica
```
Data is automatically partitioned across shards using hash slots (16384 slots).

---

### Redis vs Memcached

| Feature | Redis | Memcached |
|---|---|---|
| Data Structures | Strings, Lists, Sets, Hashes, Sorted Sets | Strings only |
| Persistence | Yes (RDB/AOF) | No |
| Replication | Yes (Master-Replica) | No |
| Clustering | Yes (Redis Cluster) | Client-side sharding |
| Pub/Sub | Yes | No |
| Lua Scripting | Yes | No |
| Memory Efficiency | Lower | Higher |
| **When to use** | Most use cases, need persistence/structures | Simple key-value, maximum memory efficiency |

**Recommendation for Spring Boot:** Use **Redis** — it's the default cache provider, has richer features, and Spring has excellent integration.

---

### Production Problem 1: Thundering Herd / Cache Stampede

**Problem:** A popular cache key expires → hundreds of concurrent requests all get a cache miss → all simultaneously query the database → DB overwhelmed.

```
Cache key "popular-carrier" expires at T=100
T=100.001: Request 1 → MISS → queries DB
T=100.002: Request 2 → MISS → queries DB
T=100.003: Request 3 → MISS → queries DB
... 500 requests all hit DB simultaneously
```

**Solutions:**

**1. Distributed Lock (Mutex):**
```java
public Carrier getCarrier(String id) {
    Carrier carrier = cache.get(id);
    if (carrier == null) {
        // Only one thread fetches from DB
        if (redisLock.tryLock("lock:carrier:" + id, Duration.ofSeconds(5))) {
            try {
                carrier = carrierRepository.findById(id);
                cache.put(id, carrier);
            } finally {
                redisLock.unlock("lock:carrier:" + id);
            }
        } else {
            // Other threads wait and retry
            Thread.sleep(50);
            return cache.get(id);
        }
    }
    return carrier;
}
```

**2. Probabilistic Early Expiry:**
Refresh the cache *before* it actually expires — a random subset of requests trigger background refresh.

---

### Production Problem 2: Hot Key

**Problem:** A single key is accessed so frequently that the Redis shard hosting it becomes a bottleneck.

**Example:** During peak season, `carrier:MAERSK` is queried 100,000 times/second.

**Solutions:**
- **Local (L1) cache** in front of Redis — Caffeine absorbs most reads
- **Key replication** — Store copies as `carrier:MAERSK:1`, `carrier:MAERSK:2`, distribute reads randomly

---

### Production Problem 3: Cache Penetration

**Problem:** Requests for **data that doesn't exist** always result in cache miss → always hit DB.

**Example:** Malicious user queries `shipment:FAKE-ID-12345` repeatedly. It's never in cache, never in DB, but every request hits the database.

**Solutions:**

**1. Negative Caching** — Cache the "not found" result with a **short TTL**:
```java
Shipment shipment = shipmentRepository.findById(id);
if (shipment == null) {
    cache.put("shipment:" + id, "NOT_FOUND", Duration.ofMinutes(5));  // Cache the miss
    return null;
}
```

**2. Bloom Filter** — A probabilistic data structure that quickly tells you "definitely NOT in DB" vs "maybe in DB":
```
Request for shipment:XYZ
    → Check Bloom Filter: "Is XYZ possibly in DB?"
    → NO → Return 404 immediately (don't hit cache or DB)
    → YES → Proceed to cache/DB lookup
```

---

### Production Problem 4: Cache Avalanche

**Problem:** Large number of cache keys **expire at the same time** → massive DB load spike.

**Example:** You load 10,000 carrier records into cache at startup, all with TTL=3600s. Exactly 1 hour later, all 10,000 expire simultaneously.

**Solution: Jittered TTL**
```java
int baseTTL = 3600;
int jitter = ThreadLocalRandom.current().nextInt(0, 600);  // 0-10 min random
cache.put(key, value, Duration.ofSeconds(baseTTL + jitter));
```

Now expiries are spread over a 10-minute window instead of a single second.

---

### Cache Warming

**Problem:** After a deployment or cache restart, the cache is empty → all requests hit DB → slow performance.

**Solution:** Preload frequently accessed data into cache at startup:
```java
@EventListener(ApplicationReadyEvent.class)
public void warmCache() {
    List<Carrier> carriers = carrierRepository.findAllActive();
    carriers.forEach(c -> cache.put("carrier:" + c.getId(), c));
    
    List<Port> ports = portRepository.findAll();
    ports.forEach(p -> cache.put("port:" + p.getCode(), p));
    
    log.info("Cache warmed: {} carriers, {} ports", carriers.size(), ports.size());
}
```

---

## Phase 10: Caching in Spring Boot (Practical Deep Dive)

### Basic Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  cache:
    type: redis
  redis:
    host: localhost
    port: 6379
    timeout: 2000ms
```

```java
@SpringBootApplication
@EnableCaching  // Enables Spring's caching infrastructure
public class Application { }
```

### Core Annotations

**@Cacheable** — Cache the result of a method:
```java
@Cacheable(value = "carriers", key = "#carrierId", unless = "#result == null")
public Carrier findCarrier(String carrierId) {
    return carrierRepository.findById(carrierId).orElse(null);
}
// First call: executes method, caches result
// Subsequent calls: returns cached result (method NOT executed)
```

**@CacheEvict** — Remove entry from cache:
```java
@CacheEvict(value = "carriers", key = "#carrier.id")
public void updateCarrier(Carrier carrier) {
    carrierRepository.save(carrier);
}

// Evict ALL entries in a cache:
@CacheEvict(value = "carriers", allEntries = true)
public void refreshAllCarriers() { }
```

**@CachePut** — Always execute method AND update cache:
```java
@CachePut(value = "carriers", key = "#carrier.id")
public Carrier updateAndReturn(Carrier carrier) {
    return carrierRepository.save(carrier);
}
// Unlike @Cacheable, the method ALWAYS executes
```

### Conditional Caching

```java
@Cacheable(
    value = "shipments",
    key = "#shipmentId",
    condition = "#shipmentId.length() > 3",    // Only cache if condition is true
    unless = "#result.status == 'IN_TRANSIT'"   // Don't cache in-transit shipments (too dynamic)
)
public Shipment getShipment(String shipmentId) { ... }
```

### TTL Per Cache (Redis Configuration)

```java
@Configuration
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeValuesWith(SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));

        Map<String, RedisCacheConfiguration> cacheConfigs = Map.of(
            "carriers", defaultConfig.entryTtl(Duration.ofHours(6)),     // Rarely changes
            "ports", defaultConfig.entryTtl(Duration.ofHours(24)),       // Almost never changes
            "shipments", defaultConfig.entryTtl(Duration.ofMinutes(2)),  // Changes frequently
            "rates", defaultConfig.entryTtl(Duration.ofMinutes(15))      // Moderate change frequency
        );

        return RedisCacheManager.builder(factory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigs)
            .build();
    }
}
```

### ⚠️ The Self-Invocation Trap

**Problem:** `@Cacheable` uses Spring AOP proxies. If a method calls another method **in the same class**, the proxy is bypassed and caching doesn't work.

```java
@Service
public class ShipmentService {

    // ❌ THIS WON'T CACHE — self-invocation bypasses proxy
    public ShipmentDetails getDetails(String id) {
        Shipment shipment = getShipment(id);  // Calls method in same class!
        return mapToDetails(shipment);
    }

    @Cacheable("shipments")
    public Shipment getShipment(String id) {
        return shipmentRepository.findById(id);
    }
}
```

**Fix:** Extract the cached method into a **separate service class**, or inject the service into itself via `@Lazy`.

### Multi-Level Cache (Caffeine + Redis)

```java
@Bean
public CacheManager cacheManager(RedisConnectionFactory redisFactory) {
    // L1: Fast in-process cache
    CaffeineCacheManager l1 = new CaffeineCacheManager();
    l1.setCaffeine(Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(Duration.ofMinutes(5)));

    // L2: Shared distributed cache
    RedisCacheManager l2 = RedisCacheManager.builder(redisFactory)
        .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1)))
        .build();

    // Composite: checks L1 first, then L2
    return new CompositeCacheManager(l1, l2);
}
```

---

## Phase 11: When Caching is an Overhead (Anti-Patterns)

Not everything needs caching. Here are scenarios where caching **hurts more than it helps**:

### Scenario 1: Read-Once, Write-Once Data

**Example:** An OTP verification API. Each OTP is generated, sent to the user, verified once, and never accessed again.

```
POST /otp/generate → creates OTP, stores in DB
POST /otp/verify   → reads OTP once, marks used
```

Caching the OTP would waste memory since it's read exactly once. The cache hit ratio would be ~0%.

---

### Scenario 2: Highly Dynamic, Per-Request Data

**Example:** A real-time bidding system for freight spot rates. Every request gets a unique price based on current supply/demand.

```
GET /spot-rate?origin=INMAA&dest=USLAX
→ Price changes every second based on live market conditions
→ Cached value is stale before it's ever read again
```

Caching this adds **latency** (cache check) and **memory** for data that's never reused.

---

### Scenario 3: Low-Traffic APIs

**Example:** An internal admin API used by 3 people, called 10 times a day. Database response time is 5ms.

```
Without cache: 5ms response
With cache:    2ms response + cache infrastructure + consistency complexity

Saving 3ms × 10 requests/day = 30ms saved per day.
Cost: Redis instance + monitoring + cache invalidation logic.
```

**Not worth it.** The complexity outweighs the benefit.

---

### Scenario 4: Write-Heavy, Read-Rare Data

**Example:** A shipment event logging system. Events are written 1000x/sec but only read when someone views the audit trail (once a week).

```
Events/day:     ~86M writes
Reads/day:      ~100
Cache hit ratio: ~0.0001%
```

Caching these events wastes memory. The writes would constantly invalidate the cache before anyone reads it.

---

### Scenario 5: Logistics Tracking Hit Once Per Event

**Example:** Each shipment milestone (picked up, departed port, arrived port, delivered) triggers one API call to update and one webhook to notify the customer.

```
Shipment SHP-001:
  Event: DEPARTED_ORIGIN → API hit once → webhook sent → never queried from cache
  Event: ARRIVED_DEST    → API hit once → webhook sent → never queried from cache
```

Each unique event is processed **exactly once**. Caching it is a pure overhead — memory allocated for data that's never reused.

---

### Scenario 6: Small Dataset That Fits in DB Memory

**Example:** A lookup table of 50 carrier codes. The entire table fits in the database's buffer pool (InnoDB keeps it in RAM automatically).

```
DB response time: 1ms (already from buffer pool)
Cache response time: 0.5ms (Redis)
```

Adding Redis to save 0.5ms adds infrastructure complexity for negligible gain. The database is already acting as an efficient cache.

---

### The Cache Overhead Checklist

Before adding caching, ask yourself:

- [ ] **Is this data read more than once?** → If no, don't cache
- [ ] **Is the hit ratio likely > 80%?** → If no, reconsider
- [ ] **Is the source latency actually a problem?** → If no, don't cache
- [ ] **Can I tolerate stale data?** → If no, caching is risky
- [ ] **Does the traffic justify the complexity?** → If no, don't cache

> **Rule of thumb:** If you can't explain *which specific performance problem* caching solves, you probably don't need it.

---

## Phase 12: Cache Observability and Tuning

You can't improve what you can't measure.

### Key Metrics to Monitor

| Metric | What It Tells You | Target |
|---|---|---|
| **Hit Ratio** | % of requests served from cache | > 90% |
| **Miss Ratio** | % of requests that go to DB | < 10% |
| **Eviction Count** | How often entries are removed for space | Should be low and steady |
| **Latency (p50/p95/p99)** | Cache response times | < 1ms for Redis |
| **Memory Usage** | How much RAM the cache consumes | Set alerts at 80% |
| **Expired Key Count** | TTL-based removals | Expected; watch for spikes |

### Monitoring in Spring Boot

```java
// Expose cache metrics via Actuator + Micrometer
management:
  endpoints:
    web:
      exposure:
        include: caches, metrics, prometheus
```

Access `GET /actuator/metrics/cache.gets` to see hit/miss counts per cache.

### When to Tune

| Observation | Action |
|---|---|
| Hit ratio < 80% | TTL too short? Wrong data being cached? |
| High eviction rate | Increase cache size or reduce TTL |
| Memory growing unbounded | Set `maxmemory` policy in Redis |
| Latency spikes | Hot key? Network issue? Serialization overhead? |

---

## Phase 13: Security and Data Sensitivity

### Rules for Caching Sensitive Data

| Data Type | Cache? | Precaution |
|---|---|---|
| PII (names, addresses) | With caution | Encrypt at rest, short TTL, no logs |
| Auth tokens / sessions | Yes (Redis) | Set TTL = session timeout, secure connection |
| Payment details | **Never** | Compliance violation risk (PCI-DSS) |
| API keys / secrets | **Never** | Use a secrets manager instead |
| Multi-tenant data | Yes | **Always scope key by tenant** |

### Multi-Tenant Cache Isolation

```java
// ❌ DANGEROUS — Tenant A can see Tenant B's data
String key = "shipment:" + shipmentId;

// ✅ SAFE — Scoped by tenant
String key = "tenant:" + tenantId + ":shipment:" + shipmentId;
```

---

## Real-World Case Studies

### Case 1: Logistics — Carrier Master Data Caching

**Problem:** A freight platform queries carrier data on every booking. 500 bookings/min × DB query per booking = unnecessary load.

**Solution:**
- Cache all carrier data in Redis with 6-hour TTL
- L1 Caffeine cache per instance for ultra-hot carriers (MAERSK, MSC, CMA CGM)
- Event-based invalidation when carrier data is updated via admin panel

**Result:** DB queries for carrier data dropped by 99.5%.

### Case 2: Logistics — Rate Card Lookup

**Problem:** Freight rate calculation involves complex joins across 5 tables. Takes 50ms per query.

**Solution:**
- Cache computed rates with key: `rate:{carrier}:{origin}:{dest}:{effectiveDate}`
- TTL = 15 minutes (rates don't change more than a few times per day)
- On tariff update → `@CacheEvict(value = "rates", allEntries = true)`

**Result:** Average rate lookup dropped from 50ms to 1ms for cached routes.

### Case 3: Logistics — Shipment Tracking (Cache with Caution)

**Problem:** Customers refresh tracking pages frequently, but showing stale status is unacceptable.

**Solution:**
- Cache tracking data with **30-second TTL** only
- On status update event → immediately invalidate the cache
- Show "Last updated: X seconds ago" on the UI to set customer expectations

**Result:** 85% cache hit ratio with maximum 30-second staleness — acceptable for the business.

---

## Key Takeaways

1. **Cache the right data** — high-read, low-write, expensive-to-compute
2. **Choose the right layer** — client, CDN, in-process, distributed, or multi-level
3. **Pick the right strategy** — Cache-Aside for most cases, Write-Behind for write-heavy
4. **Design keys carefully** — specific, scoped, versioned, with TTL
5. **Invalidation is hard** — prefer TTL + event-based combo
6. **Monitor everything** — hit ratio, evictions, latency, memory
7. **Don't cache everything** — when hit ratio is low or data is read-once, caching is overhead
8. **Secure your cache** — tenant isolation, encrypt PII, never cache secrets

---

**Next up:** [[Message Queues]]
