# Rate Limiting and Throttling

> *"The fastest way to break a system is to let everyone use it as much as they want."*

---

## Phase 1: The Fundamentals

### What is Rate Limiting?

Rate limiting is the technique of **controlling the number of requests** a client can make to an API within a given time window. Requests beyond the limit are rejected.

```
Without rate limiting:
  Client sends 10,000 requests/sec → Server overwhelmed → Crashes → ALL users affected

With rate limiting:
  Client sends 10,000 requests/sec → Rate limiter allows 100/sec → 9,900 rejected (429)
  → Server stays healthy → Other users unaffected
```

**Analogy:** Think of a **port terminal with loading bays**. Only 10 trucks can load at the same time. If 50 trucks arrive, 40 wait in a queue. If the queue is full, trucks are turned away. The terminal never gets overloaded.

### Rate Limiting vs Throttling

| Concept | Behavior | Example |
|---|---|---|
| **Rate Limiting** | Hard reject after limit | 101st request gets `429 Too Many Requests` |
| **Throttling** | Slow down / degrade gracefully | 101st request is queued or served with reduced data |

In practice, these terms are often used interchangeably. Most systems implement rate limiting (hard reject) because throttling (queueing) can lead to unbounded memory usage.

### Why Do We Need Rate Limiting?

| Problem | How Rate Limiting Helps |
|---|---|
| **DDoS attacks** | Blocks flood of malicious requests |
| **Misbehaving clients** | A buggy client in an infinite loop doesn't take down the system |
| **Fair usage** | One client can't monopolize resources at others' expense |
| **Cost control** | Prevents runaway API calls from generating massive cloud bills |
| **Service protection** | Keeps downstream services (DB, cache, 3rd party APIs) within their capacity |
| **SLA enforcement** | Free tier: 100 req/min, Pro tier: 10,000 req/min |

**Logistics example:** A carrier's webhook integration sends shipment status updates. A bug causes it to send the same update 10,000 times/sec. Without rate limiting, your tracking service, database, and notification service are all overwhelmed. With rate limiting, you accept 100/sec and reject the rest — the carrier can debug their issue without impacting your platform.

---

## Phase 2: Rate Limiting Dimensions

### What Do We Rate Limit By?

| Dimension | Use Case | Example |
|---|---|---|
| **Per user** | Prevent one user from hogging resources | User `john@acme.com` → 1,000 req/hour |
| **Per API key** | Control usage by application | API key `ak_12345` → 500 req/min |
| **Per IP address** | Block abusive IPs | IP `1.2.3.4` → 100 req/min |
| **Per tenant** | Multi-tenant SaaS fair usage | Tenant "MegaCorp" → 50,000 req/hour |
| **Per endpoint** | Protect expensive operations | `POST /bookings` → 10 req/min, `GET /tracking` → 1,000 req/min |
| **Per region** | Geographic throttling | Asia → 10,000 req/min, Europe → 10,000 req/min |
| **Global** | Total system capacity | Entire API → 100,000 req/min |

### Layered Rate Limiting

In production, you apply **multiple layers**:

```
Request arrives:
  1. Global limit: 100,000 req/min total? ✅
  2. Per-tenant limit: MegaCorp < 50,000 req/hour? ✅
  3. Per-user limit: john@acme.com < 1,000 req/hour? ✅
  4. Per-endpoint limit: POST /bookings < 10 req/min? ❌ REJECTED (429)
```

---

## Phase 3: Rate Limiting Algorithms

### Algorithm 1: Fixed Window Counter

Divide time into **fixed windows** (e.g., 1-minute intervals). Count requests in each window. Reject when count exceeds limit.

```
Limit: 100 requests per minute

Window 1 (00:00 - 00:59):
  Request count: 0...50...80...100 ← LIMIT
  Request 101 → REJECTED ❌

Window 2 (01:00 - 01:59):
  Counter resets to 0
  Request count: 0...50... ← accepts again ✅
```

