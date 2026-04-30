---
title: Error Handling and Retries
tags:
  - llm
  - error-handling
  - resilience
  - tools
  - phase-5
date: 2024-01-20
---

# Error Handling and Retries

## Why Tools Fail

When your [[Building Tool-Using Chatbots|chatbot]] calls external tools, things will go wrong. Not *might* — **will**. Networks drop, APIs rate-limit you, inputs are malformed, services go down. The question isn't whether tools fail, but how gracefully your system handles failure.

Let's build resilient tool-using systems.

---

## Error Categories

Not all failures are equal. How you respond depends on what went wrong:

### Transient Errors → Retry

- **Network timeouts** — the service is there, just slow
- **Rate limits** (HTTP 429) — too many requests, back off
- **Server errors** (HTTP 5xx) — temporary infrastructure issues
- **Connection resets** — network blips

These are temporary. Wait and try again.

### Permanent Errors → Inform the Model

- **Invalid arguments** (HTTP 400) — the model sent bad parameters
- **Authentication failures** (HTTP 401/403) — credentials are wrong or expired
- **Not found** (HTTP 404) — the resource doesn't exist
- **Validation failures** — business logic rejected the request

Don't retry these — the same input will produce the same failure. Feed the error back to the model.

### Partial Failures → Let the Model Decide

- Tool returns **incomplete data** — some fields are missing
- **Partial results** — search returned 3 of expected 10 results
- **Stale data** — cache returned old information

Give the model what you have, and let it decide whether to proceed or ask for clarification.

---

## Retry Strategies

### Exponential Backoff with Jitter

The gold standard for transient errors — wait progressively longer between retries, with randomness to prevent thundering herd:

```python
import time
import random

def retry_with_backoff(func, max_retries=3, base_delay=1.0):
    for attempt in range(max_retries):
        try:
            return func()
        except TransientError as e:
            if attempt == max_retries - 1:
                raise  # give up after max retries
            
            delay = base_delay * (2 ** attempt)  # 1s, 2s, 4s...
            jitter = random.uniform(0, delay * 0.1)  # add randomness
            time.sleep(delay + jitter)
            
    raise MaxRetriesExceeded()
```

### Max Retry Limits

Always cap retries. An infinite retry loop will:
- Burn API credits
- Block the user indefinitely
- Compound issues on an already stressed system

Reasonable defaults: **3 retries for fast APIs, 2 for slow ones.**

### Circuit Breaker Pattern

When a service is consistently failing, stop hitting it entirely:

```java
// Using Resilience4j — see Microservices Patterns
@CircuitBreaker(name = "shipmentService", fallbackMethod = "fallback")
@Retry(name = "shipmentService")
public ShipmentInfo getShipment(String trackingNumber) {
    return shipmentApiClient.getShipment(trackingNumber);
}

public ShipmentInfo fallback(String trackingNumber, Exception ex) {
    return new ShipmentInfo(trackingNumber, "UNKNOWN", 
        "Service temporarily unavailable. Please try again later.");
}
```

States: **Closed** (normal) → **Open** (failing, reject all calls) → **Half-Open** (test with a few calls).

---

## Feeding Errors Back to the Model

This is the most important pattern. When a tool fails, **tell the model what happened**:

### DON'T: Silently Fail

```python
# BAD — model has no idea what happened
try:
    result = call_api(args)
except Exception:
    result = ""  # empty response, model is confused
```

### DO: Return Structured Error Information

```python
# GOOD — model can adapt its behavior
try:
    result = call_api(args)
except RateLimitError:
    result = {
        "error": "rate_limit_exceeded",
        "message": "The shipment API is rate-limited. Try again in 30 seconds.",
        "retry_after": 30
    }
except NotFoundError:
    result = {
        "error": "not_found",
        "message": f"No shipment found with tracking number {tracking_number}. Please verify the number."
    }
except ValidationError as e:
    result = {
        "error": "invalid_input",
        "message": f"Invalid arguments: {str(e)}. Expected format: TRK-YYYY-NNNNN"
    }
```

The model receives this error and can:
- **Try a different approach** — "Let me search by order number instead"
- **Ask for clarification** — "Could you double-check that tracking number?"
- **Inform the user honestly** — "The tracking system is currently unavailable"

---

## Timeout Handling

Set reasonable timeouts and fail fast. Don't let a stuck API hang your chatbot:

```python
import httpx

async def call_tool_with_timeout(tool_name: str, args: dict):
    try:
        async with httpx.AsyncClient(timeout=10.0) as client:
            response = await client.post(f"/tools/{tool_name}", json=args)
            return response.json()
    except httpx.TimeoutException:
        return {
            "error": "timeout",
            "message": f"Tool '{tool_name}' did not respond within 10 seconds."
        }
```

