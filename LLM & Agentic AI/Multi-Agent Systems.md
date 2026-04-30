# Multi-Agent Systems

So far we've talked about single agents — one LLM reasoning and acting in a loop. But what happens when a task is too complex for one agent? We split it up. Just like we decompose monoliths into microservices, we can decompose complex AI tasks into **multiple specialized agents**.

---

## Why Multiple Agents?

- **Specialization** — A coding agent and a research agent need different system prompts, tools, and context. One agent trying to do both gets confused.
- **Parallelism** — Multiple agents can work simultaneously on independent subtasks.
- **Context management** — Each agent has a focused context window instead of one overloaded one.
- **Modularity** — Swap, upgrade, or test agents independently. Sound familiar? It's the microservice argument.

---

## Patterns

### Supervisor/Worker

One **supervisor agent** receives the task and delegates to specialized **worker agents**:

```
User: "Analyze this PR for security issues and performance problems"

Supervisor → Security Agent: "Check for injection vulnerabilities"
Supervisor → Performance Agent: "Check for N+1 queries and unnecessary allocations"

Security Agent → Supervisor: "Found SQL injection in line 42"
Performance Agent → Supervisor: "N+1 query in the user loader"

Supervisor → User: [combined report]
```

The supervisor decides who does what, collects results, and synthesizes the final output. Like a tech lead distributing code review tasks.

### Peer-to-Peer

Agents collaborate as **equals**, passing context directly to each other. No central authority — they coordinate through shared protocols.

Best for: creative tasks, brainstorming, scenarios where no single agent has full authority.

### Pipeline

Sequential handoff — each agent processes and passes to the next. Like a CI/CD pipeline:

```
Planner Agent → Coder Agent → Tester Agent → Reviewer Agent → Deployer Agent
```

Each agent has a narrow scope and clear input/output contract. Simple to reason about, but no parallelism.

### Debate/Adversarial

Two or more agents **argue** about the best approach or solution. A judge agent (or the user) picks the winner:

- Agent A proposes solution X with reasoning
- Agent B critiques X and proposes Y
- They go back and forth
- Final answer is usually better than either alone

Useful for high-stakes decisions where you want to stress-test a conclusion.

---

## Agent Roles (Examples)

In a typical coding assistant multi-agent system:

- **Planner agent** — Decomposes the user's request into subtasks
- **Researcher agent** — Searches codebases, docs, and the web for relevant context
- **Coder agent** — Writes and modifies code based on the plan
- **Tester agent** — Runs tests, generates test cases, validates behavior
- **Reviewer agent** — Critiques code for bugs, style, and security ([[Guardrails and Safety]])
- **Executor agent** — Takes actions in external systems (deploy, create PRs, send notifications)

---

## Communication

How do agents talk to each other?

### Message Passing
Structured messages with clear schemas. Each message has a sender, recipient, content type, and payload. Like RPC between services.

### Shared Memory / Blackboard
A common workspace that all agents can read from and write to. An agent writes findings; other agents pick them up. Like a shared database or event store.

### Event-Driven
Agents subscribe to events and react when relevant ones fire. "When the coder finishes, trigger the tester." Like a pub/sub system or message queue.

---

## Orchestration

Who decides which agent runs when?

- **Static workflows** — Predefined routing, like a DAG. "Always run planner → coder → tester." Simple but inflexible.
- **Dynamic routing** — A supervisor agent decides the next step based on current context. "The tests failed, route back to coder." Flexible but more complex.
- **LLM-as-router** — Use a lightweight model to classify incoming work and route to the right agent. Fast, cheap, and surprisingly effective for well-defined domains.

---

## Challenges

Multi-agent systems inherit distributed system challenges:

- **Error propagation** — One agent's hallucination becomes another's input. Garbage in, garbage out — amplified across agents.
- **Context management** — What does each agent know? Too much context is expensive; too little causes mistakes. Same tension as service boundaries.
- **Cost** — Multiple LLM calls per user request. A 5-agent pipeline might cost 5–10x a single agent. Budget carefully.
- **Debugging** — Tracing why the final output is wrong through 4 agents and 20 LLM calls is painful. You need good logging and observability.
- **Coordination overhead** — Agents waiting on each other, passing redundant context, disagreeing on approach. Just like human teams.

---

## Real-World Example: Coding Assistant

Let's trace a realistic multi-agent coding task:

```
User: "Add rate limiting to the /api/orders endpoint"

1. Planner Agent:
   - Identifies: need to find the endpoint, choose a rate limiting strategy,
     implement it, and test it
   - Creates 4 subtasks

2. Researcher Agent:
   - Finds the endpoint in src/routes/orders.ts
   - Checks if there's an existing rate limiter in the project
   - Discovers express-rate-limit is already a dependency

3. Coder Agent:
   - Implements rate limiting using the existing library
   - Applies it to the /api/orders route

4. Tester Agent:
   - Writes tests: normal requests pass, excessive requests get 429
   - Runs the test suite — all green

5. Reviewer Agent:
   - Checks: configuration is reasonable, no hardcoded values,
     error messages are clear
   - Approves
```

Each agent has focused context, specific tools, and a clear handoff contract.

---

**Key Takeaways**

1. Multi-agent systems decompose complex tasks into specialized agents — the same principle as microservices applied to AI.
2. Common patterns: **supervisor/worker** (delegated), **pipeline** (sequential), **peer-to-peer** (collaborative), and **debate** (adversarial).
3. Agents communicate via message passing, shared memory, or events — choose based on coupling needs and parallelism requirements.
4. Orchestration can be static (DAG), dynamic (supervisor decides), or LLM-routed — tradeoff between simplicity and flexibility.
5. Multi-agent challenges mirror distributed system challenges: error propagation, context boundaries, cost multiplication, and debugging complexity.
6. Start with a single agent; add agents only when a single agent's context or capabilities are genuinely insufficient.
