# Bloom Filters and Probabilistic Data Structures

> *"Sometimes it's better to be probably right than certainly slow."*

---

## Phase 1: The Problem — Membership Testing at Scale

### The Core Question

You have a massive dataset (billions of records). You need to answer one simple question **very fast**:

> "Is this element in the set?"

```
Examples:
  "Has this username been taken?"                 (billions of usernames)
  "Is this shipment ID in our system?"            (hundreds of millions of IDs)
  "Has this user already seen this notification?" (billions of user-notification pairs)
  "Is this IP address in our blocklist?"          (millions of IPs)
  "Has this URL been crawled?"                    (billions of URLs)
```

### Why Not Just Use a HashSet?

```
100 million shipment IDs, each 20 characters:
  HashSet memory: ~100M × (20 bytes + 32 bytes overhead) ≈ 5 GB RAM

1 billion URLs:
  HashSet memory: ~1B × (100 bytes + 32 bytes overhead) ≈ 120 GB RAM

That's expensive! And you only need a YES/NO answer.
```

### The Tradeoff: Exact vs Probabilistic

| Approach | Memory | Speed | Accuracy |
|---|---|---|---|
| **HashSet** | O(n) — huge | O(1) | 100% exact |
| **Database query** | Disk-based | O(log n) — slow | 100% exact |
| **Bloom Filter** | O(1) — tiny! | O(1) — fast! | Probabilistic (small false positive rate) |

**Bloom Filters trade a tiny amount of accuracy for massive memory savings.**

---

## Phase 2: Bloom Filters

### What is a Bloom Filter?

A **Bloom Filter** is a space-efficient probabilistic data structure that tests whether an element is **possibly in a set** or **definitely not in a set**.

```
Bloom Filter answers:
  "Is X in the set?"
    → "DEFINITELY NOT" (100% certain)  — item is guaranteed not present
    → "PROBABLY YES"   (might be wrong) — item is probably present (small chance of false positive)
```

**Key property:** 
- **No false negatives** — if the filter says "not present", it's DEFINITELY not present
- **Possible false positives** — if the filter says "present", it MIGHT be wrong (but rarely)

### How It Works

A Bloom Filter is a **bit array of m bits** (all initially 0) with **k hash functions**.

**Adding an element:**
```
Element: "SHP-001"
k = 3 hash functions

hash1("SHP-001") % m = position 3  → set bit 3 to 1
hash2("SHP-001") % m = position 7  → set bit 7 to 1
hash3("SHP-001") % m = position 11 → set bit 11 to 1

Bit array: [0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0]
                      ↑              ↑              ↑
                     bit 3          bit 7          bit 11
```

**Checking membership:**
```
"Is SHP-001 in the set?"
  hash1("SHP-001") % m = 3  → bit 3 = 1 ✅
  hash2("SHP-001") % m = 7  → bit 7 = 1 ✅
  hash3("SHP-001") % m = 11 → bit 11 = 1 ✅
  All bits are 1 → "PROBABLY YES" ✅

"Is SHP-999 in the set?"
  hash1("SHP-999") % m = 3  → bit 3 = 1 ✅
  hash2("SHP-999") % m = 5  → bit 5 = 0 ❌
  At least one bit is 0 → "DEFINITELY NOT" ✅ (guaranteed!)
```

**Why false positives happen:**
```
After many insertions, many bits are set to 1.
A new element might hash to positions that are ALL already 1 — 
  set by OTHER elements, not by this element.
  → Bloom filter says "probably yes" but it's actually not in the set.
```

### Visual Example

```
Insert "SHP-001": bits 3, 7, 11 → 1
Insert "SHP-002": bits 1, 7, 14 → 1
Insert "SHP-003": bits 3, 5, 9  → 1

Bit array: [0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 0, 1]

Check "SHP-999": hashes to bits 1, 3, 5
  Bit 1 = 1 ✅ (set by SHP-002)
  Bit 3 = 1 ✅ (set by SHP-001 and SHP-003)
  Bit 5 = 1 ✅ (set by SHP-003)
  
  All 1s → "PROBABLY YES" 
  But SHP-999 was NEVER inserted! → FALSE POSITIVE
```

### Tuning: Size and Hash Functions

**False positive probability formula:**

```
p ≈ (1 - e^(-kn/m))^k

Where:
  m = number of bits in the array
  k = number of hash functions
  n = number of elements inserted
  p = false positive probability
```

**Practical guidelines:**

