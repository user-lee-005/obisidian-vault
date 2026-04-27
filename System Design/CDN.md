# CDN (Content Delivery Network)

> *"The fastest request is the one that never reaches your server."*

---

## Phase 1: The Fundamentals

### What is a CDN?

A CDN is a **geographically distributed network of servers** (called edge servers or PoPs — Points of Presence) that cache and deliver content to users from the **nearest location**, reducing latency and offloading traffic from your origin server.

```
Without CDN:
  User in London → Request travels to Origin Server in Mumbai (200ms latency)
  
With CDN:
  User in London → Request served from CDN Edge in London (20ms latency)
```

**Analogy:** Imagine a global logistics company with a **central warehouse in Mumbai**. Every order worldwide ships from Mumbai — slow for London customers. Now add **regional warehouses** in London, New York, and Tokyo. Common items are stocked locally. Orders are fulfilled from the nearest warehouse. That's a CDN — regional caches for your content.

```
                    ┌───────────────┐
                    │ Origin Server │ (Mumbai)
                    │ (Your Server) │
                    └───────┬───────┘
                            │
              ┌─────────────┼─────────────┐
              ↓             ↓             ↓
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ CDN Edge │  │ CDN Edge │  │ CDN Edge │
        │ (London) │  │(New York)│  │ (Tokyo)  │
        └──────────┘  └──────────┘  └──────────┘
              ↑             ↑             ↑
          UK Users     US Users     Japan Users
```

### Why Do We Need a CDN?

| Problem | How CDN Helps |
|---|---|
| **High latency** | Content served from nearest edge (20ms vs 200ms) |
| **Origin server overload** | CDN handles 90%+ of traffic; origin handles only cache misses |
| **Traffic spikes** | CDN absorbs load during peak (Black Friday, product launch) |
| **DDoS protection** | Distributed network absorbs attack traffic across many PoPs |
| **Bandwidth costs** | CDN providers negotiate bulk bandwidth cheaper than you can |
| **Global availability** | Content available even if origin goes down (stale-while-revalidate) |

### What Can a CDN Serve?

| Content Type | Examples | Cache Duration |
|---|---|---|
| **Static assets** | CSS, JS, images, fonts, videos | Hours to months |
| **HTML pages** | Marketing pages, blog posts | Minutes to hours |
| **API responses** | Public endpoints (product catalog, tracking status) | Seconds to minutes |
| **Downloads** | Software binaries, PDFs, reports | Days to months |
| **Streaming** | Video (HLS/DASH segments), live streams | Segment-level caching |

---

## Phase 2: How a CDN Works — The Request Flow

### First Request (Cache Miss)

```
1. User in London requests: https://cdn.logistics.com/images/logo.png
2. DNS resolves cdn.logistics.com → nearest CDN edge (London PoP)
3. London Edge checks local cache → MISS (not cached yet)
4. London Edge fetches from Origin Server (Mumbai) → 200ms
5. Origin responds with logo.png + Cache-Control headers
6. London Edge caches the file and serves to user
7. Total latency: ~250ms (first request)
```

### Subsequent Requests (Cache Hit)

```
1. Another London user requests: https://cdn.logistics.com/images/logo.png
2. DNS → London Edge
3. London Edge checks cache → HIT ✅
4. Serves directly from edge cache → 10ms latency
5. Origin server is never contacted
```

### Cache Expiry and Revalidation

```
TTL expires on London Edge:

Option 1: Serve stale, revalidate in background (stale-while-revalidate)
  → User gets instant response (stale content)
  → Edge fetches fresh content from origin in background

Option 2: Revalidate before serving
  → Edge asks origin: "Has this changed?" (If-None-Match / ETag)
  → Origin responds 304 Not Modified → Edge serves cached version
  → OR Origin responds 200 with new content → Edge updates cache
```

---

## Phase 3: CDN Caching Mechanisms

