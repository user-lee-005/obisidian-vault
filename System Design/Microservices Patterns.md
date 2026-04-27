# Microservices Patterns

> *"Microservices are not about doing more with less. They're about doing less per service so you can do more as a system."*

---

## Phase 1: Monolith vs Microservices — The Foundation

### What is a Monolith?

A monolith is a **single deployable unit** that contains all features of the application — booking, tracking, billing, notifications — all in one codebase, one build, one deployment.

```
┌─────────────────────────────────────┐
│           Monolith App              │
│                                     │
│  ┌──────────┐  ┌──────────────┐    │
│  │ Booking  │  │  Tracking    │    │
│  │ Module   │  │  Module      │    │
│  ├──────────┤  ├──────────────┤    │
│  │ Billing  │  │ Notification │    │
│  │ Module   │  │ Module       │    │
│  └──────────┘  └──────────────┘    │
│                                     │
│         Shared Database             │
└─────────────────────────────────────┘
```

**Pros:**
- Simple to develop, test, and deploy initially
- Single codebase — easy to understand for small teams
- No network latency between modules (in-process calls)
- Single database — ACID transactions across modules

**Cons:**
- As it grows, becomes a **big ball of mud** — hard to understand, modify, test
- A bug in billing can crash the entire app including tracking
- Can't scale individual modules — must scale the whole thing
- Deployment of a tiny change requires redeploying everything
- Technology lock-in — entire app uses one language/framework

### What are Microservices?

Microservices is an architecture where the application is composed of **small, independent services**, each responsible for **one business capability**, communicating over the network.

```
┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐
│  Booking   │  │  Tracking  │  │  Billing   │  │Notification│
│  Service   │  │  Service   │  │  Service   │  │  Service   │
│            │  │            │  │            │  │            │
│  Own DB    │  │  Own DB    │  │  Own DB    │  │  Own DB    │
└─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └──────┬─────┘
      │               │               │                │
      └───────────────┴───────────────┴────────────────┘
                  Communication (HTTP / gRPC / Kafka)
```

**Key Characteristics:**
1. **Single Responsibility** — Each service does one thing well
2. **Independently Deployable** — Deploy booking without touching tracking
3. **Own Data Store** — Each service manages its own database
4. **Communicate via Network** — HTTP, gRPC, or message queues
5. **Technology Agnostic** — Booking in Java, Analytics in Python, Tracking in Go

### When to Use What

| Factor | Monolith | Microservices |
|---|---|---|
| Team size | < 10 developers | > 10, multiple teams |
| Domain complexity | Simple/moderate | Complex, clear boundaries |
| Scale requirements | Uniform scaling | Different modules scale differently |
| Deployment frequency | Weekly/monthly | Multiple times per day |
| Organizational structure | Single team | Multiple autonomous teams |
| Startup phase | ✅ Start here | Migrate to this when needed |

> **Important:** Most successful microservices architectures started as monoliths. Don't start with microservices — you don't know your domain boundaries well enough yet.

---

## Phase 2: Service Discovery

### The Problem

In a monolith, `BookingService` calls `TrackingService` via a method call — the address is the JVM itself. In microservices, `TrackingService` could be running on any of 20 servers, and instances come and go as they scale up/down.

```
Booking Service: "Where is Tracking Service right now?"
  - tracking-svc-1:8080 (healthy)
  - tracking-svc-2:8080 (healthy)
  - tracking-svc-3:8080 (just crashed)
  - tracking-svc-4:8080 (just started)
```

Hardcoding IPs is impossible when instances are ephemeral.

### What is Service Discovery?

A mechanism that allows services to **find each other dynamically** without hardcoded addresses.

### Client-Side Discovery

The client (calling service) is responsible for looking up available instances and choosing one.

```
1. Booking Service asks Service Registry: "Where is Tracking Service?"
2. Registry responds: ["tracking-1:8080", "tracking-2:8080"]
3. Booking Service picks one (e.g., round-robin) and calls it directly

┌──────────────┐     ┌───────────────────┐
│   Booking    │─1──→│  Service Registry  │
│   Service    │←2───│  (Eureka/Consul)   │
│              │     └───────────────────┘
│              │─3──→ tracking-2:8080
└──────────────┘
```

**Tools:** Netflix Eureka, HashiCorp Consul

**Spring Boot Example (Eureka):**
```yaml
# Tracking Service registers itself
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
  instance:
    prefer-ip-address: true
```