**Implementation:**
```java
public class FixedWindowRateLimiter {
    private final int maxRequests;
    private final long windowSizeMs;
    private long windowStart;
    private int count;

    public synchronized boolean allowRequest() {
        long now = System.currentTimeMillis();
        if (now - windowStart >= windowSizeMs) {
            windowStart = now;
            count = 0;  // Reset counter at window boundary
        }
        if (count < maxRequests) {
            count++;
            return true;
        }
        return false;  // Rejected
    }
}
```

**Pros:** Simple, low memory (one counter per window).

**Cons: The Boundary Problem!**
```
Limit: 100 req/min

Window 1 (00:00 - 00:59): 0 requests at start, 100 requests at 00:50
Window 2 (01:00 - 01:59): 100 requests at 01:10, then stops

From 00:50 to 01:10 (20-second span): 200 requests!
That's 2x the rate limit in a short burst.
```

---

### Algorithm 2: Sliding Window Log

Track the **timestamp of every request**. For each new request, count how many timestamps fall within the last window.

```
Limit: 100 requests per minute
Current time: 01:15:30

Request log: [01:14:31, 01:14:35, 01:14:40, ..., 01:15:28, 01:15:29]

Step 1: Remove entries older than 01:14:30 (60 sec ago)
Step 2: Count remaining: 95
Step 3: 95 < 100 → ALLOW ✅, add 01:15:30 to log
```

**Pros:** Accurate — no boundary problem. True sliding window.

**Cons:** Memory-heavy — stores every request timestamp. For 100,000 req/min, that's 100,000 timestamps in memory per user.

---

### Algorithm 3: Sliding Window Counter

A hybrid of Fixed Window and Sliding Window Log. Uses **two fixed windows** and a **weighted average** to approximate a sliding window.

```
Limit: 100 requests per minute
Current window (01:00 - 01:59): 40 requests so far
Previous window (00:00 - 00:59): 80 requests total

Current position in window: 25% through (01:15)

Weighted count = Previous × (1 - position%) + Current
               = 80 × 0.75 + 40
               = 60 + 40 = 100

100 ≥ 100 → REJECT ❌
```

**Pros:** Low memory (just two counters), no boundary problem.
**Cons:** Approximate (not exact), slightly more complex logic.

**This is the most commonly used algorithm in production** (used by Cloudflare, Stripe).

---

### Algorithm 4: Token Bucket

A **bucket** holds tokens. Each request consumes one token. Tokens are added at a **constant rate**. If the bucket is empty, requests are rejected.

```
Bucket capacity: 10 tokens (burst size)
Refill rate: 2 tokens/second (sustained rate)

T=0.0: Bucket has 10 tokens
T=0.0: 8 requests arrive → 8 tokens consumed → 2 remaining
T=0.5: 1 token added → 3 remaining
T=1.0: 1 token added → 4 remaining
T=1.0: 5 requests arrive → 4 allowed, 1 rejected ❌
T=2.0: 2 tokens added → 2 remaining
```

```java
public class TokenBucket {
    private final int capacity;        // Max burst
    private final double refillRate;   // Tokens per second
    private double tokens;
    private long lastRefillTime;

    public synchronized boolean allowRequest() {
        refillTokens();
        if (tokens >= 1.0) {
            tokens -= 1.0;
            return true;
        }
        return false;
    }

    private void refillTokens() {
        long now = System.nanoTime();
        double elapsed = (now - lastRefillTime) / 1_000_000_000.0;
        tokens = Math.min(capacity, tokens + elapsed * refillRate);
        lastRefillTime = now;
    }
}
```

**Key parameters:**
- **Capacity (burst size):** Maximum number of requests in a burst
- **Refill rate:** Sustained request rate (tokens/sec)

**Pros:** Allows controlled bursts while enforcing average rate. Simple. Memory-efficient.
**Cons:** Doesn't track per-window counts.

**Used by:** AWS API Gateway, Stripe API, most cloud providers.

---

### Algorithm 5: Leaky Bucket

Requests enter a **queue (bucket)**. They're processed at a **constant rate**, like water leaking from a bucket. If the bucket overflows, new requests are dropped.

