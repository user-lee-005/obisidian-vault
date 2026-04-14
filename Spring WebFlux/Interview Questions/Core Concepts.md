### ❓ What is Spring WebFlux?

**Answer:**  
Spring WebFlux is a reactive web framework built on non-blocking I/O and the Reactive Streams specification.  
It uses an **event-loop model** instead of thread-per-request and supports asynchronous data streams using `Mono` and `Flux`.

---

### ❓ Difference between Spring MVC and WebFlux?

**Answer:**

|Aspect|MVC|WebFlux|
|---|---|---|
|Model|Thread per request|Event loop|
|IO|Blocking|Non-blocking|
|Scalability|Limited by threads|High concurrency|
|Return types|Objects|Mono / Flux|

---

### ❓ What are Mono and Flux?

**Answer:**  
From Project Reactor:

- `Mono<T>` → emits 0 or 1 item
- `Flux<T>` → emits 0 to N items  
    They represent asynchronous streams of data.

---

### ❓ What is backpressure?

**Answer:**  
Backpressure is a mechanism where the **consumer controls how fast the producer emits data**, preventing memory overflow and system overload.