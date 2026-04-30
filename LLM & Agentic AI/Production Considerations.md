---
title: Production Considerations
tags:
  - agentic-ai
  - phase-7
  - production
  - operations
  - reliability
  - observability
date: 2025-07-15
---

# Production Considerations

## From Prototype to Production: What Changes?

Your agent works in your notebook. It answers questions, calls tools, gets the right answer. Ship it? Not so fast. The gap between "works on my machine" and "handles 1000 users reliably" is enormous.

In production, every failure costs money, trust, or both. Let's walk through what changes when you move from prototype to production.

---

## Cost Management

Agents are **expensive**. Unlike a single LLM call, agents loop — each request might trigger 3-10 LLM calls plus tool executions. That adds up fast.

### The Cost Problem

```
Single LLM call:    ~$0.01-0.03
Agent (5 iterations): ~$0.05-0.15
Agent (10 iterations): ~$0.10-0.30
× 10,000 daily users: $1,000 - $3,000/day
```

### Cost Control Strategies

- **Caching** — Cache tool results and even LLM responses for identical inputs. Redis works well here.
- **Model tiering** — Use cheaper models (GPT-4o-mini) for simple routing/classification steps, expensive models (GPT-4o, Claude) only for complex reasoning.
- **Early termination** — Set max iterations. If the agent hasn't solved it in 5 steps, it probably won't in 15.
- **Budget per request** — Hard limit on tokens per agent run. Reject requests that would exceed budget.
- **Prompt optimization** — Shorter system prompts = fewer tokens per call. Every token in the system prompt is repeated on every iteration.

```python
class CostGuard:
    def __init__(self, max_tokens: int = 10000, max_iterations: int = 8):
        self.max_tokens = max_tokens
        self.max_iterations = max_iterations
        self.total_tokens = 0
    
    def check(self, response_usage) -> bool:
        self.total_tokens += response_usage.total_tokens
        if self.total_tokens > self.max_tokens:
            raise BudgetExceededException(f"Token limit reached: {self.total_tokens}")
        return True
```

### Monitoring Spend

Set up alerts: if daily spend exceeds 120% of the 7-day average, something is wrong — maybe a prompt injection is causing infinite loops, or a bug is preventing termination.

---

## Latency

Agents are **slow**. Each iteration requires a round-trip to the LLM API (200-2000ms), plus tool execution time. Five iterations = 2-10 seconds minimum.

### Optimization Strategies

- **Parallel tool calls** — If the model requests multiple tools, execute them concurrently:
  ```python
  import asyncio
  
  async def execute_tools_parallel(tool_calls):
      tasks = [execute_tool(tc) for tc in tool_calls]
      return await asyncio.gather(*tasks)
  ```
- **Streaming** — Stream the final response token-by-token so users see progress immediately
- **Speculative execution** — Pre-fetch likely tool results before the model asks for them
- **Progress indicators** — Show "Searching...", "Analyzing...", "Thinking..." so users know something is happening

### Timeout Policies

```yaml
agent:
  max-duration: 30s           # total agent run time
  per-iteration-timeout: 10s  # single LLM call timeout
  tool-execution-timeout: 5s  # individual tool timeout
```

If a tool hangs or the LLM is slow, fail fast and return a partial answer rather than making the user wait indefinitely.

---

## Reliability

### LLM API Failures

APIs go down. OpenAI has outages. You need:

- **Fallback providers** — If OpenAI is down, route to Claude or a local model
- **Retry with backoff** — Transient errors (429, 503) should retry with exponential backoff
- **Circuit breakers** — If a provider fails 5 times in a row, stop trying for 30 seconds

```java
@Retryable(
    maxAttempts = 3,
    backoff = @Backoff(delay = 1000, multiplier = 2),
    include = {RateLimitException.class, ServiceUnavailableException.class}
)
public ChatResponse callLLM(Prompt prompt) {
    return chatModel.call(prompt);
}
```

### Tool Failures

Tools talk to external systems that fail. Handle gracefully:
- Return informative error messages (so the agent can adapt)
- Implement fallback tools (if primary search fails, try secondary)
- Degrade gracefully ("I couldn't access the weather service, but based on historical data...")

### Model Updates

LLM providers update models without warning. GPT-4o today might behave differently next week. Protect yourself:
- **Pin model versions** when available (e.g., `gpt-4o-2024-05-13`)
- **Regression test** — run your [[Evaluating Agent Performance|evaluation suite]] after any model change
- **Shadow testing** — run new model versions alongside the current one, compare outputs

---

## Observability

If you can't see what your agent is doing, you can't debug it, optimize it, or trust it.

### Trace Every Step

```python
import structlog

logger = structlog.get_logger()

def agent_loop_with_tracing(query, tools):
    trace_id = generate_trace_id()
    
    for i in range(max_iterations):
        logger.info("agent.iteration", 
            trace_id=trace_id, 
            iteration=i, 
            message_count=len(messages))
        
        response = call_llm(messages, tools)
        
        if response.tool_calls:
            for tc in response.tool_calls:
                logger.info("agent.tool_call",
                    trace_id=trace_id,
                    tool=tc.function.name,
                    args=tc.function.arguments)
                
                result = execute_tool(tc)
                
                logger.info("agent.tool_result",
                    trace_id=trace_id,
                    tool=tc.function.name,
                    result_length=len(result),
                    success=not result.startswith("Error"))
        else:
            logger.info("agent.complete",
                trace_id=trace_id,
                iterations=i+1,
                total_tokens=total_tokens)
            return response.content
```

