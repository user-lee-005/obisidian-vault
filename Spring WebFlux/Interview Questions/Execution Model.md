### ❓ How does WebFlux handle requests internally?

**Answer:**  
WebFlux uses an **event-loop model** (typically via Netty):

- Few threads handle many requests
- Uses non-blocking I/O
- Execution is callback-driven instead of blocking

---

### ❓ What happens if you use blocking code in WebFlux?

**Answer:**  
Blocking code blocks the event loop → reduces throughput → kills scalability.  
It must be offloaded to `boundedElastic` scheduler.

---

### ❓ What is a Scheduler in Reactor?

**Answer:**  
Schedulers control thread execution:

- `immediate` → current thread
- `single` → one reusable thread
- `parallel` → CPU-bound work
- `boundedElastic` → blocking operations (important)