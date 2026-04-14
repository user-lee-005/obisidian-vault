## 1. What actually happens in Spring Boot

### Spring MVC (traditional)

- Default server → **Apache Tomcat**
- Model → **thread-per-request**
- Built on → Servlet API

---

### Spring WebFlux (reactive)

Default server → **Netty**

But here’s the important part:

👉 **WebFlux is NOT tied to Netty**  
It supports multiple runtimes:

- Netty (default)
- Tomcat (reactive adapter)
- Jetty
- Undertow

---

## 2. Then why does everyone say “WebFlux = Netty”?

Because in practice:

### Netty is:

- Fully **non-blocking**
- Built for **event loops**
- No Servlet baggage
- Designed for high concurrency

So when you use WebFlux with default config:

spring-boot-starter-webflux

👉 Spring Boot auto-configures **Netty**

---

## 3. Can WebFlux run on Tomcat?

Yes — but here’s the catch:

You can run WebFlux on:

- **Apache Tomcat**

But:

- It uses a **Servlet 3.1+ non-blocking API**
- Still carries Servlet overhead
- Not as efficient as Netty for event-loop workloads

👉 So technically possible, but not optimal.

---

## 4. Core difference (this is what actually matters)

### Tomcat model:

- Thread per request
- Blocking I/O (by default)
- Even async features still rely on thread pools

### Netty model:

- Few threads (event loop)
- Handles thousands of connections
- Non-blocking from the ground up

---

## 5. Real mental model (don’t mess this up)

- **WebFlux = programming model (reactive)**
- **Netty = runtime engine (event loop server)**

They are NOT the same thing.

---

## 6. Brutal truth

If you:

- Use WebFlux + Netty → you get real benefits
- Use WebFlux + blocking DB (JPA) → you just shot yourself in the foot
- Use WebFlux + Tomcat → you lose part of the advantage

---

## 7. When Netty actually matters

Netty shines when:

- 10k+ concurrent connections
- Long-lived connections (SSE, WebSockets)
- Streaming APIs
- Gateway / proxy services

[[Replace Netty in Configuration]]