### Cache-Control Headers

The origin server tells the CDN **how to cache** content via HTTP headers.

```http
# Cache for 1 hour on CDN and browser
Cache-Control: public, max-age=3600

# Cache on CDN for 1 day, browser for 1 hour
Cache-Control: public, max-age=3600, s-maxage=86400

# Don't cache on CDN (private/personalized content)
Cache-Control: private, max-age=3600

# Don't cache at all
Cache-Control: no-store

# Always revalidate with origin before serving
Cache-Control: no-cache

# Serve stale content while revalidating in background
Cache-Control: public, max-age=3600, stale-while-revalidate=60
```

**Key directives:**

| Directive | Meaning |
|---|---|
| `public` | CDN AND browser can cache |
| `private` | Only browser can cache (not CDN) — for user-specific data |
| `max-age=N` | Cache for N seconds (browser + CDN) |
| `s-maxage=N` | Cache for N seconds on CDN only (overrides max-age for CDN) |
| `no-cache` | Must revalidate with origin every time (still stored, but checked) |
| `no-store` | Don't cache anywhere at all |
| `stale-while-revalidate=N` | Serve stale for N seconds while fetching fresh in background |
| `stale-if-error=N` | Serve stale for N seconds if origin is down |

### ETag (Entity Tag)

A fingerprint of the content. Used for conditional requests.

```
First request:
  Response: ETag: "abc123", Content: [logo.png data]

Cache expired, revalidation:
  Request:  If-None-Match: "abc123"
  Response: 304 Not Modified (content hasn't changed, use cache)
  
  OR
  
  Response: 200 OK, ETag: "def456" (content changed, here's the new version)
```

### Vary Header

Tells the CDN to cache **different versions** based on request headers.

```http
Vary: Accept-Encoding
# Cache separate versions for gzip, br, and uncompressed

Vary: Accept-Language
# Cache separate versions for en, fr, de

Vary: Accept-Encoding, Accept-Language
# Cache separate versions for each combination
```

⚠️ **Avoid `Vary: *`** — it effectively disables CDN caching.

---

## Phase 4: CDN Architecture Patterns

### Pattern 1: Pull-Based CDN (Origin Pull)

CDN fetches content from origin **on demand** (on first cache miss).

```
User → CDN Edge: "Give me /images/logo.png"
CDN Edge: "Don't have it. Let me fetch from origin."
CDN Edge → Origin: GET /images/logo.png
Origin → CDN Edge: [data]
CDN Edge → caches it → serves to user
```

**Pros:** Simple setup — just point CDN at your origin.
**Cons:** First request is slow (cache miss → origin fetch).
**Used by:** Cloudflare, AWS CloudFront, Fastly (default mode).

---

### Pattern 2: Push-Based CDN

You **proactively upload** content to CDN edges before users request it.

```
Deploy script: Upload all static assets to CDN storage
  → logo.png pushed to all 200 edge locations
  → CSS/JS bundles pushed to all edges

User → CDN Edge: "Give me /images/logo.png"
CDN Edge: "Already have it!" → serves immediately
```

**Pros:** No first-request penalty — content is pre-populated.
**Cons:** You must manage what's pushed; stale content if not updated.
**Used by:** AWS S3 + CloudFront, Azure CDN (when using blob storage).

---

### Pattern 3: Pull + Push Hybrid

Push critical assets, pull everything else.

```
Deploy: Push CSS, JS bundles, critical images → all edges
Runtime: API responses, user uploads → pulled on demand
```

**This is the most common production setup.**

---

### Pattern 4: CDN as a Reverse Proxy

CDN sits in front of your **entire application** — not just static assets.

```
ALL requests go through CDN:
  Static assets → served from edge cache
  API requests → passed through to origin (with optional caching)
  
  Client → CDN Edge → Origin API Server
                    → CDN caches GET /api/tracking/SHP-001 for 30 seconds
```

