# Authentication and Authorization

> *"Authentication is about who you are. Authorization is about what you can do."*

---

## Phase 1: The Fundamentals

### Authentication vs Authorization

| | Authentication (AuthN) | Authorization (AuthZ) |
|---|---|---|
| **Question** | "Who are you?" | "What are you allowed to do?" |
| **Analogy** | Showing your ID at the airport | Your boarding pass says which gate/seat |
| **Input** | Credentials (username/password, token) | Identity + resource + action |
| **Output** | Identity (user ID, claims) | Allow or Deny |
| **Happens** | First | After authentication |

```
Request flow:
  Client → [Authentication: Who is this?] → [Authorization: Can they do this?] → Resource

Example:
  API request with JWT token:
    1. AuthN: Validate JWT → Identity: user "john@acme.com", role "SHIPPER"
    2. AuthZ: Can role "SHIPPER" access POST /api/bookings? → YES ✅
    3. AuthZ: Can role "SHIPPER" access DELETE /api/carriers? → NO ❌ (403)
```

---

## Phase 2: Authentication Methods

### Method 1: Session-Based Authentication

The traditional approach — server creates a **session** after login, stored server-side.

```
1. Client: POST /login { username: "john", password: "secret" }
2. Server: Validates credentials → Creates session in memory/DB
           Session ID: "sess_abc123"
3. Server: Returns Set-Cookie: SESSION_ID=sess_abc123
4. Client: Subsequent requests include cookie automatically
5. Server: Looks up SESSION_ID → finds user "john" → authenticated ✅
```

```
┌────────┐                    ┌───────────┐
│ Client │ ──Cookie:sess123─→ │  Server   │
│        │                    │           │
│        │                    │ Sessions: │
│        │                    │ sess123→  │
│        │                    │  john@acme│
└────────┘                    └───────────┘
```

**Pros:** Simple, easy to invalidate (delete session), session data stored securely on server.
**Cons:** Not scalable — sessions stored in server memory (sticky sessions needed or shared session store like Redis). Not suitable for APIs consumed by mobile/SPAs.

---

### Method 2: Token-Based Authentication (JWT)

Server issues a **self-contained token** after login. The token itself contains the user's identity and claims. Server doesn't store anything.

```
1. Client: POST /login { username: "john", password: "secret" }
2. Server: Validates credentials → Creates JWT token
3. Server: Returns { "token": "eyJhbG..." }
4. Client: Stores token (localStorage, secure cookie)
5. Client: Authorization: Bearer eyJhbG...
6. Server: Validates JWT signature → extracts user identity → authenticated ✅
```

### JWT Structure (JSON Web Token)

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJqb2huQGFjbWUuY29tIiwicm9sZSI6IlNISVBQRVIifQ.abc123

Header.Payload.Signature

Header (algorithm + type):
  { "alg": "HS256", "typ": "JWT" }

Payload (claims — the actual data):
  {
    "sub": "john@acme.com",        // Subject (user ID)
    "role": "SHIPPER",              // Custom claim
    "tenantId": "ACME",             // Custom claim
    "iat": 1705312200,              // Issued at
    "exp": 1705315800               // Expires at (1 hour later)
  }

Signature:
  HMAC-SHA256(base64(header) + "." + base64(payload), SECRET_KEY)
```

**The server signs the token with a secret key.** Anyone can READ the payload (it's base64, not encrypted), but only the server can CREATE or MODIFY it (because they need the secret key for the signature).

```java
// Spring Boot JWT creation
String token = Jwts.builder()
    .setSubject(user.getEmail())
    .claim("role", user.getRole())
    .claim("tenantId", user.getTenantId())
    .setIssuedAt(new Date())
    .setExpiration(new Date(System.currentTimeMillis() + 3600000))  // 1 hour
    .signWith(Keys.hmacShaKeyFor(secretKey.getBytes()), SignatureAlgorithm.HS256)
    .compact();