```java
// Booking Service discovers and calls Tracking Service
@FeignClient(name = "tracking-service")  // Eureka resolves this name
public interface TrackingClient {
    @GetMapping("/api/tracking/{shipmentId}")
    TrackingInfo getTracking(@PathVariable String shipmentId);
}
```

**Pros:** Client has full control over load balancing strategy.
**Cons:** Every client needs discovery logic; language-specific.

---

### Server-Side Discovery

A **load balancer** sits between the client and services. The client calls the load balancer, which routes to a healthy instance.

```
┌──────────────┐     ┌────────────────┐     ┌─────────────────┐
│   Booking    │────→│ Load Balancer  │────→│ Tracking Service │
│   Service    │     │  (Nginx/ALB)   │     │   Instance 1/2/3 │
└──────────────┘     └───────┬────────┘     └─────────────────┘
                             │
                     ┌───────────────────┐
                     │  Service Registry  │
                     └───────────────────┘
```

**Tools:** AWS ALB/NLB, Kubernetes Service, Nginx

**Pros:** Client is simple — just calls a single URL; language-agnostic.
**Cons:** Extra network hop; load balancer becomes a potential bottleneck.

---

### DNS-Based Discovery

Services register with a DNS server. The service name resolves to IP addresses of healthy instances.

```
Booking Service calls: http://tracking-service/api/tracking/SHP-001
DNS resolves "tracking-service" → 10.0.1.5, 10.0.1.6, 10.0.1.7
```

**Tools:** Consul DNS, Kubernetes CoreDNS, AWS Cloud Map

**Kubernetes makes this automatic:**
```yaml
# tracking-service.default.svc.cluster.local resolves to pod IPs
apiVersion: v1
kind: Service
metadata:
  name: tracking-service
spec:
  selector:
    app: tracking
  ports:
    - port: 8080
```

---

## Phase 3: API Gateway

### The Problem

A client (web/mobile app) needs data from multiple microservices. Without a gateway, the client must know about every service:

```
❌ Without Gateway:
  Mobile App → Booking Service (https://booking.internal:8080)
  Mobile App → Tracking Service (https://tracking.internal:8081)
  Mobile App → Billing Service (https://billing.internal:8082)
  Mobile App → Notification Service (https://notification.internal:8083)

Problems:
  - Client knows internal service URLs
  - Multiple round trips from mobile
  - Each service needs its own auth logic
  - No central rate limiting or logging
```

### What is an API Gateway?

A **single entry point** for all client requests. It routes, composes, and filters requests to backend microservices.

```
✅ With Gateway:
  Mobile App → [API Gateway] → Booking Service
                             → Tracking Service
                             → Billing Service
                             → Notification Service

Client only knows one URL: https://api.logistics.com
```

### Responsibilities

| Responsibility | Description |
|---|---|
| **Routing** | `/api/bookings/*` → Booking Service, `/api/tracking/*` → Tracking Service |
| **Authentication** | Validate JWT/OAuth token once at the gateway |
| **Rate Limiting** | 100 requests/min per client |
| **Load Balancing** | Distribute requests across service instances |
| **Request Composition** | Combine data from multiple services into one response |
| **Protocol Translation** | Client speaks REST → Gateway speaks gRPC to internal services |
| **Caching** | Cache frequently requested responses |
| **Logging & Monitoring** | Central request logging and metrics |

### API Gateway Pattern — BFF (Backend for Frontend)

Different clients need different data. Mobile needs lightweight data, web dashboard needs rich data.

```
Mobile App  → [Mobile BFF Gateway]  → Services (returns minimal JSON)
Web App     → [Web BFF Gateway]     → Services (returns rich JSON)
Partner API → [Partner API Gateway] → Services (returns XML/EDI)
```

### Tools

| Gateway | Type | Best For |
|---|---|---|
| **Spring Cloud Gateway** | Java-native | Spring Boot microservices |
| **Kong** | Open-source | Multi-language, plugin ecosystem |
| **AWS API Gateway** | Managed | Serverless, AWS-native |
| **Nginx** | Reverse proxy | Simple routing, high performance |
| **Envoy** | Service mesh sidecar | Kubernetes, Istio |

