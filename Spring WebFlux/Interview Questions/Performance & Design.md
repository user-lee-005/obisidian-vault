### ❓ When should you NOT use WebFlux?

**Answer:**

- DB-heavy apps (JPA)
- CPU-bound workloads
- Low concurrency systems

---

### ❓ Why is WebFlux scalable?

**Answer:**

- No thread-per-request
- Uses event loop
- Efficient handling of I/O-bound tasks