**Tools:** Cloudflare (default mode), Fastly, AWS CloudFront.

---

## Phase 5: CDN for API Responses

CDNs aren't just for images and CSS — you can cache **API responses** too.

### When to Cache API Responses

| API Endpoint | Cacheable? | TTL | Why |
|---|---|---|---|
| `GET /api/carriers` | ✅ Yes | 1 hour | Carrier list rarely changes |
| `GET /api/ports` | ✅ Yes | 24 hours | Port codes almost never change |
| `GET /api/tracking/{id}` | ⚠️ Carefully | 30 seconds | Status changes, but stale is OK briefly |
| `POST /api/bookings` | ❌ No | 0 | Mutation — never cache |
| `GET /api/user/profile` | ❌ No | 0 | Personalized — different per user |
| `GET /api/rates?from=X&to=Y` | ✅ Yes | 15 minutes | Same route = same rate for a period |

### Implementing API Caching

```java
@GetMapping("/api/carriers")
public ResponseEntity<List<Carrier>> getCarriers() {
    List<Carrier> carriers = carrierService.findAllActive();
    return ResponseEntity.ok()
        .cacheControl(CacheControl.maxAge(Duration.ofHours(1))
                                   .cachePublic()
                                   .staleWhileRevalidate(Duration.ofMinutes(5)))
        .body(carriers);
}

@GetMapping("/api/tracking/{id}")
public ResponseEntity<TrackingInfo> getTracking(@PathVariable String id) {
    TrackingInfo info = trackingService.getStatus(id);
    return ResponseEntity.ok()
        .cacheControl(CacheControl.maxAge(Duration.ofSeconds(30))
                                   .cachePublic())
        .eTag(info.getVersion())
        .body(info);
}

@PostMapping("/api/bookings")
public ResponseEntity<Booking> createBooking(@RequestBody BookingRequest req) {
    // Mutations are NEVER cached
    return ResponseEntity.ok()
        .cacheControl(CacheControl.noStore())
        .body(bookingService.create(req));
}
```

### Cache Key for API Responses

CDN caches by URL + selected headers. Ensure different requests get different cache entries:

```
Same URL, different responses:
  GET /api/rates?from=INMAA&to=USLAX    → Rate: $2,500
  GET /api/rates?from=INMAA&to=GBFXT    → Rate: $1,800

These are different URLs → automatically different cache entries ✅

Same URL, different users:
  GET /api/profile  (User A) → User A's data
  GET /api/profile  (User B) → User B's data

These must NOT be cached by CDN! → Use Cache-Control: private
```

---

## Phase 6: Cache Invalidation on CDN

The hardest problem — when you update content on origin, how do you remove stale content from 200+ edge servers worldwide?

### Strategy 1: TTL-Based Expiry

Just set a short TTL and let it expire naturally.

```
Cache-Control: max-age=60  → Content refreshes every 60 seconds
```

**Pros:** Simple, no active invalidation needed.
**Cons:** Users see stale content for up to the full TTL duration.

---

### Strategy 2: Purge / Invalidation API

Actively tell the CDN to remove specific cached content.

```bash
# Cloudflare: Purge specific URL
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone}/purge_cache" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"files": ["https://cdn.logistics.com/api/carriers"]}'

# AWS CloudFront: Create invalidation
aws cloudfront create-invalidation \
  --distribution-id E1234567890 \
  --paths "/api/carriers" "/images/logo.png"
```

**Pros:** Immediate effect.
**Cons:** Purging is not instant (propagation takes seconds to minutes). Costs money (AWS charges per invalidation). Rate limited.

---

### Strategy 3: Versioned URLs (Cache Busting)

Include a version/hash in the URL. New version = new URL = new cache entry.

```
v1: /assets/app.abc123.js  → cached forever
v2: /assets/app.def456.js  → new URL, no stale cache issue

Or with query parameter:
  /api/carriers?v=20240115  → cached
  /api/carriers?v=20240116  → new cache entry
```