### Spring Cloud Gateway Example

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: booking-service
          uri: lb://booking-service   # Eureka-resolved
          predicates:
            - Path=/api/bookings/**
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args:
                name: bookingCB
                fallbackUri: forward:/fallback/booking
                
        - id: tracking-service
          uri: lb://tracking-service
          predicates:
            - Path=/api/tracking/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
```

---

## Phase 4: Circuit Breaker Pattern

### The Problem — Cascading Failures

In microservices, services call other services. If one service is slow or down, the calling service **waits**, consuming threads. Eventually, the caller runs out of threads and also fails. This cascades up the chain.

```
❌ Cascading Failure:
  User → Booking Service → Rate Service (slow! 30-sec timeout)
                          → Booking holds thread for 30 sec
                          → All 200 threads waiting on Rate Service
                          → Booking Service stops accepting requests
                          → Users see 503 errors
                          → Even Tracking Service (unrelated!) is impacted 
                            because Booking is hogging shared resources
```

A single slow dependency takes down the entire system.

### What is a Circuit Breaker?

Inspired by electrical circuit breakers — when a service detects repeated failures to a dependency, it **"trips" the circuit** and stops calling that dependency temporarily. This prevents cascading failures.

### The Three States

```
┌──────────┐    failures > threshold    ┌──────────┐
│          │ ──────────────────────────→ │          │
│  CLOSED  │                             │   OPEN   │
│ (normal) │ ←────────────────────────── │ (failing)│
│          │     success in half-open    │          │
└──────────┘                             └────┬─────┘
      ↑                                       │
      │          timeout expires              │
      │                                       ↓
      │                                ┌──────────┐
      └────────────────────────────────│HALF-OPEN │
               success                 │ (testing) │
                                       └──────────┘
```

**CLOSED (Normal):** All requests pass through to the dependency. Failures are counted.

**OPEN (Failing):** Requests **immediately fail** with a fallback response — no call is made to the failing dependency. This gives the dependency time to recover.

**HALF-OPEN (Testing):** After a timeout, a limited number of test requests are allowed through. If they succeed → back to CLOSED. If they fail → back to OPEN.

### Configuration Parameters

| Parameter | Meaning | Typical Value |
|---|---|---|
| **Failure Rate Threshold** | % of failures before tripping | 50% |
| **Slow Call Threshold** | % of slow calls before tripping | 80% |
| **Slow Call Duration** | What counts as "slow" | 3 seconds |
| **Sliding Window Size** | Number of calls to evaluate | Last 10-100 calls |
| **Wait Duration in Open** | How long to stay open before half-open | 30-60 seconds |
| **Permitted Calls in Half-Open** | Test calls in half-open state | 3-5 calls |

### Resilience4j Implementation (Spring Boot)

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      rateService:
        sliding-window-size: 10
        failure-rate-threshold: 50
        slow-call-rate-threshold: 80
        slow-call-duration-threshold: 3s
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
        register-health-indicator: true
```

```java
@Service
public class BookingService {

    private final RateServiceClient rateClient;

    @CircuitBreaker(name = "rateService", fallbackMethod = "getDefaultRate")
    public FreightRate getRate(String origin, String destination) {
        return rateClient.calculateRate(origin, destination);  // External call
    }

    // Fallback when circuit is open or call fails
    private FreightRate getDefaultRate(String origin, String dest, Exception ex) {
        log.warn("Rate service unavailable, using cached rate: {}", ex.getMessage());
        return cachedRateService.getLastKnownRate(origin, dest);
    }
}
```

### What Happens in Practice

```
Request 1:  Rate Service responds in 100ms  ✅ (CLOSED)
Request 2:  Rate Service responds in 200ms  ✅ (CLOSED)
Request 3:  Rate Service timeout (5 sec)    ❌ (CLOSED, 1 failure)
Request 4:  Rate Service timeout            ❌ (CLOSED, 2 failures)
...
Request 10: Rate Service timeout            ❌ (CLOSED, 6/10 failures = 60% > 50%)
                                            → Circuit trips to OPEN

Request 11: Returns cached rate immediately (no call made) → OPEN
Request 12: Returns cached rate immediately → OPEN
...
(30 seconds pass)                           → HALF-OPEN

Request 50: Rate Service responds ✅        → Test passed
Request 51: Rate Service responds ✅        → Test passed
Request 52: Rate Service responds ✅        → 3/3 passed → CLOSED (normal)
```

---

## Phase 5: Retry Pattern

### The Problem

Transient failures — network blips, temporary overload, DNS hiccups — are common in distributed systems. A single failed call doesn't mean the service is down.

### Basic Retry

```java
@Retry(name = "rateService", fallbackMethod = "getDefaultRate")
@CircuitBreaker(name = "rateService", fallbackMethod = "getDefaultRate")
public FreightRate getRate(String origin, String destination) {
    return rateClient.calculateRate(origin, destination);
}
```

```yaml
resilience4j:
  retry:
    instances:
      rateService:
        max-attempts: 3
        wait-duration: 500ms
        retry-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        ignore-exceptions:
          - com.logistics.exceptions.InvalidRouteException  # Don't retry business errors
```

### Exponential Backoff

Instead of retrying immediately, wait progressively longer:

```
Attempt 1: fails → wait 500ms
Attempt 2: fails → wait 1000ms
Attempt 3: fails → wait 2000ms
Attempt 4: fails → give up, use fallback
```

```yaml
resilience4j:
  retry:
    instances:
      rateService:
        max-attempts: 4
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
```

### Retry + Jitter

When many clients retry at the same time, they hit the recovering service simultaneously — the **thundering herd** problem.

**Solution:** Add randomness (jitter) to retry delays:

```
Client A: Attempt 2 after 500ms + random(0-200ms) = 612ms
Client B: Attempt 2 after 500ms + random(0-200ms) = 543ms
Client C: Attempt 2 after 500ms + random(0-200ms) = 701ms
(Requests spread out instead of spiking simultaneously)
```

### ⚠️ When NOT to Retry

| Scenario | Why |
|---|---|
| **4xx errors (client errors)** | The request is wrong — retrying won't fix it |
| **Non-idempotent operations** | `POST /bookings` without idempotency key → creates duplicates! |
| **Business logic errors** | "Insufficient funds" won't change on retry |
| **Circuit is OPEN** | The circuit breaker already knows the service is down |

> **Order of application:** Retry wraps the call, Circuit Breaker wraps the retry.
> `CircuitBreaker → Retry → Actual Call`

---

## Phase 6: Bulkhead Pattern

### The Problem

A service calls multiple dependencies. One slow dependency consumes all available threads, starving other (healthy) dependencies.

```
Booking Service (200 threads total):
  → Rate Service (slow!)     → 180 threads waiting
  → Tracking Service (healthy) → 20 threads remaining → starts failing too!
  → Notification Service (healthy) → 0 threads available → completely blocked!
```

### What is a Bulkhead?

Named after watertight compartments in ships — if one compartment floods, others remain dry.

**Bulkhead isolates thread pools** so a slow dependency can only consume its allocated quota.

```
Booking Service:
  ┌─────────────────────┐
  │ Rate Service Pool    │ → Max 50 threads (if Rate is slow, only these 50 are stuck)
  ├─────────────────────┤
  │ Tracking Pool        │ → Max 50 threads (unaffected by Rate slowness)
  ├─────────────────────┤
  │ Notification Pool    │ → Max 30 threads (unaffected by Rate slowness)
  ├─────────────────────┤
  │ General Pool         │ → Max 70 threads (for other work)
  └─────────────────────┘
```

### Resilience4j Bulkhead

```yaml
resilience4j:
  bulkhead:
    instances:
      rateService:
        max-concurrent-calls: 50        # Max 50 parallel calls to Rate Service
        max-wait-duration: 500ms         # Wait max 500ms if all 50 slots are busy
      trackingService:
        max-concurrent-calls: 50
        max-wait-duration: 500ms
```

```java
@Bulkhead(name = "rateService", fallbackMethod = "getDefaultRate")
@CircuitBreaker(name = "rateService", fallbackMethod = "getDefaultRate")
@Retry(name = "rateService")
public FreightRate getRate(String origin, String destination) {
    return rateClient.calculateRate(origin, destination);
}
```

### Thread Pool Bulkhead

For even stricter isolation, use a **separate thread pool** (not just a semaphore):

```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      rateService:
        max-thread-pool-size: 20
        core-thread-pool-size: 10
        queue-capacity: 50
```

---

## Phase 7: Timeout Pattern

### The Problem

Without timeouts, a call to a slow service waits **indefinitely**, holding resources hostage.

### Setting Timeouts

```yaml
resilience4j:
  timelimiter:
    instances:
      rateService:
        timeout-duration: 3s        # Fail after 3 seconds
        cancel-running-future: true  # Cancel the underlying thread
```

```java
@TimeLimiter(name = "rateService", fallbackMethod = "getDefaultRate")
@CircuitBreaker(name = "rateService", fallbackMethod = "getDefaultRate")
public CompletableFuture<FreightRate> getRate(String origin, String dest) {
    return CompletableFuture.supplyAsync(() -> rateClient.calculateRate(origin, dest));
}
```

### How to Choose Timeout Values

| Approach | Method |
|---|---|
| **p99 latency + buffer** | If Rate Service p99 = 800ms → timeout = 2-3 sec |
| **Business SLA** | "Booking page must load in 5 sec" → each dependency gets 2 sec |
| **Circuit breaker alignment** | Timeout should be ≤ circuit breaker's slow call threshold |

### The Full Resilience Stack

The order matters — each pattern wraps the previous:

```
Request → TimeLimiter → CircuitBreaker → Retry → Bulkhead → Actual Call

1. TimeLimiter: "Give up after 3 seconds"
2. CircuitBreaker: "Don't even try if the service is known to be down"
3. Retry: "Try 3 times for transient failures"
4. Bulkhead: "Only allow 50 concurrent calls"
5. Actual Call: The real HTTP/gRPC call
```

---

## Phase 8: Rate Limiting

### The Problem

A misbehaving client (or DDoS attack) sends thousands of requests per second, overwhelming your service and impacting all users.

### What is Rate Limiting?

Controlling the **number of requests** a client can make within a time window.

### Common Algorithms

#### 1. Fixed Window Counter

Count requests in fixed time windows (e.g., per minute).

```
Window: 00:00 - 00:01 → 80 requests (limit 100) ✅
Window: 00:01 - 00:02 → 120 requests → first 100 ✅, last 20 rejected ❌

Problem: Burst at window boundary
  00:00:50 → 90 requests
  00:01:10 → 90 requests
  = 180 requests in 20 seconds! (defeats the per-minute limit)
```

#### 2. Sliding Window Log

Track timestamps of all requests. Count how many fall in the last N seconds.

```
Request at 00:01:15 → count requests since 00:00:15
  Found 95 → allow (< 100 limit)

Request at 00:01:16 → count requests since 00:00:16
  Found 101 → reject ❌
```

**Pros:** Accurate. **Cons:** Memory-heavy (stores every timestamp).

#### 3. Token Bucket

A bucket holds tokens. Each request consumes one token. Tokens are added at a fixed rate.

```
Bucket capacity: 10 tokens
Refill rate: 2 tokens/second

T=0: Bucket has 10 tokens
T=0: 8 requests arrive → 8 tokens consumed → 2 left
T=1: 2 tokens added → 4 in bucket
T=1: 5 requests arrive → 4 succeed, 1 rejected
```

**Pros:** Allows short bursts while enforcing average rate.
**Used by:** AWS API Gateway, Spring Cloud Gateway.

#### 4. Leaky Bucket

Requests enter a bucket (queue). They're processed at a constant rate, like water leaking from a bucket.

```
Incoming: 100 requests/sec (bursty)
Processing: 10 requests/sec (constant)
Queue: holds up to 50 → rest are dropped

Smooths out traffic to a constant rate.
```

### Spring Cloud Gateway Rate Limiting

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
                redis-rate-limiter.replenishRate: 10    # 10 requests/sec sustained
                redis-rate-limiter.burstCapacity: 20     # Allow burst of 20
                key-resolver: "#{@userKeyResolver}"      # Rate limit per user
```

```java
@Bean
public KeyResolver userKeyResolver() {
    return exchange -> Mono.just(
        exchange.getRequest().getHeaders().getFirst("X-User-Id")
    );
}
```

### Rate Limit Response

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1672531260
```

---

## Phase 9: Saga Pattern — Distributed Transactions

### The Problem

In a monolith, a booking involves multiple database operations wrapped in a **single ACID transaction**:

```java
@Transactional
public void createBooking(BookingRequest request) {
    bookingRepo.save(booking);         // Step 1
    inventoryRepo.reserve(container);   // Step 2
    paymentRepo.charge(amount);         // Step 3
    // If step 3 fails → everything rolls back automatically
}
```

In microservices, each service has its **own database**. You can't have a single ACID transaction across services.

```
Booking Service (DB 1) → Inventory Service (DB 2) → Payment Service (DB 3)

If Payment fails, how do you undo the Booking and Inventory reservation?
There's no distributed @Transactional.
```

### What is a Saga?

A saga is a sequence of **local transactions** across services where each step has a corresponding **compensating transaction** (undo action) that runs if a later step fails.

### Choreography-Based Saga

Services communicate via **events**. Each service listens for events and acts accordingly — no central coordinator.

```
1. Booking Service creates booking → publishes "BookingCreated"

2. Inventory Service hears "BookingCreated"
   → Reserves container → publishes "InventoryReserved"

3. Payment Service hears "InventoryReserved"
   → Charges payment → publishes "PaymentCompleted"

4. Booking Service hears "PaymentCompleted"
   → Marks booking as CONFIRMED

─── If Payment fails: ───

3. Payment Service → publishes "PaymentFailed"

4. Inventory Service hears "PaymentFailed"
   → Releases container (compensating transaction)

5. Booking Service hears "PaymentFailed"
   → Marks booking as CANCELLED (compensating transaction)
```

```
Happy path:
  Booking ──event──→ Inventory ──event──→ Payment ──event──→ Done ✅

Failure path:
  Booking ──event──→ Inventory ──event──→ Payment ❌
                     ←──undo───          ←──undo event
  Booking undone ← Inventory released
```

**Pros:** Loosely coupled, no single point of failure.
**Cons:** Hard to track overall state, complex failure handling, debugging is difficult.

---

### Orchestration-Based Saga

A **central orchestrator** (saga coordinator) controls the flow and tells each service what to do.

```
┌─────────────────────────────────────────┐
│           Saga Orchestrator             │
│                                         │
│  Step 1: Tell Booking Service: CREATE   │──→ Booking Service ✅
│  Step 2: Tell Inventory Service: RESERVE│──→ Inventory Service ✅
│  Step 3: Tell Payment Service: CHARGE   │──→ Payment Service ❌
│                                         │
│  Compensation:                          │
│  Step 2 undo: RELEASE inventory         │──→ Inventory Service ✅
│  Step 1 undo: CANCEL booking            │──→ Booking Service ✅
└─────────────────────────────────────────┘
```

**Pros:** Easy to track saga state, clear flow, centralized error handling.
**Cons:** Orchestrator is a single point of failure, tighter coupling to the orchestrator.

### Choreography vs Orchestration

| Aspect | Choreography | Orchestration |
|---|---|---|
| Coordination | Decentralized (events) | Centralized (orchestrator) |
| Coupling | Lower | Higher (to orchestrator) |
| Complexity | Harder to trace flow | Easier to understand |
| Failure handling | Distributed, complex | Centralized, simpler |
| Best for | Simple sagas (2-3 steps) | Complex sagas (4+ steps) |

**Logistics example:** A multi-leg freight booking involving booking confirmation → container allocation → customs declaration → carrier notification → insurance registration. This has 5+ steps with complex compensation → **Orchestration** is the better fit.

---

## Phase 10: Strangler Fig Pattern — Migrating from Monolith

### The Problem

You have a large monolith and want to migrate to microservices. You can't rewrite everything at once — it's too risky and too slow.

### What is the Strangler Fig Pattern?

Named after strangler fig trees that grow around a host tree and eventually replace it. You **gradually extract microservices** from the monolith, routing traffic to the new service while the old code remains as fallback.

```
Phase 1: All traffic → Monolith
  ┌──────────────────────────────┐
  │           Monolith           │
  │  Booking | Tracking | Billing│
  └──────────────────────────────┘

Phase 2: Extract Tracking
  ┌──────────────┐      ┌──────────────────┐
  │  API Gateway │─────→│ Tracking Service  │ (new microservice)
  │              │      └──────────────────┘
  │              │─────→│      Monolith      │
  │              │      │ Booking | Billing  │
  └──────────────┘      └──────────────────┘

Phase 3: Extract Billing
  ┌──────────────┐      ┌──────────────────┐
  │  API Gateway │─────→│ Tracking Service  │
  │              │─────→│ Billing Service   │ (new microservice)
  │              │─────→│     Monolith      │
  │              │      │     Booking       │
  └──────────────┘      └──────────────────┘

Phase 4: Monolith fully replaced
  All traffic → microservices, monolith decommissioned
```

### Implementation Steps

1. **Identify a bounded context** — pick a module with clear boundaries (e.g., Tracking)
2. **Build the microservice** — implement the new service alongside the monolith
3. **Route traffic** — use API Gateway to route `/api/tracking/*` to the new service
4. **Verify** — run both old and new in parallel, compare results (shadow mode)
5. **Cut over** — switch all traffic to the new service
6. **Remove old code** — delete the tracking module from the monolith

### Best Practices

- Start with the **least coupled** module (fewer dependencies = easier extraction)
- Use **feature flags** to gradually shift traffic (10% → 50% → 100%)
- Run in **shadow mode** first — both old and new process requests, compare results
- Don't change functionality during migration — just move it

---

## Phase 11: Sidecar Pattern

### What is it?

A **sidecar** is a helper container that runs alongside your main service container (in the same pod in Kubernetes). It handles cross-cutting concerns so your service doesn't have to.

```
┌──────────────────────────────────┐
│           Kubernetes Pod          │
│                                   │
│  ┌─────────────┐  ┌───────────┐  │
│  │   Booking   │  │  Sidecar  │  │
│  │   Service   │←→│  (Envoy)  │  │
│  │  (your code)│  │           │  │
│  └─────────────┘  └───────────┘  │
│                                   │
└──────────────────────────────────┘
```

### What the Sidecar Handles

| Concern | Without Sidecar | With Sidecar |
|---|---|---|
| **mTLS / encryption** | Each service implements TLS | Sidecar handles TLS transparently |
| **Retries / circuit breaking** | Code in every service | Sidecar handles automatically |
| **Logging** | Each service configures logging | Sidecar collects and ships logs |
| **Metrics** | Each service exposes metrics | Sidecar collects metrics |
| **Traffic management** | Manual load balancing | Sidecar routes intelligently |

### Service Mesh

When **every service gets a sidecar**, you have a **service mesh** — a dedicated infrastructure layer for service-to-service communication.

```
Service A ←→ Sidecar A ←→ Sidecar B ←→ Service B
                  ↕                          ↕
            Control Plane (Istio/Linkerd)
            (configures all sidecars centrally)
```

**Tools:** Istio (Envoy sidecar), Linkerd, Consul Connect

**When to use:** Large-scale microservices (50+ services) where implementing resilience patterns in every service is unsustainable.

---

## Phase 12: Distributed Tracing

### The Problem

A single user request passes through 5-10 microservices. When something fails or is slow, **where is the problem?**

```
User request → API Gateway → Booking → Rate → Inventory → Payment → Notification
                                            ↑
                                     Slow here? Failed here?
```

Logs from individual services don't show the full picture.

### What is Distributed Tracing?

A system that tracks a **single request's journey** across all microservices, measuring latency at each hop.

### Core Concepts

| Concept | Meaning |
|---|---|
| **Trace** | The entire journey of one request across all services |
| **Span** | One unit of work within a trace (e.g., "Rate Service → calculate rate") |
| **Trace ID** | Unique ID propagated across all services for this request |
| **Span ID** | Unique ID for this specific unit of work |
| **Parent Span** | The span that triggered this span |

```
Trace ID: abc-123

API Gateway [span-1] ──────────────────────────────── 500ms
  └── Booking Service [span-2] ────────────────────── 400ms
        ├── Rate Service [span-3] ─────────────────── 200ms
        │     └── DB Query [span-4] ───────────────── 150ms  ← Bottleneck!
        └── Inventory Service [span-5] ────────────── 50ms
```

### Implementation with Spring Boot + Micrometer Tracing

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-zipkin</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 1.0    # Sample 100% in dev, 10% in prod
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
```

Spring Boot automatically:
- Generates trace IDs and span IDs
- Propagates them via HTTP headers (`traceparent`)
- Adds them to logs (so you can search logs by trace ID)
- Exports to Zipkin/Jaeger for visualization

```
// Log output automatically includes trace context:
2024-01-15 10:30:45 [traceId=abc123, spanId=def456] INFO BookingService - Creating booking BK-001
2024-01-15 10:30:45 [traceId=abc123, spanId=ghi789] INFO RateService - Calculating rate for INMAA→USLAX
```

### Tools

| Tool | Type |
|---|---|
| **Zipkin** | Open-source, simple UI |
| **Jaeger** | Open-source (by Uber), rich features |
| **AWS X-Ray** | Managed, AWS-native |
| **Datadog APM** | Commercial, full observability |
| **Grafana Tempo** | Open-source, integrates with Grafana |

---

## Phase 13: Health Checks and Self-Healing

### Why Health Checks Matter

In microservices, services go up and down constantly. Load balancers, service registries, and orchestrators need to know if a service is **healthy** before routing traffic to it.

### Types of Health Checks

| Type | Checks | Example |
|---|---|---|
| **Liveness** | "Is the process running?" | JVM is alive, not deadlocked |
| **Readiness** | "Can it accept traffic?" | DB connection is up, cache is warmed |
| **Startup** | "Has it finished starting?" | Migrations completed, cache loaded |

### Spring Boot Actuator Health

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true    # Enables /actuator/health/liveness and /readiness
  health:
    circuitbreakers:
      enabled: true      # Show circuit breaker status in health
```

```
GET /actuator/health

{
  "status": "UP",
  "components": {
    "db": { "status": "UP", "details": { "database": "PostgreSQL" } },
    "redis": { "status": "UP" },
    "diskSpace": { "status": "UP" },
    "circuitBreakers": {
      "rateService": { "status": "UP", "state": "CLOSED" }
    }
  }
}
```

### Kubernetes Integration

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: booking-service
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            failureThreshold: 30
            periodSeconds: 10
```

**Self-healing:** If liveness probe fails → Kubernetes **restarts** the container. If readiness probe fails → Kubernetes **stops routing traffic** to it.

---

## Phase 14: Observability — The Three Pillars

### Logs, Metrics, Traces

| Pillar | What | Tool |
|---|---|---|
| **Logs** | Detailed event records | ELK (Elasticsearch, Logstash, Kibana), Loki |
| **Metrics** | Numerical measurements over time | Prometheus + Grafana |
| **Traces** | Request journey across services | Zipkin, Jaeger, Tempo |

### Structured Logging

```java
// ❌ Unstructured — hard to search/filter
log.info("Booking created for customer " + customerId + " shipment " + shipmentId);

// ✅ Structured — easily queryable
log.info("Booking created", 
    Map.of("customerId", customerId, "shipmentId", shipmentId, "carrier", carrier));
```

### Prometheus Metrics in Spring Boot

```yaml
management:
  metrics:
    export:
      prometheus:
        enabled: true
  endpoints:
    web:
      exposure:
        include: prometheus
```

```java
// Custom business metric
@Component
public class BookingMetrics {
    private final Counter bookingsCreated;
    private final Timer bookingDuration;

    public BookingMetrics(MeterRegistry registry) {
        this.bookingsCreated = Counter.builder("bookings.created.total")
            .description("Total bookings created")
            .tag("type", "freight")
            .register(registry);
        
        this.bookingDuration = Timer.builder("bookings.creation.duration")
            .description("Time to create a booking")
            .register(registry);
    }
}
```

### The Four Golden Signals (Google SRE)

| Signal | Question | Metric |
|---|---|---|
| **Latency** | How long do requests take? | p50, p95, p99 response time |
| **Traffic** | How many requests are we handling? | Requests/second |
| **Errors** | What fraction of requests fail? | Error rate (5xx/total) |
| **Saturation** | How full is the system? | CPU, memory, thread pool usage |

---

## Phase 15: When Microservices Are Overhead

### Scenario 1: Small Team, Simple Domain

**3 developers, simple CRUD app** — microservices mean each developer maintains infrastructure (Docker, CI/CD, monitoring) instead of writing features.

---

### Scenario 2: Startup MVP

**You need to ship fast.** Microservices force you to make architectural decisions before you understand your domain. Start with a well-structured monolith.

---

### Scenario 3: Tightly Coupled Operations

**All operations require data from all modules** — e.g., every request needs booking + tracking + billing data. You'll spend more time on inter-service communication than computation.

---

### Scenario 4: No DevOps Capability

**Microservices require:** container orchestration (Kubernetes), CI/CD pipelines per service, centralized logging, distributed tracing, service mesh. If you don't have DevOps maturity, you'll drown in operational complexity.

---

### The Microservices Overhead Checklist

- [ ] Do you have **multiple teams** that need independent deployment?
- [ ] Do different modules have **different scaling needs**?
- [ ] Is your domain **well-understood** with clear boundaries?
- [ ] Do you have **DevOps maturity** (CI/CD, K8s, monitoring)?
- [ ] Is the operational complexity **justified** by the benefits?

> **If you answered "No" to 3+ questions, stick with a monolith (or modular monolith).**

---

## Key Takeaways

1. **Start monolith, migrate to microservices** when you outgrow it (Strangler Fig)
2. **Service Discovery** lets services find each other dynamically (Eureka, Consul, K8s DNS)
3. **API Gateway** is the single entry point — handles auth, routing, rate limiting
4. **Circuit Breaker** prevents cascading failures — Resilience4j is the go-to for Spring Boot
5. **Bulkhead** isolates failures — one slow dependency doesn't take down everything
6. **Saga pattern** replaces distributed transactions — choreography for simple, orchestration for complex
7. **Distributed Tracing** (Zipkin/Jaeger) is essential — you can't debug what you can't see
8. **The Three Pillars** of observability: Logs + Metrics + Traces
9. **Don't over-architect** — microservices have massive operational overhead

---

**Next up:** [[CAP Theorem]]
