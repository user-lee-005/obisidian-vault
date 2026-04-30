---
tags:
  - llm
  - api
  - openai
  - phase-4
---

# OpenAI API Deep Dive

Now that we understand how LLMs work under the hood ([[Transformer Architecture]], [[Attention Mechanism]]), let's actually *use* one. The OpenAI API is the most popular way to integrate LLM capabilities into your applications — and understanding it well translates to working with any LLM provider.

---

## Getting Started

**API Keys** — You'll need an account at `platform.openai.com`. Generate a secret key and treat it like a database password. Never commit it to source control.

**Pricing Model** — OpenAI charges **per token** (roughly ¾ of a word). You pay for both input tokens (your prompt) and output tokens (the response). GPT-4o is cheaper than GPT-4, and GPT-3.5-turbo is cheaper still. Always check the pricing page — it changes frequently.

**Rate Limits** — You'll hit limits on:
- Requests per minute (RPM)
- Tokens per minute (TPM)
- Requests per day (RPD)

New accounts start with lower limits. As you spend more, limits increase automatically.

---

## Chat Completions API

This is the API you'll use 90% of the time. It takes a **messages array** — a conversation history — and returns the next assistant message.

### The Messages Array

Every message has a `role` and `content`:

- **system** — Sets behavior, personality, constraints (invisible to the user)
- **user** — The human's input
- **assistant** — Previous model responses (for multi-turn context)

### Request Structure

```json
{
  "model": "gpt-4o",
  "messages": [
    {"role": "system", "content": "You are a helpful logistics assistant."},
    {"role": "user", "content": "What's the difference between FCL and LCL shipping?"}
  ],
  "temperature": 0.7,
  "max_tokens": 500
}
```

### Response Structure

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "FCL (Full Container Load) means..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 32,
    "completion_tokens": 145,
    "total_tokens": 177
  }
}
```

### Key Parameters

| Parameter | What it does | Typical values |
|-----------|-------------|----------------|
| `model` | Which model to use | `gpt-4o`, `gpt-4o-mini`, `gpt-3.5-turbo` |
| `temperature` | Randomness (0 = deterministic, 2 = creative) | 0.0–1.0 for most uses |
| `max_tokens` | Cap on response length | 500–4000 |
| `top_p` | Nucleus sampling (alternative to temperature) | 0.9–1.0 |

> [!tip] Temperature vs top_p
> Use one or the other, not both. Temperature is more intuitive — start there.

---

## Streaming

Without streaming, the user stares at a blank screen until the *entire* response is generated. With streaming, tokens arrive as they're produced — just like ChatGPT's typing effect.

**How it works:** Server-Sent Events (SSE). The API sends a stream of `data:` chunks, each containing a token delta.

```python
import openai

client = openai.OpenAI()

stream = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Explain containerization"}],
    stream=True
)

for chunk in stream:
    delta = chunk.choices[0].delta.content
    if delta:
        print(delta, end="", flush=True)
```

In a Spring Boot app, you'd use `WebFlux` and return a `Flux<String>` to stream tokens to the frontend. See [[Spring AI Framework]] for the Java approach.

---

## Function Calling / Tool Use

This is where things get powerful. Instead of just generating text, the model can decide to **call functions** you define — accessing databases, APIs, calculators, anything.

### Step 1: Define Your Functions

```json
{
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_shipment_status",
        "description": "Get the current status of a shipment by tracking number",
        "parameters": {
          "type": "object",
          "properties": {
            "tracking_number": {
              "type": "string",
              "description": "The shipment tracking number"
            }
          },
          "required": ["tracking_number"]
        }
      }
    }
  ]
}
```

### Step 2: Model Decides to Call

When the user asks "Where is my shipment TRK-12345?", the model responds with a `tool_calls` array instead of text content.

### Step 3: Execute and Feed Back

You execute the function locally, then send the result back as a message with `role: "tool"`. The model then generates a natural language response incorporating that data.

> [!note] The model never executes code
> It only *requests* function calls. You decide whether to actually execute them. This is your security boundary.

---

## Structured Outputs

Need the model to return valid JSON? Use `response_format`:

```json
{
  "model": "gpt-4o",
  "response_format": { "type": "json_object" },
  "messages": [
    {"role": "system", "content": "Return responses as JSON with fields: summary, confidence, next_steps"},
    {"role": "user", "content": "Analyze this shipping delay..."}
  ]
}
```

The model will *always* return valid JSON. Combine with [[Prompt Engineering Techniques]] for reliable structured outputs.

---

## Embeddings API

Embeddings convert text into dense vectors — numerical representations that capture semantic meaning. Two similar sentences will have similar vectors.

```python
response = client.embeddings.create(
    model="text-embedding-3-small",
    input="container shipping from Shanghai to Rotterdam"
)

vector = response.data[0].embedding  # List of 1536 floats
```

Use cases: semantic search, clustering, recommendation. See [[Vector Databases]] and [[RAG - Retrieval Augmented Generation]] for putting these to work.

---

## Best Practices

- **Error handling** — The API can fail. Wrap calls in retry logic.
- **Exponential backoff** — On rate limit errors (429), wait 1s, 2s, 4s, 8s...
- **Token counting** — Use `tiktoken` library to count tokens *before* sending. Avoid surprises.
- **Timeouts** — Set reasonable timeouts (30–60s). LLMs can be slow on complex prompts.

---

## Cost Optimization

- **Model selection** — Use `gpt-4o-mini` for simple tasks, `gpt-4o` for complex reasoning
- **Prompt caching** — OpenAI caches long prefixes automatically (system prompts, few-shot examples)
- **Batching** — The Batch API offers 50% discount for non-time-sensitive workloads
- **Shorter prompts** — Every token costs money. Be concise in your [[Prompt Engineering Techniques]]
- **Max tokens** — Set it appropriately to avoid paying for overly long responses

---

## Key Takeaways

1. The Chat Completions API uses a **messages array** with system/user/assistant roles — this is the universal interface for modern LLMs
2. **Streaming** is essential for UX — nobody wants to wait 10 seconds staring at a spinner
3. **Function calling** lets models interact with your systems — the model requests, you execute
4. **Structured outputs** (JSON mode) eliminate fragile regex parsing of LLM responses
5. **Embeddings** are the bridge between text and [[Vector Databases]] — fundamental for [[RAG - Retrieval Augmented Generation|RAG]]
6. Always implement **retry logic with exponential backoff** — APIs are unreliable
7. Cost optimization starts with choosing the **right model** for each task — not everything needs GPT-4
