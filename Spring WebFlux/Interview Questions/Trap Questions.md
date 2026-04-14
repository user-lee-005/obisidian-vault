### ❓ Does `produces = TEXT_EVENT_STREAM_VALUE` make it non-blocking?

**Answer:**  
No.  
Non-blocking depends on:

- Runtime (Netty)
- Code (no blocking calls)

---

### ❓ Is WebFlux always faster than MVC?

**Answer:**  
No.

- Better for I/O-bound workloads
- Worse for CPU-bound or blocking tasks

---

### ❓ Can WebFlux run on Tomcat?

**Answer:**  
Yes, but:

- Uses servlet async model
- Less efficient than Netty