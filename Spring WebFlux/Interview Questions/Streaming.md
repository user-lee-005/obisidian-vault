### ❓ What is `TEXT_EVENT_STREAM_VALUE`?

**Answer:**  
It enables **Server-Sent Events (SSE)** where data is streamed continuously over HTTP instead of waiting for full response.

---

### ❓ Difference between SSE and WebSocket?

**Answer:**

|SSE|WebSocket|
|---|---|
|One-way (server → client)|Two-way|
|HTTP-based|TCP-based|
|Simpler|More complex|

---

### ❓ How to implement streaming in WebFlux?

**Answer:**

@GetMapping(produces = MediaType.TEXT_EVENT_STREAM_VALUE)  
public Flux<String> stream() {  
    return Flux.interval(Duration.ofSeconds(1))  
               .map(i -> "data " + i);  
}