```java
// Spring WebClient with timeout
public Mono<ToolResult> callTool(String name, Map<String, Object> args) {
    return webClient.post()
        .uri("/tools/{name}", name)
        .bodyValue(args)
        .retrieve()
        .bodyToMono(ToolResult.class)
        .timeout(Duration.ofSeconds(10))
        .onErrorReturn(TimeoutException.class,
            ToolResult.error("Tool timed out after 10 seconds"));
}
```

---

## Graceful Degradation

When a tool is completely down, what's the fallback?

### Answer from Knowledge with a Disclaimer

```
"I wasn't able to reach the live tracking system right now, but based on typical 
transit times for Chennai→London shipments, your package likely arrives in 5-7 
business days. Would you like me to try again later?"
```

### Suggest Alternative Actions

```
"The booking system is currently unavailable. Here are your options:
1. I can save these details and you can book when it's back up
2. I can email the support team on your behalf
3. Would you like the booking reference to do it manually?"
```

### Queue for Later Execution

For non-urgent actions, acknowledge and defer:

```python
result = {
    "status": "queued",
    "message": "The email service is temporarily down. Your message has been queued and will be sent within the hour.",
    "queue_id": "Q-2024-789"
}
```

---

## Observability

You can't fix what you can't see. Log everything about tool calls:

```python
import structlog

logger = structlog.get_logger()

async def execute_tool(tool_call):
    start = time.time()
    
    try:
        result = await dispatch_tool(tool_call)
        duration = time.time() - start
        
        logger.info("tool_call_success",
            tool=tool_call.function.name,
            duration_ms=round(duration * 1000),
            args=tool_call.function.arguments
        )
        return result
        
    except Exception as e:
        duration = time.time() - start
        
        logger.error("tool_call_failure",
            tool=tool_call.function.name,
            duration_ms=round(duration * 1000),
            error=str(e),
            error_type=type(e).__name__
        )
        return {"error": str(e)}
```

Track these metrics:
- **Success/failure rate** per tool
- **Latency** (p50, p95, p99) per tool
- **Error types** distribution
- **Retry frequency** — are you hitting limits?

---

## Spring Patterns: @Retryable + Resilience4j

```java
@Service
public class ResilientToolExecutor {

    @Retryable(
        retryFor = {TimeoutException.class, ServiceUnavailableException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2.0)
    )
    public ToolResult executeTool(String name, Map<String, Object> args) {
        return toolRegistry.get(name).execute(args);
    }

    @Recover
    public ToolResult recover(Exception ex, String name, Map<String, Object> args) {
        return ToolResult.error(
            "Tool '%s' failed after 3 attempts: %s".formatted(name, ex.getMessage())
        );
    }
}
```

Combine with Resilience4j for circuit breaking:

```java
@CircuitBreaker(name = "externalApi", fallbackMethod = "apiFallback")
@RateLimiter(name = "externalApi")
@Retry(name = "externalApi")
public ApiResponse callExternalApi(ApiRequest request) {
    return restTemplate.postForObject(apiUrl, request, ApiResponse.class);
}
```

---

## Python Patterns: tenacity Library

```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    retry=retry_if_exception_type((TimeoutError, ConnectionError))
)
def call_external_api(endpoint: str, params: dict) -> dict:
    response = httpx.post(endpoint, json=params, timeout=10)
    response.raise_for_status()
    return response.json()

# Usage in tool execution
def execute_tool_safely(tool_call) -> dict:
    try:
        return call_external_api(tool_call.endpoint, tool_call.args)
    except Exception as e:
        # Feed error back to model
        return {
            "error": type(e).__name__,
            "message": str(e),
            "suggestion": "The service may be down. Try again or use an alternative."
        }
```

---

## Key Takeaways

1. Tools **will** fail — design for failure from day one, not as an afterthought
2. Classify errors: **transient** (retry), **permanent** (inform model), **partial** (let model decide)
3. Use **exponential backoff with jitter** for retries — always cap max attempts
4. **Circuit breakers** prevent cascading failures when a service is consistently down
5. **Always feed errors back to the model** as structured messages — never silently fail or return empty
6. The model can adapt to errors: try alternatives, ask for clarification, or honestly inform the user
7. Set **aggressive timeouts** — a 10s timeout is usually better than waiting 60s for a hung service
8. **Log everything**: tool name, duration, success/failure, error types — you need this for debugging and optimization
9. In Spring: combine `@Retryable` + Resilience4j circuit breakers for production-grade resilience
10. In Python: use `tenacity` for declarative retry policies with clean syntax
