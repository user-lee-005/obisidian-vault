# Memory Systems for Agents

An LLM has no memory by default. Every call is stateless. But agents need to remember — what the user asked, what they've tried, what worked, what failed. Let's explore how we give agents memory.

---

## Why Agents Need Memory

The **context window** is finite. Even with 128k tokens, a complex agent task can easily overflow it. And more fundamentally:

- Tasks span many interactions (multi-session workflows)
- Agents need to learn from past experience
- Users expect continuity ("Remember when I asked you to...")
- Avoiding repeated mistakes requires knowing what failed before

Think of it like the difference between a stateless HTTP request and a service with a database. Without memory, every agent invocation starts from zero.

---

## Types of Memory

### Short-Term Memory (Working Memory)

This is the **conversation history** — the context window itself. It's what the LLM can "see" right now.

**Limitations:**
- Fixed token budget (even 128k fills up fast with tool outputs)
- Everything in context costs money per call

**Management strategies:**
- **Sliding window** — Keep only the last N messages, drop the oldest
- **Summarization** — Periodically compress old messages into a summary
- **Importance-based pruning** — Keep messages that seem relevant, drop routine ones
- **Tool output truncation** — Store only key results, not full API responses

### Long-Term Memory

Persistent storage that survives across sessions. This is where [[Vector Databases]] and [[RAG - Retrieval Augmented Generation]] come in for agents.

**How it works:**
- Agent experiences get embedded and stored in a vector database
- Before each reasoning step, relevant memories are retrieved
- The agent sees: current context + relevant past experiences

**Key decisions:**
- **When to write** — After completing a task? After learning something new? Continuously?
- **When to retrieve** — Every turn? Only when the agent asks? Based on similarity threshold?
- **What to store** — Raw messages? Summaries? Key facts? Decisions and their outcomes?

### Episodic Memory

Records of **specific past experiences** — what happened, in what context, and how it turned out.

- "Last time the user asked about deployment, they meant the staging environment"
- "I tried using the REST API for bulk imports and it timed out — the batch endpoint works better"
- "This codebase uses Gradle, not Maven — don't suggest pom.xml"

Episodic memory helps agents **avoid repeating mistakes** and **adapt to specific users/environments**. It's like a developer's personal notes about a project's quirks.

### Procedural Memory

**Learned skills and procedures** — how to do things, not what happened.

- Tool documentation and API schemas
- Step-by-step procedures for common tasks
- Code patterns specific to a codebase

This can be:
- Retrieved via RAG (documentation lookup)
- Fine-tuned into the model (baked-in skills)
- Stored as reusable prompts/templates

---

## Memory Architecture in Practice

A practical agent memory system typically combines several components:

- **Scratchpad** — Temporary notes during a task. "I found 3 files that match, need to check each one." Lives only for the current task.
- **Conversation buffer + summary** — Recent messages in full, older ones summarized. Balances detail and token budget.
- **Semantic search over past interactions** — Vector store of past conversations, retrieved when relevant. "Have I helped this user with authentication before?"
- **Structured storage** — Entities, relationships, facts stored in a knowledge graph or database. "User prefers Python. Project uses PostgreSQL. Deploy target is AWS."

```
┌─────────────────────────────────────────┐
│           Agent Reasoning               │
├─────────────────────────────────────────┤
│  Context Window (Working Memory)        │
│  ┌──────────┐ ┌──────────┐ ┌────────┐  │
│  │Scratchpad│ │  Recent  │ │Retrieved│  │
│  │  Notes   │ │ Messages │ │Memories │  │
│  └──────────┘ └──────────┘ └────────┘  │
├─────────────────────────────────────────┤
│  Long-Term Storage                      │
│  ┌───────────┐ ┌─────────┐ ┌────────┐  │
│  │  Vector   │ │  Facts  │ │Episodes│  │
│  │  Store    │ │  (KG)   │ │  Log   │  │
│  └───────────┘ └─────────┘ └────────┘  │
└─────────────────────────────────────────┘
```

---

## Challenges

- **What to remember** — Storing everything is expensive and noisy. Storing too little loses important context.
- **When to forget** — Outdated memories can mislead. How do you expire stale facts?
- **Retrieval quality** — Semantic search isn't perfect. Relevant memories might not be retrieved; irrelevant ones might be.
- **Contradictions** — Old memory says "use API v1"; new info says "v1 is deprecated." How to resolve?
- **Privacy** — Long-term memory of user data has compliance implications (GDPR, data retention).

---

## Practical Implementation

A simple but effective setup:

1. **Vector store** (e.g., Pinecone, Weaviate, pgvector) for semantic retrieval
2. **Metadata** on each memory: timestamp, source, confidence, tags
3. **Recency scoring** — Blend semantic similarity with how recent the memory is
4. **Explicit memory operations** — Let the agent explicitly "save" and "recall" memories as tool calls

This gives you a working memory system without over-engineering. Start simple, add sophistication (knowledge graphs, decay functions, contradiction resolution) only when you hit real problems.

---

**Key Takeaways**

1. Agents need memory because context windows are finite and tasks span multiple interactions — without memory, every invocation starts from zero.
2. **Short-term memory** is the context window itself; manage it with summarization, sliding windows, and pruning.
3. **Long-term memory** uses vector databases for persistent, semantically-searchable storage across sessions.
4. **Episodic memory** records past experiences and outcomes, helping agents avoid repeating mistakes.
5. **Procedural memory** stores learned skills and procedures — how to do things, retrievable via RAG or baked into the model.
6. Start with a simple vector store + metadata + recency scoring; add complexity only when needed.
