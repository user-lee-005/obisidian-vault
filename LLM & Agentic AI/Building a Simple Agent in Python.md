---
title: Building a Simple Agent in Python
tags:
  - agentic-ai
  - phase-7
  - python
  - agents
  - implementation
date: 2025-07-15
---

# Building a Simple Agent in Python

## Goal: Build a ReAct Agent from Scratch

Let's build an agent from the ground up — no LangChain, no frameworks, just Python and the OpenAI API. The goal is to deeply understand what's actually happening inside agent frameworks before we ever use one. Once you see how simple the core loop is, frameworks will feel like helpful shortcuts rather than black magic.

We're building a [[ReAct Pattern]] agent: the model **thinks**, **acts** (calls a tool), and **observes** (reads the result), looping until it has a final answer.

---

## Components We Need

Every agent needs exactly four things:

- **LLM** — the brain (OpenAI API in our case)
- **Tools** — functions the agent can call to interact with the world
- **Agent Loop** — the think → act → observe cycle
- **Conversation Memory** — the full message history so the LLM has context

That's it. An agent is just a loop around an LLM that can call functions. Let's build each piece.

---

## Step 1: Define Tools

Tools are just Python functions with metadata that tells the LLM what they do. We use OpenAI's function calling format:

```python
import json
import math
import requests

# The actual tool implementations
def search(query: str) -> str:
    """Search the web using DuckDuckGo Instant Answer API."""
    resp = requests.get("https://api.duckduckgo.com/", params={"q": query, "format": "json"})
    data = resp.json()
    return data.get("AbstractText") or f"No results found for: {query}"

def calculate(expression: str) -> str:
    """Safely evaluate a math expression."""
    try:
        allowed_names = {"abs": abs, "round": round, "min": min, "max": max, "pow": pow}
        result = eval(expression, {"__builtins__": {}}, allowed_names)
        return str(result)
    except Exception as e:
        return f"Error: {e}"

# Tool definitions for the OpenAI API
tools = [
    {
        "type": "function",
        "function": {
            "name": "search",
            "description": "Search the web for current information. Use for facts, people, events, etc.",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "The search query"}
                },
                "required": ["query"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "calculate",
            "description": "Evaluate a mathematical expression. Use for any computation.",
            "parameters": {
                "type": "object",
                "properties": {
                    "expression": {"type": "string", "description": "Math expression to evaluate, e.g. '2 ** 10'"}
                },
                "required": ["expression"]
            }
        }
    }
]

# Map names to functions for execution
tool_functions = {
    "search": search,
    "calculate": calculate,
}
```

Notice: the tool definitions are just JSON schemas. The LLM never sees our Python code — it only sees the name, description, and parameter schema.

---

## Step 2: The Agent Loop

This is the heart of the agent. It's surprisingly simple:

```python
from openai import OpenAI

client = OpenAI()  # uses OPENAI_API_KEY env var

SYSTEM_PROMPT = """You are a helpful assistant with access to tools.
Think step by step. Use tools when you need information or computation.
Always verify your answers when possible."""

def agent_loop(user_query: str, tools: list, max_iterations: int = 10) -> str:
    """Run the agent loop: think → act → observe → repeat."""
    
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": user_query}
    ]
    
    for i in range(max_iterations):
        print(f"\n--- Iteration {i + 1} ---")
        
        # Think: ask the LLM what to do next
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=tools,
            tool_choice="auto"  # let the model decide
        )
        
        message = response.choices[0].message
        messages.append(message)  # add assistant response to history
        
        # Check: does the model want to call tools?
        if message.tool_calls:
            # Act: execute each tool call
            for tool_call in message.tool_calls:
                result = execute_tool(tool_call)
                print(f"  Tool: {tool_call.function.name}({tool_call.function.arguments})")
                print(f"  Result: {result[:100]}...")
                
                # Observe: add the result back to messages
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": result
                })
        else:
            # No tool calls — the agent is done, return final answer
            print(f"\n✓ Final answer reached in {i + 1} iterations")
            return message.content
    
    return "Max iterations reached. Partial answer: " + (message.content or "No response")
```

The pattern is beautifully simple:
1. Send messages to LLM
2. If the LLM wants to call tools → execute them, add results to messages, loop
3. If the LLM doesn't call tools → it's done, return the answer

---

## Step 3: Tool Execution with Error Handling

Tools fail. APIs go down, inputs are malformed, things break. A robust agent handles this gracefully:

