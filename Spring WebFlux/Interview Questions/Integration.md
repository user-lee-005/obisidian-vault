### ❓ Can you use JPA with WebFlux?

**Answer:**  
Yes, but it’s blocking → defeats WebFlux benefits.  
Use reactive DB drivers like R2DBC instead.

---

### ❓ What is R2DBC?

**Answer:**  
A non-blocking alternative to JDBC that supports reactive database access using `Mono` and `Flux`.

---

### ❓ How do you call external APIs?

**Answer:**  
Use `WebClient` (reactive, non-blocking) instead of RestTemplate.

---

### ❓ WebClient vs RestTemplate?

**Answer:**

|WebClient|RestTemplate|
|---|---|
|Non-blocking|Blocking|
|Reactive|Traditional|
|Streaming support|No|