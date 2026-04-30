# What are AI Agents

## Definition

An **AI agent** is a system that uses an LLM as its "brain" to **autonomously decide** what actions to take to accomplish a goal. Think of it like a microservice that doesn't just respond to requests — it *plans*, *acts*, and *adapts* based on results.

The key word is **autonomy**. A chatbot waits for your input. An agent takes initiative.

---

## Agent vs Chatbot

Let's be clear about the distinction:

| | **Chatbot** | **Agent** |
|---|---|---|
| Behavior | Reactive, responds to prompts | Proactive, pursues goals |
| Turns | Single-turn or simple multi-turn | Multi-step, potentially unbounded |
| Tools | None or minimal | Extensive tool use ([[Tool Use Patterns]]) |
| Planning | None | Decomposes tasks into steps |
| Self-correction | None | Observes results, retries on failure |
| Autonomy | Zero — human drives everything | Variable — can act independently |

A chatbot is a **function**: input → output. An agent is a **running process**: it has a goal, a loop, and state.

---

## The Agent Loop

Every agent, regardless of framework, follows this fundamental loop:

1. **Perceive** — Take in information (user goal, tool outputs, environment state)
2. **Think** — Reason about what to do next (the LLM's job)
3. **Act** — Execute an action (call a tool, write code, send a message)
4. **Observe** — Check the result of that action
5. **Repeat** — Until the goal is achieved or the agent gives up

This is analogous to a game loop or an event loop in software — the agent keeps cycling until a termination condition is met.

---

## Levels of Autonomy

Not all "agents" are equal. Think of it as a spectrum:

- **Level 0 — Simple prompt-response**: A vanilla chatbot. No tools, no planning. Just text in, text out.
- **Level 1 — Tool-using assistant**: The model can call functions ([[Function Calling in LLMs]]) but only does what you explicitly ask. You're still the planner.
- **Level 2 — Single-agent with planning (ReAct)**: The model decides *which* tools to use, *when* to use them, and *how* to chain results. It reasons and acts autonomously within a task.
- **Level 3 — Multi-agent collaboration**: Multiple specialized agents coordinate. A planner agent delegates to a coder agent, a tester agent, etc. ([[Multi-Agent Systems]])
- **Level 4 — Fully autonomous**: Minimal human oversight. The agent operates for extended periods, making high-level decisions. Think: "Deploy this feature to production and monitor it."

Most practical systems today operate at Level 1–2, with Level 3 emerging rapidly.

---

## Real-World Agent Examples

- **Coding agents** (GitHub Copilot agent mode, Devin) — Understand a codebase, plan changes, write code, run tests, iterate on failures
- **Research agents** (Perplexity-style deep research) — Decompose a research question, search multiple sources, synthesize findings, cite references
- **Customer service agents** — Handle complex workflows end-to-end: check order status, initiate refunds, escalate edge cases
- **Data analysis agents** — Receive a dataset, explore it, generate visualizations, write summary reports, answer follow-up questions

---

## Why Agents Are Hard

Building agents sounds great in theory. In practice:

- **Reliability** — LLMs are probabilistic. A 95% success rate per step means ~60% success over 10 steps.
- **Error cascading** — One bad tool call early on corrupts everything downstream. Like a failing microservice in a chain.
- **Safety** — An agent with write access to production can cause real damage ([[Guardrails and Safety]])
- **Cost** — Each reasoning step costs tokens. A complex agent task might use 50–100k tokens total.
- **Observability** — Debugging why an agent took a wrong turn 8 steps ago is painful.

---

## The Human-in-the-Loop Spectrum

There's a fundamental tradeoff:

**Fully autonomous** ←→ **Human approval at every step**

- More autonomy = faster, more scalable, but riskier
- More human oversight = safer, but defeats the purpose of automation

In practice, most production agents use a **tiered approach**: autonomous for low-risk actions, human approval for high-risk ones. "Send a Slack message? Go ahead. Delete a database table? Ask me first."

---

**Key Takeaways**

1. An AI agent is an LLM-powered system that autonomously plans and executes actions toward a goal — it's a running process, not a function call.
2. The core agent loop is Perceive → Think → Act → Observe → Repeat, analogous to an event loop in software.
3. Autonomy exists on a spectrum from simple chatbots (Level 0) to fully autonomous systems (Level 4); most practical agents today are Level 1–2.
4. Agents are hard because LLM errors compound across steps — reliability, safety, cost, and observability are all open challenges.
5. Production agents need thoughtful human-in-the-loop design: full autonomy for safe actions, approval gates for risky ones.