```
Incoming: Bursty traffic (0, 0, 50, 0, 0, 100, 0, ...)
Outgoing: Constant (10, 10, 10, 10, 10, 10, 10, ...)

Queue capacity: 50
Processing rate: 10 requests/sec

T=0: 50 requests arrive → Queue: [50 requests]
T=1: Process 10, Queue: [40 remaining]
T=2: Process 10, Queue: [30 remaining]
     100 new requests arrive → 20 added to queue (capacity 50), 80 DROPPED
T=3: Process 10, Queue: [40 remaining]
```

**Pros:** Smooths out traffic — downstream services see a constant rate.
**Cons:** Doesn't allow bursts at all (even legitimate ones). Added latency from queueing.

**Difference from Token Bucket:**

| | Token Bucket | Leaky Bucket |
|---|---|---|
| Bursts | Allows bursts (up to bucket size) | No bursts — constant output rate |
| Behavior | Rejects when tokens exhausted | Queues requests (or drops if full) |
| Output | Bursty | Smooth/constant |
| Best for | APIs (allow short bursts) | Network traffic shaping |

---

### Algorithm Comparison

| Algorithm | Accuracy | Memory | Burst Handling | Complexity | Production Use |
|---|---|---|---|---|---|
| **Fixed Window** | Low (boundary issue) | Very low | Allows 2x burst at boundary | Simple | Basic use cases |
| **Sliding Log** | Exact | High | No burst issue | Moderate | Audit-heavy systems |
| **Sliding Counter** | Approximate | Low | Smooth | Moderate | **Cloudflare, Stripe** |
| **Token Bucket** | Good | Very low | Controlled bursts | Simple | **AWS, most APIs** |
| **Leaky Bucket** | Good | Low | No bursts | Moderate | Network shaping |

---

## Phase 4: Where to Implement Rate Limiting

### Layer 1: API Gateway (Most Common)

Centralized rate limiting at the entry point — before requests reach any backend service.

```
Client → [API Gateway + Rate Limiter] → Backend Services

All requests pass through the gateway.
Rate limiting happens BEFORE any service processing.
```

**Tools:** Spring Cloud Gateway, Kong, AWS API Gateway, Nginx

**Spring Cloud Gateway Example:**
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: booking-service
          uri: lb://booking-service
          predicates:
            - Path=/api/bookings/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 50     # 50 requests/sec sustained
                redis-rate-limiter.burstCapacity: 100     # Burst up to 100
                redis-rate-limiter.requestedTokens: 1     # 1 token per request
                key-resolver: "#{@apiKeyResolver}"
```

```java
@Bean
public KeyResolver apiKeyResolver() {
    return exchange -> Mono.just(
        exchange.getRequest().getHeaders().getFirst("X-API-Key")
    );
}
```

---

### Layer 2: Application-Level (Per-Service)

Each microservice implements its own rate limiting for fine-grained control.

```java
// Using Resilience4j Rate Limiter in Spring Boot
@RateLimiter(name = "bookingApi", fallbackMethod = "rateLimitFallback")
@GetMapping("/api/bookings/{id}")
public Booking getBooking(@PathVariable String id) {
    return bookingService.findById(id);
}

private Booking rateLimitFallback(String id, RequestNotPermitted ex) {
    throw new ResponseStatusException(HttpStatus.TOO_MANY_REQUESTS, 
        "Rate limit exceeded. Try again later.");
}
```

```yaml
resilience4j:
  ratelimiter:
    instances:
      bookingApi:
        limit-for-period: 100          # 100 requests per period
        limit-refresh-period: 1s       # Period = 1 second
        timeout-duration: 0s           # Don't wait, immediately reject
```

---

### Layer 3: Load Balancer / CDN

Rate limiting at the edge — closest to the client.

```
Client → [CDN/WAF Rate Limiter] → Load Balancer → Services