| Elements (n) | Desired FP Rate | Bits (m) | Hash Functions (k) | Memory |
|---|---|---|---|---|
| 1 million | 1% | 9.6M bits | 7 | **1.2 MB** |
| 1 million | 0.1% | 14.4M bits | 10 | 1.8 MB |
| 100 million | 1% | 960M bits | 7 | **120 MB** |
| 1 billion | 1% | 9.6B bits | 7 | **1.2 GB** |

Compare: a HashSet of 1 billion strings would use ~100 GB. A Bloom Filter uses **1.2 GB** for 1% false positive rate. **~80x memory savings.**

### Bloom Filter Limitations

| Limitation | Explanation |
|---|---|
| **No deletion** | Can't remove elements (clearing a bit might affect other elements) |
| **No enumeration** | Can't list what's in the set |
| **No counting** | Can't tell how many times an element was added |
| **Fixed capacity** | Must estimate size upfront; resizing requires rebuilding |
| **False positives** | Rate increases as more elements are added |

---

## Phase 3: Bloom Filter Variants

### Counting Bloom Filter

Replaces each bit with a **counter** (typically 4 bits). Allows deletion.

```
Standard:  bit array [0, 1, 0, 1, 1]
Counting:  counter array [0, 2, 0, 1, 3]

Insert: increment counters at hash positions
Delete: decrement counters at hash positions
Check:  all counters > 0? → probably yes
```

**Tradeoff:** 4x more memory than standard Bloom Filter.

---

### Cuckoo Filter

An alternative to Bloom Filters that supports **deletion** and has better space efficiency for low false positive rates.

```
Instead of bits, stores fingerprints in a hash table with cuckoo hashing.

Insert: Store fingerprint at one of two candidate positions
Delete: Remove fingerprint from its position
Lookup: Check both candidate positions for the fingerprint
```

**Pros:** Supports deletion, better performance at low FP rates (<3%).
**Cons:** Fixed capacity (can fail to insert when full).

---

### Scalable Bloom Filter

Automatically **grows** as more elements are added by chaining multiple Bloom Filters.

```
When filter 1 reaches target capacity:
  → Create filter 2 with tighter FP rate
  → New inserts go to filter 2
  → Lookups check all filters

Filter 1: 1M elements, 1% FP
Filter 2: 2M elements, 0.5% FP
Filter 3: 4M elements, 0.25% FP

Combined FP rate stays within bounds.
```

---

## Phase 4: Bloom Filters in Practice

### Use Case 1: Cache Penetration Prevention

As discussed in [[Caching]] — prevent queries for non-existent data from hitting the database.

```java
// At startup, load all existing shipment IDs into Bloom Filter
BloomFilter<String> shipmentFilter = BloomFilter.create(
    Funnels.stringFunnel(Charset.defaultCharset()),
    10_000_000,  // expected insertions
    0.01          // 1% false positive rate
);
shipmentRepository.findAllIds().forEach(shipmentFilter::put);

// On API request
public Shipment getShipment(String id) {
    // Step 1: Bloom Filter check (instant, no DB call)
    if (!shipmentFilter.mightContain(id)) {
        return null;  // DEFINITELY not in DB — skip cache and DB entirely
    }
    
    // Step 2: Check cache
    Shipment cached = cache.get(id);
    if (cached != null) return cached;
    
    // Step 3: Query DB (only reached for real IDs + rare false positives)
    return shipmentRepository.findById(id);
}
```

**Impact:** Malicious requests for fake shipment IDs are blocked at the Bloom Filter level — no cache pollution, no DB load.

---

### Use Case 2: Duplicate Detection

Detect duplicate events in a high-throughput stream.

```java
// Webhook deduplication
BloomFilter<String> processedEvents = BloomFilter.create(
    Funnels.stringFunnel(Charset.defaultCharset()),
    50_000_000,  // 50M events per day
    0.001         // 0.1% false positive rate
);

public void handleWebhook(WebhookEvent event) {
    if (processedEvents.mightContain(event.getEventId())) {
        log.info("Duplicate event {}, skipping", event.getEventId());
        return;  // Probably already processed
    }
    
    processEvent(event);
    processedEvents.put(event.getEventId());
}
```

**Note:** The 0.1% false positive rate means 1 in 1,000 unique events might be falsely skipped. For non-critical deduplication, this is acceptable.

---

### Use Case 3: Username / Email Availability Check

