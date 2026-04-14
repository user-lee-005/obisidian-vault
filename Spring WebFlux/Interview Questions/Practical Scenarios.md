### ❓ Design a real-time notification system

**Answer:**

- Use WebFlux + SSE/WebSocket
- Non-blocking pipeline
- Event-driven architecture

---

### ❓ How to handle 50k concurrent users?

**Answer:**

- Event loop (Netty)
- Non-blocking APIs
- Backpressure handling
- Avoid blocking DB calls

---

### ❓ How to debug reactive pipelines?

**Answer:**

- `.log()`
- `checkpoint()`
- Enable Reactor debug mode