```

**Pros:** Stateless — server stores nothing. Scales perfectly across multiple servers. Works great for APIs, mobile, SPAs. Self-contained — carries user info.

**Cons:** Can't be revoked easily (token valid until expiry). Payload is readable (don't put secrets). Token size grows with claims.

### Session vs JWT Comparison

| Aspect | Session-Based | JWT |
|---|---|---|
| **State** | Stateful (server stores sessions) | Stateless (token is self-contained) |
| **Storage** | Server-side (memory/Redis) | Client-side (cookie/localStorage) |
| **Scalability** | Needs shared session store | Scales naturally (no server state) |
| **Revocation** | Easy (delete session) | Hard (must use blacklist or short TTL) |
| **Cross-domain** | Cookie-bound (CORS issues) | Bearer token works anywhere |
| **Mobile apps** | Awkward (cookies not natural) | Natural (Authorization header) |
| **Best for** | Traditional web apps | APIs, microservices, mobile, SPAs |

---

### Method 3: API Keys

A static, long-lived key assigned to a client application (not a user).

```http
GET /api/tracking/SHP-001
X-API-Key: ak_live_abc123def456
```

**Use case:** Machine-to-machine communication, partner integrations.

```java
@Component
public class ApiKeyFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                      HttpServletResponse response, 
                                      FilterChain chain) {
        String apiKey = request.getHeader("X-API-Key");
        
        ApiKeyEntity key = apiKeyRepository.findByKey(apiKey);
        if (key == null || key.isRevoked()) {
            response.setStatus(401);
            return;
        }
        
        SecurityContext context = SecurityContextHolder.createEmptyContext();
        context.setAuthentication(new ApiKeyAuthentication(key.getTenantId(), key.getPermissions()));
        SecurityContextHolder.setContext(context);
        
        chain.doFilter(request, response);
    }
}
```

**Pros:** Simple for partners. Easy to revoke per partner. Trackable (per-key analytics).
**Cons:** No user identity (it's the application, not a person). Must be kept secret. Usually paired with rate limiting.

---

## Phase 3: OAuth 2.0

### The Problem

You want to let a **third-party application** access your user's data without the user sharing their password.

```
❌ Without OAuth:
  Third-party app: "Give me your logistics platform password"
  User: "Okay... my password is 'secret123'"
  (Third-party now has full access forever. Can't revoke without changing password.)

✅ With OAuth:
  Third-party app: "I need access to your shipments"
  User: "Let me authorize you through the platform"
  Platform: "Here's a limited-scope, time-limited access token"
  (User can revoke access anytime. Third-party never sees the password.)
```

### OAuth 2.0 Roles

| Role | Who | Example |
|---|---|---|
| **Resource Owner** | The user who owns the data | John (shipper at ACME Corp) |
| **Client** | The third-party app requesting access | Carrier's tracking dashboard |
| **Authorization Server** | Issues tokens after user consent | Your auth service (or Keycloak/Okta) |
| **Resource Server** | Hosts the protected API | Your booking/tracking API |

### Authorization Code Flow (Most Common)

```
1. User clicks "Connect to Logistics Platform" in Carrier App

2. Carrier App redirects user to Authorization Server:
   GET https://auth.logistics.com/authorize?
     client_id=carrier-app&
     redirect_uri=https://carrier-app.com/callback&
     response_type=code&
     scope=shipments:read tracking:read&
     state=random123

3. User sees consent screen: "Carrier App wants to read your shipments. Allow?"
   User clicks "Allow"

4. Authorization Server redirects back to Carrier App:
   https://carrier-app.com/callback?code=AUTH_CODE_XYZ&state=random123

5. Carrier App exchanges auth code for tokens (server-to-server):
   POST https://auth.logistics.com/token
   { 
     grant_type: "authorization_code",
     code: "AUTH_CODE_XYZ",
     client_id: "carrier-app",
     client_secret: "carrier-secret"
   }

6. Authorization Server responds:
   {
     access_token: "eyJhbG...",    // Short-lived (15 min)
     refresh_token: "dGhpcyBp...",  // Long-lived (30 days)
     token_type: "Bearer",
     expires_in: 900,
     scope: "shipments:read tracking:read"
   }

7. Carrier App uses access token:
   GET https://api.logistics.com/shipments
   Authorization: Bearer eyJhbG...
```

### Client Credentials Flow (Machine-to-Machine)

No user involved — one service authenticates as itself to access another service's API.

```
POST https://auth.logistics.com/token
{
    grant_type: "client_credentials",
    client_id: "rate-service",
    client_secret: "rate-service-secret",
    scope: "carriers:read"
}

Response:
{
    access_token: "eyJhbG...",
    token_type: "Bearer",
    expires_in: 3600
}
```

**Use case:** Microservice-to-microservice communication. Rate Service needs to call Carrier Service.

---

### Access Token vs Refresh Token

| | Access Token | Refresh Token |
|---|---|---|
| **Purpose** | Authorize API requests | Get new access tokens |
| **Lifetime** | Short (15 min - 1 hour) | Long (days - months) |
| **Stored** | Client-side (memory preferred) | Client-side (secure storage) |
| **Sent to** | Resource Server (API) | Authorization Server only |
| **If stolen** | Limited damage (expires soon) | More dangerous (can mint new access tokens) |

```
Token refresh flow:
  Access token expired → Client sends refresh token to auth server
  → Auth server validates refresh token → Issues new access + refresh token
  → Old refresh token invalidated (rotation)
```

---

## Phase 4: Authorization Models

### Model 1: Role-Based Access Control (RBAC)

Users are assigned **roles**, and roles have **permissions**.

```
Roles:
  ADMIN    → can do everything
  SHIPPER  → can create/view bookings, view tracking
  CARRIER  → can update tracking, view assigned shipments
  VIEWER   → can only read

User "john@acme.com" → Role: SHIPPER
  ✅ POST /api/bookings (create booking)
  ✅ GET /api/tracking/SHP-001 (view tracking)
  ❌ DELETE /api/carriers/MAERSK (admin only)
```

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.POST, "/api/bookings/**").hasAnyRole("ADMIN", "SHIPPER")
                .requestMatchers(HttpMethod.GET, "/api/tracking/**").authenticated()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}

// Method-level security
@PreAuthorize("hasRole('SHIPPER') or hasRole('ADMIN')")
@PostMapping("/api/bookings")
public Booking createBooking(@RequestBody BookingRequest request) { ... }
```

**Pros:** Simple, well-understood, easy to audit.
**Cons:** Role explosion (too many roles for fine-grained control). Doesn't handle resource-level access well.

---

### Model 2: Attribute-Based Access Control (ABAC)

Access decisions based on **attributes** of the user, resource, action, and environment.

```
Policy: "A shipper can cancel a booking only if:
  - They own the booking (user.tenantId == booking.tenantId)
  - The booking is in PENDING status
  - The request is made during business hours"

Attributes:
  User:     { role: "SHIPPER", tenantId: "ACME" }
  Resource: { type: "booking", tenantId: "ACME", status: "PENDING" }
  Action:   "CANCEL"
  Context:  { time: "14:30", ip: "10.0.0.1" }

Decision: ALLOW ✅ (all conditions met)
```

**Pros:** Extremely fine-grained. Handles complex real-world policies.
**Cons:** Complex to implement and maintain. Harder to audit.

---

### Model 3: Resource-Based Access Control

Authorization is checked against the **specific resource** being accessed.

```java
@PostMapping("/api/bookings/{id}/cancel")
public void cancelBooking(@PathVariable String id, @AuthenticationPrincipal UserDetails user) {
    Booking booking = bookingService.findById(id);
    
    // Resource-level check: does this user's tenant own this booking?
    if (!booking.getTenantId().equals(user.getTenantId())) {
        throw new AccessDeniedException("Not your booking");
    }
    
    // Business rule check: can this booking be cancelled?
    if (booking.getStatus() != BookingStatus.PENDING) {
        throw new IllegalStateException("Only pending bookings can be cancelled");
    }
    
    bookingService.cancel(id);
}
```

---

## Phase 5: Auth in Microservices

### The Challenge

In a monolith, one security filter handles everything. In microservices, **each service needs to verify identity and enforce permissions**.

```
❌ Each service calls auth server:
  Client → Booking Service → calls Auth Server (validate token)
  Client → Tracking Service → calls Auth Server (validate token)
  Client → Billing Service → calls Auth Server (validate token)
  
  Auth Server becomes a bottleneck! Every request = extra network hop.
```

### Pattern 1: API Gateway Authentication

Gateway validates the token **once**. Backend services trust the gateway.

```
Client → [API Gateway: validates JWT] → forwards request with user info
  → Booking Service (trusts gateway, reads X-User-Id header)
  → Tracking Service (trusts gateway, reads X-User-Id header)
```

```yaml
# Spring Cloud Gateway validates JWT
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.logistics.com
```

Gateway adds trusted headers:
```
X-User-Id: john@acme.com
X-Tenant-Id: ACME
X-User-Roles: SHIPPER
```

Backend services read these headers (no token validation needed):
```java
@GetMapping("/api/bookings")
public List<Booking> getBookings(@RequestHeader("X-Tenant-Id") String tenantId) {
    return bookingService.findByTenant(tenantId);
}
```

⚠️ **Security requirement:** Backend services must ONLY accept requests from the gateway (network-level restriction via VPC/security groups).

---

### Pattern 2: JWT Validation at Each Service

Each service independently validates the JWT. No gateway dependency.

```
Client → Booking Service (validates JWT itself)
Client → Tracking Service (validates JWT itself)

Each service has the JWT public key and validates independently.
```

```java
// Each service configured as OAuth2 resource server
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> 
            jwt.jwkSetUri("https://auth.logistics.com/.well-known/jwks.json")))
        .build();
}
```

**Pros:** No single point of failure. Each service is independently secure.
**Cons:** Each service needs JWT validation logic. Token can't be revoked until expiry.

---

### Pattern 3: Service-to-Service Authentication

When Booking Service calls Rate Service internally, how does Rate Service know it's a legitimate call?

**Option A: Propagate user's JWT**
```
Client (JWT: user) → Booking Service → passes same JWT → Rate Service
Rate Service validates JWT → knows the original user
```

**Option B: Service credentials (Client Credentials flow)**
```
Booking Service → gets its own token via client_credentials grant
  → calls Rate Service with service-level token
Rate Service: "This is Booking Service calling, it has permission to read rates"
```

**Option C: mTLS (Mutual TLS)**
```
Booking Service ←→ Rate Service
Both present TLS certificates to each other.
Identity verified at the transport layer — no tokens needed.
```

Used in service meshes (Istio) — mTLS between all services automatically.

---

## Phase 6: Token Security Best Practices

### Token Storage (Frontend)

| Storage | XSS Vulnerable? | CSRF Vulnerable? | Recommendation |
|---|---|---|---|
| **localStorage** | Yes ❌ | No | Avoid for sensitive tokens |
| **sessionStorage** | Yes ❌ | No | Slightly better (tab-scoped) |
| **HttpOnly Cookie** | No ✅ | Yes (mitigate with CSRF token) | **Recommended** |
| **In-memory (variable)** | No ✅ | No | Best security, lost on refresh |

**Best practice:** Store access token **in memory** (JavaScript variable). Store refresh token in **HttpOnly, Secure, SameSite=Strict cookie**.

### Token Revocation

JWTs are stateless — once issued, they're valid until expiry. What if a user logs out or is compromised?

**Strategy 1: Short-lived tokens (15 min)**
```
Access token TTL: 15 minutes
If stolen, attacker has at most 15 minutes of access.
User logs out → refresh token revoked → no new access tokens after 15 min.
```

**Strategy 2: Token blacklist**
```java
// On logout, add token to blacklist (Redis with TTL)
public void logout(String tokenId) {
    redis.opsForValue().set("blacklist:" + tokenId, "revoked", 
        Duration.ofMinutes(15));  // TTL matches token expiry
}

// On every request, check blacklist
public boolean isRevoked(String tokenId) {
    return redis.hasKey("blacklist:" + tokenId);
}
```

**Strategy 3: Refresh token rotation**
```
Every time a refresh token is used, a NEW refresh token is issued.
The old one is invalidated.
If attacker steals a refresh token and the legitimate user also uses it:
  → One of them gets an "already used" error → fraud detected → all tokens revoked.
```

---

## Phase 7: Multi-Tenancy Security

In logistics SaaS, **tenant isolation** is critical — Tenant A must never see Tenant B's shipments.

### Data Isolation at Every Layer

```java
// Repository level: always filter by tenant
@Query("SELECT s FROM Shipment s WHERE s.tenantId = :tenantId AND s.id = :id")
Optional<Shipment> findByIdAndTenant(@Param("id") String id, @Param("tenantId") String tenantId);

// Service level: extract tenant from security context
@Service
public class ShipmentService {
    public Shipment getShipment(String shipmentId) {
        String tenantId = SecurityContextHolder.getContext()
            .getAuthentication().getTenantId();
        return shipmentRepo.findByIdAndTenant(shipmentId, tenantId)
            .orElseThrow(() -> new NotFoundException("Shipment not found"));
        // Note: if shipment belongs to another tenant, returns 404 (not 403)
        // This prevents tenant enumeration attacks
    }
}
```

### ⚠️ Common Vulnerability: Insecure Direct Object Reference (IDOR)

```
❌ VULNERABLE:
  GET /api/shipments/SHP-001
  → Returns shipment regardless of who's asking
  → Attacker guesses SHP-002, SHP-003... → sees other tenants' data!

✅ SECURE:
  GET /api/shipments/SHP-001
  → Server checks: does SHP-001 belong to the requesting tenant?
  → Yes → return data
  → No → return 404 (not 403, to prevent enumeration)
```

---

## Phase 8: When Auth is an Overhead

### Scenario 1: Internal Service-to-Service in Trusted Network

Services communicating within a VPC with network-level security (security groups). Adding JWT validation to every internal call adds latency for minimal security benefit.

**Alternative:** Network-level security + mTLS in a service mesh.

---

### Scenario 2: Public, Non-Sensitive APIs

A public API that returns non-sensitive, publicly available data (weather, public transit schedules).

**Alternative:** API key for rate limiting only, no user authentication.

---

### Scenario 3: Prototyping / MVP

Early-stage product where speed matters more than security polish.

**Alternative:** Basic auth or simple API keys. Add proper auth before going to production.

---

## Key Takeaways

1. **JWT is the standard for API authentication** — stateless, scalable, works across services
2. **OAuth 2.0 Authorization Code flow** for third-party access; **Client Credentials** for service-to-service
3. **Short-lived access tokens (15 min) + refresh tokens** = best balance of security and UX
4. **RBAC** for simple role-based access; **ABAC** for complex, attribute-based policies
5. **API Gateway authentication** — validate once at the edge, trust internally
6. **Tenant isolation is non-negotiable** in multi-tenant SaaS — filter EVERYTHING by tenant
7. **Never store JWTs in localStorage** — use HttpOnly cookies or in-memory
8. **Token rotation** and **blacklisting** solve the JWT revocation problem
9. **Return 404 (not 403)** for resources belonging to other tenants — prevent enumeration

---

**See also:** [[Rate Limiting and Throttling]] (API key + rate limiting), [[Microservices Patterns]] (API Gateway, Service Mesh)