Cloudflare, AWS CloudFront, Akamai can rate limit at the edge.
DDoS attacks are blocked before they reach your infrastructure.
```

---

### Layer 4: Database Connection Limiting

Protect the database from too many concurrent connections:

```yaml
# HikariCP connection pool
spring:
  datasource:
    hikari:
      maximum-pool-size: 50          # Max 50 DB connections
      connection-timeout: 30000       # Wait max 30s for a connection
      
# If all 50 connections are busy, new requests wait (up to timeout) or fail.
# This IS a form of rate limiting — protecting the DB.
```

---

## Phase 5: Distributed Rate Limiting

### The Problem

With multiple API servers behind a load balancer, each server has its own rate limiter. A client can send 100 req/sec to EACH server.

```
Limit: 100 req/sec per user

❌ Local rate limiting:
  Server 1: User A → 100 req/sec ← at limit
  Server 2: User A → 100 req/sec ← at limit
  Server 3: User A → 100 req/sec ← at limit
  
  Total: User A sends 300 req/sec! (3x the limit)
```

### Solution: Centralized Counter with Redis

All servers share a **single counter** stored in Redis.

```
Server 1 ──┐
Server 2 ──┼──→ Redis: "user:A:rate = 87" ← single source of truth
Server 3 ──┘
```

**Redis Implementation (Sliding Window Counter):**
```java
@Component
public class RedisRateLimiter {

    private final StringRedisTemplate redis;
    
    public boolean isAllowed(String key, int maxRequests, Duration window) {
        String redisKey = "ratelimit:" + key;
        long now = System.currentTimeMillis();
        long windowStart = now - window.toMillis();
        
        // Atomic operation using Redis sorted set
        redis.executePipelined((RedisCallback<Object>) connection -> {
            byte[] keyBytes = redisKey.getBytes();
            // Remove expired entries
            connection.zRemRangeByScore(keyBytes, 0, windowStart);
            // Add current request
            connection.zAdd(keyBytes, now, (now + ":" + UUID.randomUUID()).getBytes());
            // Count requests in window
            connection.zCard(keyBytes);
            // Set TTL on the key
            connection.expire(keyBytes, window.getSeconds());
            return null;
        });
        
        Long count = redis.opsForZSet().size(redisKey);
        return count != null && count <= maxRequests;
    }
}
```

**Redis Lua Script (Atomic Token Bucket):**
```lua
-- Token bucket in Redis (atomic)
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local bucket = redis.call("hmget", key, "tokens", "last_refill")
local tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

-- Refill tokens
local elapsed = now - last_refill
local new_tokens = math.min(capacity, tokens + elapsed * refill_rate)

-- Check if request is allowed
if new_tokens >= requested then
    new_tokens = new_tokens - requested
    redis.call("hmset", key, "tokens", new_tokens, "last_refill", now)
    redis.call("expire", key, math.ceil(capacity / refill_rate) * 2)
    return 1  -- Allowed
else
    redis.call("hmset", key, "tokens", new_tokens, "last_refill", now)
    redis.call("expire", key, math.ceil(capacity / refill_rate) * 2)
    return 0  -- Rejected
end
```

**Why Lua?** Redis executes Lua scripts **atomically** — no race conditions between checking and incrementing.

---

### Race Conditions in Distributed Rate Limiting

```
❌ Non-atomic check-then-increment:
  Server 1: READ count = 99        (< 100, allowed!)
  Server 2: READ count = 99        (< 100, allowed!)  ← Race condition!
  Server 1: WRITE count = 100
  Server 2: WRITE count = 101      ← Exceeded limit!
```

**Solutions:**
1. **Redis Lua scripts** — atomic read-check-write
2. **Redis MULTI/EXEC** — transactions
3. **Redis INCR + TTL** — atomic increment with fixed window

```java
// Simple atomic rate limiter with Redis INCR
public boolean isAllowed(String userId, int limit, Duration window) {
    String key = "rate:" + userId + ":" + Instant.now().getEpochSecond() / window.getSeconds();
    Long count = redis.opsForValue().increment(key);
    if (count == 1) {
        redis.expire(key, window);  // Set TTL on first request
    }
    return count <= limit;
}
```

---

## Phase 6: Rate Limit Response Design

### HTTP Response for Rejected Requests

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1672531260

{
    "error": "rate_limit_exceeded",
    "message": "You have exceeded the rate limit of 100 requests per minute.",
    "retryAfter": 30
}
```