**Pros:** No purging needed. Old and new versions can coexist.
**Cons:** Requires build tooling to generate versioned filenames.

**This is the standard approach for static assets** — tools like Webpack, Vite add content hashes to filenames automatically.

---

### Strategy 4: Stale-While-Revalidate

Serve stale content immediately, fetch fresh content in the background.

```
Cache-Control: max-age=60, stale-while-revalidate=30

T=0: Content cached, max-age=60
T=60: Content expired, but still within stale-while-revalidate window
  → User gets stale content instantly
  → Edge fetches fresh content from origin in background
T=61: Fresh content arrives → edge cache updated
T=90: stale-while-revalidate window expires
  → If still not refreshed, must fetch from origin before serving
```

**This is the best balance** of freshness and performance for most use cases.

---

## Phase 7: CDN Security

### DDoS Protection

CDN distributes attack traffic across hundreds of PoPs worldwide:

```
Without CDN:
  Attacker → 10 Gbps traffic → Your origin (1 Gbps capacity) → DOWN 💥

With CDN:
  Attacker → 10 Gbps traffic → Distributed across 200 CDN PoPs
  → Each PoP handles 50 Mbps → Within capacity → Origin protected ✅
```

### WAF (Web Application Firewall)

CDN-integrated WAF filters malicious requests at the edge:

```
Request → CDN Edge → WAF checks:
  ✅ Normal request → forward to origin
  ❌ SQL injection attempt → blocked
  ❌ XSS payload → blocked
  ❌ Bot traffic → challenged with CAPTCHA
```

### HTTPS / TLS Termination

CDN handles TLS encryption at the edge:

```
User ──HTTPS──→ CDN Edge (TLS terminated here)
                ──HTTP or HTTPS──→ Origin

Benefits:
  - CDN manages SSL certificates (auto-renewal)
  - Edge is closer → TLS handshake is faster
  - Origin doesn't need to handle TLS overhead
```

### Signed URLs / Tokens

Restrict access to content — only authorized users can fetch it:

```
Generate signed URL:
  https://cdn.logistics.com/docs/BOL-SHP001.pdf?token=abc123&expires=1672531260

CDN Edge validates:
  - Is the token valid? ✅
  - Has it expired? No ✅
  → Serve content

Without valid token → 403 Forbidden
```

---

## Phase 8: CDN Providers Comparison

| Provider | PoPs | Strengths | Pricing Model | Best For |
|---|---|---|---|---|
| **Cloudflare** | 300+ | DDoS, WAF, Workers (edge compute), free tier | Free tier + paid plans | Most use cases, budget-friendly |
| **AWS CloudFront** | 400+ | Deep AWS integration, Lambda@Edge | Pay per request + bandwidth | AWS-native apps |
| **Fastly** | 80+ | Real-time purging (<150ms), Compute@Edge | Pay per request + bandwidth | Dynamic content, APIs |
| **Akamai** | 4,000+ | Largest network, enterprise features | Enterprise pricing | Large enterprises |
| **Azure CDN** | 170+ | Azure integration, multiple CDN providers | Pay per bandwidth | Azure-native apps |
| **Google Cloud CDN** | 150+ | GCP integration, Anycast | Pay per bandwidth | GCP-native apps |

### Decision Framework

```
Need free tier + DDoS protection?            → Cloudflare
Already on AWS?                              → CloudFront
Need instant purging for dynamic content?    → Fastly
Enterprise with massive scale?               → Akamai
Already on Azure/GCP?                        → Azure CDN / Cloud CDN
```

---

## Phase 9: Edge Computing (CDN + Compute)

Modern CDNs don't just cache — they can **run code at the edge**.

### What is Edge Compute?

Instead of just caching static content, run **small functions** at CDN edge locations — closer to the user.

