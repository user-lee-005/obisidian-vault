### ❓ How do you handle errors?

**Answer:**

- `.onErrorResume()` → fallback
- `.onErrorReturn()` → default value
- Global exception handler

---

### ❓ Difference between map and flatMap?

**Answer:**

- `map` → synchronous transformation
- `flatMap` → asynchronous, returns Mono/Flux

Example:

Mono<User> → Mono<String>  
map(User::getName)