```
User types "john@acme.com" in signup form:
  → Bloom Filter check: "Definitely not taken" → show green checkmark ✅
  → Bloom Filter check: "Might be taken" → query database to confirm
  
Most checks resolve instantly without a DB query.
```

---

### Use Case 4: Web Crawler URL Deduplication

```
Crawler has visited 10 billion URLs.
New URL found: "Is this already crawled?"
  → Bloom Filter: "Definitely not" → crawl it
  → Bloom Filter: "Might be" → skip (accept rare false positive = skip a few valid URLs)
```

---

### Redis Bloom Filter

```bash
# Redis has a Bloom Filter module (RedisBloom)
BF.RESERVE shipment-filter 0.01 10000000  # 1% FP, 10M capacity
BF.ADD shipment-filter "SHP-001"
BF.ADD shipment-filter "SHP-002"
BF.EXISTS shipment-filter "SHP-001"  → 1 (probably exists)
BF.EXISTS shipment-filter "SHP-999"  → 0 (definitely not)
```

---

## Phase 5: HyperLogLog — Cardinality Estimation

### The Problem

"How many **unique** visitors accessed our tracking portal today?"

```
Exact counting:
  Store every unique visitor ID in a HashSet.
  100 million unique visitors × 16 bytes each = 1.6 GB RAM

HyperLogLog:
  Estimates unique count using ~12 KB of memory.
  Accuracy: ±0.81% (99.19% accurate)
```

### How It Works (Simplified)

HyperLogLog uses the statistical properties of hash values:

```
Hash each element. Look at the binary representation.
Count leading zeros in the hash.

hash("user-001") = 0001... (3 leading zeros)
hash("user-002") = 0000001... (6 leading zeros)
hash("user-003") = 01... (1 leading zero)

The MORE leading zeros you observe, the MORE unique elements there are.
(Seeing 20 leading zeros means you've hashed ~2^20 ≈ 1 million elements)

HLL uses multiple "registers" and takes the harmonic mean for accuracy.
```

### Redis HyperLogLog

```bash
# Count unique tracking page visitors
PFADD tracking:visitors:2024-01-15 "user-001" "user-002" "user-003"
PFADD tracking:visitors:2024-01-15 "user-001"  # Duplicate, not counted

PFCOUNT tracking:visitors:2024-01-15  → 3 (approximate unique count)

# Merge counts across days
PFMERGE tracking:visitors:week tracking:visitors:2024-01-15 tracking:visitors:2024-01-16
PFCOUNT tracking:visitors:week  → approximate unique visitors for the week
```

**Memory:** Each HyperLogLog key uses at most **12 KB**, regardless of how many elements you add. You could count 1 billion unique elements in 12 KB.

### Use Cases

| Use Case | Why HyperLogLog |
|---|---|
| Unique visitors per page | Can't store billions of user IDs |
| Unique search queries per day | Memory-efficient counting |
| Unique IP addresses accessing an API | DDoS detection (approximate is fine) |
| Unique shipments tracked per customer | Dashboard metric |

---

## Phase 6: Count-Min Sketch — Frequency Estimation

### The Problem

"What are the **most frequently accessed** shipment IDs in the last hour?"

```
Exact counting:
  HashMap<String, Integer> — one entry per unique key
  100 million unique shipments × ~50 bytes each = 5 GB RAM

Count-Min Sketch:
  Estimates frequency using a fixed-size 2D array
  Memory: configurable (e.g., 1 MB for millions of items)
  Accuracy: never underestimates, may slightly overestimate
```

### How It Works

A **2D array** of counters (d rows × w columns) with d hash functions.

```
d = 4 hash functions, w = 10 columns

Insert "SHP-001":
  hash1("SHP-001") % 10 = 3 → row 0, col 3: increment
  hash2("SHP-001") % 10 = 7 → row 1, col 7: increment
  hash3("SHP-001") % 10 = 1 → row 2, col 1: increment
  hash4("SHP-001") % 10 = 5 → row 3, col 5: increment

Query "SHP-001" frequency:
  Check row 0 col 3: 15
  Check row 1 col 7: 12
  Check row 2 col 1: 18
  Check row 3 col 5: 14
  
  Estimated frequency = MIN(15, 12, 18, 14) = 12
  (MIN because collisions only increase counts, never decrease)
```

### Use Cases

| Use Case | Why Count-Min Sketch |
|---|---|
| **Hot key detection** | Find most accessed cache keys in Redis |
| **Top-N queries** | "Most tracked shipments this hour" |
| **Network traffic** | Detect heavy-hitter IP addresses |
| **Click counting** | Approximate ad click frequency |

