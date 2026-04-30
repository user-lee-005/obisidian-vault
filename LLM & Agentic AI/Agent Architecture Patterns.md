# Agent Architecture Patterns

Now that we know [[What are AI Agents]], let's look at *how* they're structured. Just like software has design patterns (Strategy, Observer, Pipeline), agents have architecture patterns that determine how they reason and act.

---

## ReAct (Reasoning + Acting)

The most common and foundational pattern. The model alternates between **thinking** and **doing**:

```
Thought: I need to find the user's order status. Let me check the database.
Action: query_orders(user_id="123")
Observation: Order #456, status: "shipped", tracking: "XYZ789"
Thought: I have the info. Let me format a response.
Action: respond("Your order #456 has shipped! Tracking: XYZ789")
```

**How it works:**
- The LLM generates a reasoning trace ("Thought")
- Then picks an action (a [[Function Calling in LLMs|function call]])
- Observes the result
- Reasons about the next step
- Repeats until done

**Strengths:** Simple, effective, widely used. Easy to debug because reasoning is explicit.

**Limitation:** No lookahead planning. It's greedy — decides one step at a time without considering the full path. Like navigating by only looking at the next intersection.

---

## Plan-and-Execute

Imagine the difference between a developer who codes line-by-line vs one who writes pseudocode first. That's Plan-and-Execute:

1. **Plan phase**: Create a full plan (list of numbered steps)
2. **Execute phase**: Work through each step sequentially
3. **Re-plan** (optional): If execution reveals the plan was wrong, revise it

**Example:**
```
Plan:
1. Search the codebase for authentication middleware
2. Identify where JWT validation happens
3. Add rate limiting before the validation step
4. Write tests for the new rate limiter
5. Run the test suite

Executing step 1...
Executing step 2...
[Step 2 reveals there's no middleware — it's inline]
Re-planning: Need to extract auth logic into middleware first...
```

**Strengths:** Better for complex multi-step tasks. The plan provides structure and can be shown to users for approval.

**Tradeoff:** Planning takes tokens and time upfront. For simple tasks, it's overkill.

---

## Reflexion

Some tasks are hard to get right on the first try. Reflexion adds a **self-improvement loop**:

1. **Attempt** — Agent tries to complete the task
2. **Evaluate** — Agent (or a separate evaluator) assesses the result
3. **Reflect** — Agent identifies what went wrong and why
4. **Retry** — Agent tries again with lessons learned

Think of it like TDD for agents: write, fail, reflect, fix, repeat.

**When it helps:** Tasks where first attempts often fail — complex code generation, multi-constraint optimization, writing that must meet specific criteria.

**Stored reflections** become a form of [[Memory Systems for Agents|episodic memory]] — "Last time I tried to parse this API response as JSON and it failed because it returns XML when there's an error."

---

## Tree of Thoughts

ReAct is linear — one thought, one action, move forward. But what if the problem has **multiple valid approaches**?

Tree of Thoughts explores **multiple reasoning paths in parallel**:

```
Goal: Optimize this database query

Branch A: Add an index on the WHERE clause column
Branch B: Rewrite as a JOIN instead of subquery  
Branch C: Add caching at the application layer

Evaluating...
Branch A: estimated 10x improvement, low risk ✓
Branch B: estimated 5x improvement, medium complexity
Branch C: doesn't solve the root cause

→ Pursuing Branch A
```

**How it works:**
- Generate multiple candidate approaches
- Evaluate each one (using the LLM or heuristics)
- Pursue the most promising path
- Backtrack if a path fails

**Good for:** Problems with multiple valid approaches, creative tasks, complex reasoning where the first path isn't always best.

---

## LATS (Language Agent Tree Search)

LATS combines Tree of Thoughts with **Monte Carlo Tree Search** (MCTS) — the algorithm behind AlphaGo:

- Explore multiple action paths
- Simulate outcomes
- Reflect on failures
- Backtrack intelligently to try alternatives

It's the most sophisticated pattern — powerful but expensive. Best for high-stakes tasks where getting the right answer matters more than speed.

---

## Choosing a Pattern

| Task Type | Recommended Pattern |
|---|---|
| Simple tool use (lookup, CRUD) | **ReAct** |
| Complex multi-step workflows | **Plan-and-Execute** |
| Tasks that often need iteration | **Reflexion** |
| Open-ended reasoning with multiple paths | **Tree of Thoughts** |
| High-stakes, complex reasoning | **LATS** |

In practice, many agents **combine** patterns. A Plan-and-Execute agent might use ReAct for individual step execution, with Reflexion if a step fails.

---

**Key Takeaways**

1. **ReAct** (Thought → Action → Observation loop) is the default agent pattern — simple, effective, and easy to debug, but greedy with no lookahead.
2. **Plan-and-Execute** separates planning from execution, giving structure to complex tasks at the cost of upfront token spend.
3. **Reflexion** adds a self-improvement loop — attempt, evaluate, reflect, retry — making agents learn from their own mistakes within a session.
4. **Tree of Thoughts** explores multiple reasoning paths in parallel, useful when problems have several valid approaches.
5. Patterns can be combined — a Plan-and-Execute agent might use ReAct for step execution and Reflexion for error recovery.
6. Choose the simplest pattern that works; complexity has a cost in tokens, latency, and debuggability.