### Standard Headers

| Header | Meaning | Example |
|---|---|---|
| `X-RateLimit-Limit` | Max requests allowed in window | `100` |
| `X-RateLimit-Remaining` | Requests remaining in current window | `23` |
| `X-RateLimit-Reset` | Unix timestamp when window resets | `1672531260` |
| `Retry-After` | Seconds to wait before retrying | `30` |

**Always include these headers** — even on successful requests. It helps well-behaved clients pace themselves.

```java
@Component
public class RateLimitResponseFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpResponse response = exchange.getResponse();
        response.getHeaders().add("X-RateLimit-Limit", "100");
        response.getHeaders().add("X-RateLimit-Remaining", 
            String.valueOf(getRemainingRequests(exchange)));
        response.getHeaders().add("X-RateLimit-Reset", 
            String.valueOf(getResetTimestamp()));
        return chain.filter(exchange);
    }
}
```

---

## Phase 7: Throttling Strategies

When you don't want to hard-reject requests, you can **degrade gracefully**.

### Strategy 1: Request Queueing

Queue excess requests and process them when capacity is available.

```
Incoming: 200 req/sec
Capacity: 100 req/sec
Queue size: 500

100 req/sec → processed immediately
100 req/sec → queued → processed when slots free up
If queue > 500 → THEN reject
```

**Risk:** Unbounded queues can consume all memory. Always set a queue size limit.

---

### Strategy 2: Response Degradation

Serve a **simplified response** when under load.

```java
@GetMapping("/api/tracking/{id}")
public TrackingResponse getTracking(@PathVariable String id) {
    if (isUnderLoad()) {
        // Serve cached/simplified response
        return trackingCache.getBasicStatus(id);  // Just status, no full history
    }
    // Full response with complete tracking history
    return trackingService.getFullTracking(id);
}
```

---

### Strategy 3: Priority-Based Throttling

Different clients get different treatment under load.

```
Available capacity: 1,000 req/sec

Platinum customers: guaranteed 500 req/sec (never throttled)
Gold customers:     guaranteed 300 req/sec
Free tier:          200 req/sec (first to be throttled)

Under extreme load:
  Free tier → reject
  Gold → degrade
  Platinum → full service
```

```yaml
rate-limiting:
  tiers:
    platinum:
      requests-per-second: 500
      burst: 1000
      priority: 1         # Highest priority, last to be throttled
    gold:
      requests-per-second: 300
      burst: 600
      priority: 2
    free:
      requests-per-second: 100
      burst: 150
      priority: 3         # Lowest priority, first to be throttled
```

---

### Strategy 4: Backpressure

The system signals to the client to **slow down** rather than rejecting outright.

```
Normal response:
  HTTP 200, processing time: 5ms

Under load:
  HTTP 200, processing time: 500ms ← Deliberately slower
  Header: X-Backpressure: medium

Client detects backpressure → reduces request rate → system recovers
```

---

## Phase 8: API Quota Management

### Quotas vs Rate Limits

| | Rate Limit | Quota |
|---|---|---|
| **Time scale** | Short (per second/minute) | Long (per day/month) |
| **Purpose** | Prevent burst abuse | Control total usage |
| **Example** | 100 req/min | 10,000 req/day or 1M req/month |
| **Reset** | Every minute | Daily/monthly |

### Implementing Quotas

