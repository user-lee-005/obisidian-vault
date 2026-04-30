---
title: Function Calling in LLMs
tags:
  - llm
  - function-calling
  - tools
  - phase-5
date: 2024-01-20
---

# Function Calling in LLMs

## The Limitation: LLMs Can Only Generate Text

Here's the fundamental truth — LLMs are **text prediction machines**. They can't check the weather, query a database, send an email, or book a flight. They can only *write about* doing those things. So how do we bridge the gap between "generating text about actions" and "actually performing actions"?

The answer is **function calling** — and it's one of the most powerful capabilities in modern AI systems.

---

## The Solution: Function Calling

Function calling is an elegant contract between the LLM and your application:

- The **LLM decides WHAT to call** (which function, with what arguments)
- **YOUR code DOES the calling** (executes the function, handles errors, returns results)

The model never directly touches your systems. It just outputs a structured request saying "I'd like to call this function with these parameters." Your application is the gatekeeper.

This is critical for safety — imagine letting an LLM directly execute database queries or API calls without any application-level control. Function calling keeps humans (and their code) in the loop.

---

## How It Works: Step by Step

Let's walk through the full lifecycle:

1. **You define available functions** — name, description, parameters as JSON Schema
2. **User sends a message** — "What's the weather in Chennai?"
3. **LLM decides whether to call a function** — it reads the descriptions and picks the right one
4. **LLM outputs a structured function call** — `{"name": "get_weather", "arguments": {"city": "Chennai"}}`
5. **YOUR application executes the function** — calls the actual weather API
6. **You feed the result back to the LLM** — `{"temperature": 32, "condition": "sunny"}`
7. **LLM generates a final response** — "It's currently 32°C and sunny in Chennai!"

The model is essentially a **routing layer** — it figures out intent, extracts parameters, and formats the request. Your code does the heavy lifting.

---

## OpenAI Function Calling Format

Here's how you define tools for the OpenAI API:

```json
{
  "model": "gpt-4",
  "messages": [{"role": "user", "content": "What's the weather in Chennai?"}],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get the current weather for a given city",
        "parameters": {
          "type": "object",
          "properties": {
            "city": {
              "type": "string",
              "description": "The city name, e.g. Chennai, Bangalore"
            },
            "units": {
              "type": "string",
              "enum": ["celsius", "fahrenheit"],
              "description": "Temperature units"
            }
          },
          "required": ["city"]
        }
      }
    }
  ]
}
```

When the model decides to call this function, the response looks like:

```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "tool_calls": [{
        "id": "call_abc123",
        "type": "function",
        "function": {
          "name": "get_weather",
          "arguments": "{\"city\": \"Chennai\", \"units\": \"celsius\"}"
        }
      }]
    }
  }]
}
```

Notice: `arguments` is a **JSON string**, not a parsed object. You'll need to deserialize it.

---

## Anthropic Tool Use Format (Comparison)

Anthropic's Claude uses a slightly different structure:

```json
{
  "model": "claude-sonnet-4-20250514",
  "tools": [
    {
      "name": "get_weather",
      "description": "Get current weather for a city",
      "input_schema": {
        "type": "object",
        "properties": {
          "city": {"type": "string", "description": "City name"}
        },
        "required": ["city"]
      }
    }
  ],
  "messages": [{"role": "user", "content": "Weather in Chennai?"}]
}
```

The response includes a `tool_use` content block instead of `tool_calls`. Same concept, slightly different structure. Both follow the pattern: **define → call → execute → return**.

---

## Parallel Function Calling

Modern models can request **multiple function calls in a single response**. Imagine the user asks: "Compare the weather in Chennai and Bangalore."

The model returns:

```json
{
  "tool_calls": [
    {"id": "call_1", "function": {"name": "get_weather", "arguments": "{\"city\": \"Chennai\"}"}},
    {"id": "call_2", "function": {"name": "get_weather", "arguments": "{\"city\": \"Bangalore\"}"}}
  ]
}
```

Your application can execute these **in parallel**, feed both results back, and the model composes a unified answer. This dramatically reduces round-trips for multi-step queries.

---

## Why the Model Doesn't Execute Functions Itself

Three reasons:

- **Safety** — You control what actually happens. The model can suggest calling `delete_all_users()`, but your code can reject it.
- **Control** — You apply business logic, rate limits, authentication, and authorization before execution.
- **Reliability** — Models hallucinate. They might generate invalid arguments. Your code validates before executing.

Think of function calling as the model filling out a form, and your application deciding whether to process that form.

---

## Key Takeaways

1. LLMs can't execute actions — they can only **describe** what action to take via structured output
2. Function calling is a contract: the model decides **what**, your code does **how**
3. Functions are defined via JSON Schema — clear descriptions help the model choose correctly
4. The lifecycle is: define → user message → model decides → structured call → execute → return result → final response
5. Parallel calling enables multiple tool invocations in a single turn
6. The application is always the gatekeeper — never let models directly execute untrusted operations
7. This pattern is the foundation for [[Tool Use Patterns]] and [[Building Tool-Using Chatbots]]