### What to Track

| Metric | Why |
|--------|-----|
| Success rate | Are agents completing tasks? |
| Latency (p50, p95, p99) | How fast are responses? |
| Tokens per request | Cost tracking |
| Tool call frequency | Which tools are used most? |
| Error rate by tool | Which tools are unreliable? |
| Iterations per task | Agent efficiency |
| Cost per request | Budget monitoring |

### Observability Tools

- **LangSmith** — LangChain's tracing platform (great for LangGraph agents)
- **Weights & Biases** — experiment tracking, can log agent traces
- **OpenTelemetry** — standard tracing (add spans for each agent step)
- **Custom dashboards** — Grafana + Prometheus for your own metrics
- **Spring Boot Actuator** — health checks and metrics for Java agents

---

## Scaling

### Stateless Agent Design

Agents should be **stateless** — persist conversation state externally (Redis, database), not in process memory. This enables:
- Horizontal scaling (any instance can handle any request)
- Recovery from crashes (state survives restarts)
- Load balancing across instances

```java
@Service
public class StatelessAgentService {
    
    private final ChatClient chatClient;
    private final RedisTemplate<String, String> redis;
    
    public String chat(String sessionId, String message) {
        // Load state from Redis
        List<Message> history = loadHistory(sessionId);
        history.add(new UserMessage(message));
        
        // Run agent (stateless)
        String response = chatClient.prompt()
            .messages(history)
            .call().content();
        
        // Persist updated state
        history.add(new AssistantMessage(response));
        saveHistory(sessionId, history);
        
        return response;
    }
}
```

### Queue-Based Architecture

For long-running agents, use async execution:

```
User → API → Message Queue → Agent Worker → Result Store → User polls/webhook
```

This decouples the user request from agent execution. The user gets an immediate "processing" response, and the agent runs in the background.

### Rate Limit Management

LLM APIs have rate limits. When scaling to many concurrent users:
- **Pool API keys** — distribute requests across multiple keys
- **Backpressure** — if rate-limited, queue requests rather than failing
- **Priority queues** — paid users get higher priority
- **Token budgets** — allocate daily token budgets per user/team

---

## Security in Production

### API Key Management
- Never hardcode keys. Use a secret manager (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault)
- Rotate keys on a schedule
- Use different keys for dev/staging/prod

### Input Validation
Validate at every layer — [[Prompt Injection and Security|prompt injection]] is a real threat:
- Sanitize user input before it reaches the LLM
- Validate tool arguments before execution
- Check tool outputs before returning to users

### Audit Trail
Log every agent action with enough detail to reconstruct what happened:
- Who triggered the agent?
- What tools were called with what arguments?
- What data was accessed?
- What was the final output?

This is critical for compliance, debugging, and [[Guardrails and Safety Layers|safety monitoring]].

### Data Isolation (Multi-Tenant)
If your agent serves multiple customers:
- Each customer's data must be isolated
- Tools should scope queries to the current tenant
- Never let one customer's data leak into another's context

---

## Reference Architecture

A production-ready agent system in the Spring ecosystem:

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│ API Gateway │────▶│ Spring Boot  │────▶│ LLM Provider│
│ (rate limit)│     │ Agent Service│     │ (OpenAI)    │
└─────────────┘     └──────┬───────┘     └─────────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  Redis   │ │ Postgres │ │ Vector DB│
        │ (memory) │ │ (state)  │ │  (RAG)   │
        └──────────┘ └──────────┘ └──────────┘
              │
              ▼
        ┌──────────────┐
        │ Observability│
        │ (Grafana/OTel)│
        └──────────────┘
```

Components:
- **API Gateway** — authentication, rate limiting, request routing
- **Agent Service** — [[Building an Agent with Spring AI|Spring AI]] application with tool definitions
- **Redis** — conversation memory, response caching
- **Postgres** — persistent state, audit logs, user data
- **Vector DB** — [[RAG - Retrieval Augmented Generation|RAG]] for knowledge retrieval
- **Observability** — metrics, traces, alerts

---

## Key Takeaways

1. **Cost adds up fast** — agents make multiple LLM calls per request; use caching, model tiering, and budget limits to control spend
2. **Latency is inherent** — agent loops are sequential; mitigate with parallel tool calls, streaming, and progress indicators
3. **Reliability requires redundancy** — fallback providers, retries with backoff, and circuit breakers for when APIs fail
4. **Observability is non-negotiable** — trace every agent step, track success rates, latency percentiles, and cost per request
5. **Design for statelessness** — persist state externally (Redis/DB) so agents can scale horizontally and survive restarts
6. **Security at every layer** — vault for keys, input validation for [[Prompt Injection and Security|injection prevention]], audit trails for compliance
7. Start with the simplest production setup that works, then add complexity (caching, queues, multi-provider) as scale demands it