```java
@Service
public class QuotaService {

    private final RedisTemplate<String, String> redis;

    public QuotaCheckResult checkQuota(String tenantId) {
        String dailyKey = "quota:daily:" + tenantId + ":" + LocalDate.now();
        String monthlyKey = "quota:monthly:" + tenantId + ":" + YearMonth.now();

        Long dailyUsage = redis.opsForValue().increment(dailyKey);
        Long monthlyUsage = redis.opsForValue().increment(monthlyKey);

        // Set TTL on first use
        if (dailyUsage == 1) redis.expire(dailyKey, Duration.ofDays(2));
        if (monthlyUsage == 1) redis.expire(monthlyKey, Duration.ofDays(35));

        TenantPlan plan = tenantService.getPlan(tenantId);
        
        if (monthlyUsage > plan.getMonthlyQuota()) {
            return QuotaCheckResult.exceeded("Monthly quota exceeded");
        }
        if (dailyUsage > plan.getDailyQuota()) {
            return QuotaCheckResult.exceeded("Daily quota exceeded");
        }
        return QuotaCheckResult.allowed(plan.getDailyQuota() - dailyUsage);
    }
}
```

---

## Phase 9: When Rate Limiting is an Overhead

### Scenario 1: Internal Service-to-Service Calls

Microservices calling each other within a trusted network. You control both sides.

```
Booking Service → Rate Service (internal, trusted)
Adding rate limiting: adds latency (Redis call) for no real protection.
Better: use circuit breakers and bulkheads instead.
```

---

### Scenario 2: Batch Processing Pipelines

A nightly job processing 1M records. Rate limiting would artificially slow it down.

```
Batch job: Process all pending shipment invoices (1M records)
Rate limit: 100 req/sec → would take 2.7 hours instead of 5 minutes.
```

**Solution:** Exempt batch jobs from rate limiting, or give them a dedicated high-limit quota.

---

### Scenario 3: Extremely Low Traffic

Admin API used by 3 people, 10 requests/day. Rate limiting infrastructure (Redis, Lua scripts) costs more than the problem it solves.

---

### Scenario 4: WebSocket / Streaming Connections

Long-lived connections where the concept of "requests per second" doesn't apply.

**Better:** Limit concurrent connections and message rate within the stream.

---

## Phase 10: Real-World Case Studies

### Case 1: Logistics — Partner API Rate Limiting

**Problem:** A freight platform exposes APIs to 200 carrier partners. Some carriers poll for updates every second (wasteful), while others send malformed requests in loops.

**Solution:**
```
Rate limit per carrier API key:
  Standard tier: 60 req/min, burst 10
  Premium tier: 600 req/min, burst 50
  
Rate limit per endpoint:
  GET /tracking/{id}:     200 req/min (high read volume expected)
  POST /bookings:         20 req/min (expensive operation)
  POST /bulk-upload:      5 req/min (very expensive)

Implementation: Redis token bucket at API Gateway (Spring Cloud Gateway)
```

**Result:** Carrier with buggy integration was auto-limited at 60 req/min instead of flooding the system with 5,000 req/min.

---

### Case 2: Logistics — Webhook Rate Limiting

**Problem:** Platform sends webhooks to customers when shipment status changes. Some customers' webhook endpoints are slow (5-sec response). During peak, thousands of webhooks queue up.

**Solution:**
```
Outbound webhook rate limiting:
  Per customer: max 50 concurrent webhook deliveries
  Global: max 5,000 concurrent deliveries

If customer endpoint is slow:
  → Queue excess webhooks
  → Exponential backoff on failures
  → After 24 hours of failures → pause and notify customer
```

---

## Key Takeaways

1. **Token Bucket** is the go-to algorithm — allows bursts, simple, memory-efficient
2. **Sliding Window Counter** is the best for accuracy without high memory
3. **Rate limit at the API Gateway** — centralized, before any backend processing
4. **Use Redis for distributed rate limiting** — shared counters across all servers
5. **Always return rate limit headers** — help clients self-regulate
6. **Layer your limits** — global → tenant → user → endpoint
7. **Distinguish rate limits (burst) from quotas (total usage)**
8. **Degrade gracefully** when possible — queue, simplify, prioritize before hard-rejecting
9. **Exempt internal services** — use circuit breakers/bulkheads instead

---

**See also:** [[Microservices Patterns]] (Phase 8 for Spring Boot integration), [[Caching]] (for caching rate limit counters)