```python
def execute_tool(tool_call) -> str:
    """Execute a tool call with error handling."""
    name = tool_call.function.name
    
    # Parse arguments
    try:
        args = json.loads(tool_call.function.arguments)
    except json.JSONDecodeError as e:
        return f"Error parsing arguments: {e}"
    
    # Look up the function
    func = tool_functions.get(name)
    if not func:
        return f"Unknown tool: {name}. Available: {list(tool_functions.keys())}"
    
    # Execute with timeout and error handling
    try:
        result = func(**args)
        return str(result)
    except TypeError as e:
        return f"Invalid arguments for {name}: {e}"
    except Exception as e:
        return f"Tool execution error: {type(e).__name__}: {e}"
```

The key insight: we return error messages **as tool results** rather than crashing. This lets the LLM read the error and try again — agents are surprisingly good at self-correcting when you give them useful error messages.

---

## Step 4: Adding Memory

Our agent already has "within-conversation" memory — the `messages` list accumulates the full history. But what about memory **across conversations**?

```python
class AgentMemory:
    """Simple conversation memory that persists across calls."""
    
    def __init__(self, max_messages: int = 50):
        self.messages = []
        self.max_messages = max_messages
    
    def add(self, message):
        self.messages.append(message)
        # Trim old messages (keep system prompt + recent history)
        if len(self.messages) > self.max_messages:
            system = self.messages[0]
            self.messages = [system] + self.messages[-self.max_messages:]
    
    def get_messages(self) -> list:
        return self.messages.copy()
    
    def clear(self):
        self.messages = []

def agent_with_memory(user_query: str, memory: AgentMemory, tools: list) -> str:
    """Agent that maintains conversation history across calls."""
    
    if not memory.get_messages():
        memory.add({"role": "system", "content": SYSTEM_PROMPT})
    
    memory.add({"role": "user", "content": user_query})
    
    for i in range(10):
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=memory.get_messages(),
            tools=tools,
            tool_choice="auto"
        )
        
        message = response.choices[0].message
        memory.add(message)
        
        if message.tool_calls:
            for tool_call in message.tool_calls:
                result = execute_tool(tool_call)
                memory.add({"role": "tool", "tool_call_id": tool_call.id, "content": result})
        else:
            return message.content
    
    return "Max iterations reached."
```

Now the agent remembers previous conversations and can reference earlier context.

---

## Step 5: Running It

```python
if __name__ == "__main__":
    # Single query
    answer = agent_loop("What is the population of Tokyo and what is it squared?", tools)
    print(f"\nAnswer: {answer}")
    
    # Multi-turn with memory
    memory = AgentMemory()
    print(agent_with_memory("What's the capital of France?", memory, tools))
    print(agent_with_memory("What's its population?", memory, tools))  # "its" = France's capital
    print(agent_with_memory("Multiply that by 3", memory, tools))      # "that" = population
```

Example output:
```
--- Iteration 1 ---
  Tool: search({"query": "population of Tokyo"})
  Result: Tokyo has a population of approximately 13.96 million...
--- Iteration 2 ---
  Tool: calculate({"expression": "13960000 ** 2"})
  Result: 194880160000000000...
--- Iteration 3 ---
✓ Final answer reached in 3 iterations

Answer: The population of Tokyo is approximately 13.96 million. Squared, that's about 1.95 × 10^14.
```

---

## What We Learned

The agent is **just a loop** around three things:
1. An LLM that decides what to do
2. Tools that interact with the world
3. Memory that accumulates context

Every agent framework — [[LangGraph and Agent Frameworks|LangGraph]], CrewAI, Semantic Kernel — is fundamentally doing this same loop with extra features on top.

---

## Limitations of This Simple Agent

- **No planning** — it acts greedily, one step at a time
- **No reflection** — it doesn't evaluate whether its approach is working
- **No backtracking** — if it goes down a wrong path, it can't undo
- **No parallelism** — tool calls are sequential
- **Token limit** — long conversations will exceed context window
- **No [[Guardrails and Safety Layers|safety guardrails]]** — it will try anything the tools allow

These limitations are exactly what frameworks and [[Agent Architecture Patterns|architecture patterns]] solve. But understanding this simple version first makes those solutions make sense.

---

## Key Takeaways

1. An agent is fundamentally a **loop**: send messages to LLM → execute tool calls → append results → repeat until done
2. You need only four components: **LLM, tools, agent loop, and memory** — everything else is optimization
3. Tool definitions are **JSON schemas** — the LLM never sees your implementation code, only descriptions
4. Error handling should return errors **as tool results** so the LLM can self-correct
5. Memory is just the **message history** — maintaining it across conversations gives the agent context
6. This simple agent has real limitations (no planning, no reflection) that motivate [[Agent Architecture Patterns|more advanced patterns]]
7. Every agent framework is doing this same loop — understanding the internals demystifies the magic