```
Traditional:
  User → CDN Edge (cache only) → Origin (all logic)

Edge Compute:
  User → CDN Edge (cache + run logic) → Origin (only when needed)
```

### Use Cases

| Use Case | Implementation | Benefit |
|---|---|---|
| **A/B testing** | Route 50% of users to variant A at the edge | No origin involvement |
| **Geo-routing** | Redirect to regional API based on user location | Zero-latency routing |
| **Auth validation** | Validate JWT at edge, reject unauthorized early | Origin never sees bad requests |
| **Header manipulation** | Add/modify headers based on device type | Personalization at edge |
| **URL rewriting** | Transform legacy URLs to new format | No origin code change |

### Tools

| CDN | Edge Compute Feature |
|---|---|
| Cloudflare | **Workers** (JavaScript/WASM) |
| AWS CloudFront | **Lambda@Edge** / **CloudFront Functions** |
| Fastly | **Compute@Edge** (WASM) |
| Vercel | **Edge Functions** |

---

## Phase 10: When CDN is an Overhead

### Scenario 1: Internal / Private APIs

APIs only accessed by internal microservices within the same VPC. CDN adds latency (extra hop) for no benefit.

---

### Scenario 2: Highly Personalized Content

Every response is unique per user (dashboard, inbox, profile). Cache hit ratio = 0%.

```
GET /api/dashboard (User A) → unique response
GET /api/dashboard (User B) → different unique response

CDN caches nothing useful. Extra hop adds latency.
```

---

### Scenario 3: Real-Time Data

WebSocket connections, live trading data, real-time chat. Caching is meaningless when every message is new.

---

### Scenario 4: Single-Region Users

All users are in one city/region. A CDN with 200 global PoPs provides no geographic benefit. A simple reverse proxy (Nginx) is cheaper.

---

## Phase 11: Real-World Case Studies

### Case 1: Logistics — Tracking Portal CDN

**Problem:** A freight tracking portal is accessed by customers in 40 countries. Static assets (JS, CSS, images) load slowly for non-Asian users because the origin is in Mumbai.

**Solution:**
```
Cloudflare CDN:
  Static assets: Cache forever (versioned filenames)
  API GET /tracking/{id}: Cache 30 seconds (stale-while-revalidate: 10s)
  API POST /bookings: Bypass CDN (no-store)
  
WAF rules: Block known bot signatures, rate limit by IP
```

**Result:** Page load time dropped from 4.5 sec (London) to 1.2 sec. Origin bandwidth reduced by 85%.

---

### Case 2: Logistics — Document Download CDN

**Problem:** Customers download Bills of Lading (BOL), invoices, and customs documents. PDFs are 2-5 MB each. 50,000 downloads/day during peak overload the origin.

**Solution:**
```
AWS CloudFront + S3:
  Documents stored in S3 → served via CloudFront
  Signed URLs: Each document link expires in 24 hours (security)
  Cache: 7 days (documents are immutable once generated)
```

**Result:** Origin handles only document generation. All downloads served from CDN. S3 bandwidth costs dropped 70%.

---

## Key Takeaways

1. **CDN caches content at the edge** — closest to the user, for lowest latency
2. **Use `Cache-Control` headers** to tell CDN what and how long to cache
3. **Pull CDN is simplest** — CDN fetches from origin on cache miss
4. **Cache APIs too** — not just static assets; GET endpoints with `s-maxage`
5. **Versioned URLs** for static assets — no purging headaches
6. **`stale-while-revalidate`** is the best balance of freshness and speed
7. **CDN provides security** — DDoS absorption, WAF, TLS termination
8. **Edge compute** is the future — run auth, routing, and A/B tests at the edge
9. **Don't CDN everything** — private APIs, personalized content, and real-time data don't benefit

---

**See also:** [[Caching]] (Layer 2: CDN Caching), [[Rate Limiting and Throttling]], [[Load Balancers]]