---

## Phase 7: Consistent Hashing

Covered in detail in [[Hash Ring]] and [[Sharding and Replication]], but here's the data structure perspective.

### The Problem

Distributing data across N nodes using `hash(key) % N`. When you add/remove a node, almost ALL keys need to be remapped.

```
3 nodes: hash(key) % 3
  "SHP-001" → node 1
  "SHP-002" → node 0
  "SHP-003" → node 2

Add a 4th node: hash(key) % 4
  "SHP-001" → node 3  ← MOVED
  "SHP-002" → node 2  ← MOVED  
  "SHP-003" → node 2  ← same (lucky)

~75% of keys need to move! Massive data migration.
```

### Consistent Hashing Solution

Arrange nodes on a **hash ring** (0 to 2^32). Each key hashes to a position on the ring and is assigned to the **next node clockwise**.

```
Ring positions:
  Node A at position 1000
  Node B at position 5000
  Node C at position 9000

  Key "SHP-001" hashes to 3000 → assigned to Node B (next clockwise)
  Key "SHP-002" hashes to 7000 → assigned to Node C

Adding Node D at position 7500:
  Only keys between 7000-7500 move from Node C to Node D
  Everything else stays! (~25% of Node C's keys move, other nodes unaffected)
```

### Virtual Nodes

Problem: With few physical nodes, data may be unevenly distributed.

Solution: Each physical node gets **multiple positions** (virtual nodes) on the ring.

```
Physical Node A → Virtual nodes at positions: 1000, 4000, 8000
Physical Node B → Virtual nodes at positions: 2000, 6000, 10000

More virtual nodes → more even distribution of keys.
```

**Used by:** Cassandra (default), DynamoDB, Redis Cluster, Memcached (client-side).

---

## Phase 8: When Probabilistic Data Structures Are Overhead

### Scenario 1: Small Datasets

If your set has 10,000 elements, a HashSet uses ~500 KB. A Bloom Filter saves no meaningful memory. Use the exact data structure.

---

### Scenario 2: Zero Tolerance for Errors

If even a 0.01% false positive rate is unacceptable (e.g., "Is this financial transaction ID already processed?"), use exact data structures.

---

### Scenario 3: Need to Delete / Update

Standard Bloom Filters can't delete. If your use case requires removing elements, use a Counting Bloom Filter, Cuckoo Filter, or just a HashSet.

---

### Scenario 4: Need Exact Counts

If you need the EXACT number of unique visitors (not an estimate), HyperLogLog won't work. Use a database `COUNT(DISTINCT ...)`.

---

## Phase 9: Comparison Table

| Data Structure | Question Answered | Space | False Positives | False Negatives | Deletion |
|---|---|---|---|---|---|
| **Bloom Filter** | "Is X in the set?" | Very small | Yes (small %) | No | No |
| **Counting Bloom** | "Is X in the set?" | Small | Yes | No | Yes |
| **Cuckoo Filter** | "Is X in the set?" | Small | Yes (lower) | No | Yes |
| **HyperLogLog** | "How many unique Xs?" | 12 KB fixed | N/A | N/A | N/A |
| **Count-Min Sketch** | "How often is X?" | Configurable | Overestimates | N/A | N/A |
| **Consistent Hashing** | "Which node for X?" | O(nodes) | N/A | N/A | Yes |

---

## Key Takeaways

1. **Bloom Filters** answer "is X in the set?" with zero false negatives and configurable false positive rate
2. **80x memory savings** vs HashSet for membership testing at scale
3. **Cache penetration prevention** is the #1 use case for Bloom Filters in system design
4. **HyperLogLog** counts unique elements in 12 KB — use for analytics (unique visitors, unique queries)
5. **Count-Min Sketch** estimates frequency — use for hot key detection and top-N queries
6. **Consistent Hashing** minimizes data movement when adding/removing nodes — backbone of distributed systems
7. **Don't over-optimize** — for small datasets (<100K elements), exact data structures are fine
8. **Redis supports all of these** — Bloom Filters (RedisBloom), HyperLogLog (PFADD/PFCOUNT), Sorted Sets for top-N

---

**See also:** [[Caching]] (cache penetration + Bloom Filters), [[Hash Ring]] (consistent hashing), [[Sharding and Replication]] (consistent hashing for sharding)
