# Planning and Reasoning

An agent without planning is just a chatbot with tools. **Planning** is what transforms a tool-using assistant into something that can tackle complex, multi-step problems. Let's explore how agents think ahead.

---

## Why Planning Matters

Consider a task: "Migrate this Spring Boot app from Java 11 to Java 17."

You can't do that in one step. You need to:
1. Check what Java 11 features are deprecated in 17
2. Scan the codebase for affected code
3. Update the build config
4. Fix compilation errors
5. Run tests
6. Fix test failures
7. Verify the app starts correctly

A **greedy** agent (pure [[Agent Architecture Patterns|ReAct]]) would just start doing *something* without considering the full scope. A **planning** agent maps out the journey before taking the first step.

---

## Goal Decomposition

The first job of planning is breaking a big goal into manageable subtasks.

**Hierarchical Task Networks:**
```
Migrate to Java 17
├── Update build configuration
│   ├── Update pom.xml Java version
│   └── Update CI pipeline JDK version
├── Fix source code compatibility
│   ├── Replace deprecated APIs
│   └── Handle module system changes
└── Validate
    ├── Run unit tests
    ├── Run integration tests
    └── Smoke test deployment
```

**LLM-based decomposition** is straightforward — you literally ask: "Break this task into steps." The model is surprisingly good at this for well-understood domains. Less reliable for novel problems.

The key insight: decomposition itself is a skill. Better decomposition → better execution. This is why some frameworks have a dedicated **planner agent** ([[Multi-Agent Systems]]).

---

## Planning Strategies

### Top-Down
Start with the goal. Ask: "What are the major steps?" Then for each step: "What are the sub-steps?"

- **Pro:** Ensures alignment with the overall objective
- **Con:** May miss low-level constraints that make a step infeasible

### Bottom-Up
Start with available tools and actions. Ask: "What can I do right now? What does that enable next?"

- **Pro:** Grounded in reality — every step is actionable
- **Con:** May not lead toward the goal efficiently

### Iterative (Plan a Few Steps, Execute, Re-Plan)

The pragmatic middle ground. Plan 2–3 steps ahead, execute them, observe results, then plan the next 2–3 steps.

- **Pro:** Adapts to new information. Doesn't waste effort planning steps that may become irrelevant.
- **Con:** May lack global coherence — you might paint yourself into a corner.

This is what most practical agents use. Full upfront planning is brittle; purely reactive is inefficient. **Iterative planning** balances the two.

---

## Self-Reflection and Critique

Planning isn't just about the first plan — it's about **evaluating and improving** as you go.

**After each step:**
- "Did that work? Was the output what I expected?"
- "Am I still on track toward the goal?"
- "Should I adjust the remaining plan?"

**Internal critique** — the model evaluates its own outputs:
- "This code handles the happy path but what about errors?"
- "My search query returned too many results — I should be more specific"
- "I assumed the API returns JSON but I should verify"

**Explicit reflection prompts** make this systematic:
```
Before proceeding, consider:
- What could go wrong with this approach?
- Am I making any assumptions I haven't verified?
- Is there a simpler way to achieve this?
```

This connects to [[Agent Architecture Patterns|Reflexion]] — building structured self-improvement into the agent loop.

---

## Iterative Refinement

Many tasks aren't one-shot. They benefit from a **draft → critique → improve** cycle:

1. **Draft** — Generate an initial solution
2. **Critique** — Evaluate it against requirements/constraints
3. **Improve** — Fix identified issues
4. **Repeat** — Until quality threshold is met

This pattern works for:
- Code generation (write → test → fix → test again)
- Writing tasks (draft → review → revise)
- Data analysis (query → check results → refine query)
- Configuration (apply → validate → adjust)

It's essentially TDD applied to agent behavior — red-green-refactor for AI.

---

## Chain-of-Thought in Planning

When agents reason explicitly ([[Chain of Thought Prompting]]), their planning becomes:

- **Transparent** — You can see *why* the agent chose a particular approach
- **Debuggable** — When something goes wrong, the reasoning trace shows where logic broke down
- **Steerable** — You can correct the agent's reasoning mid-plan

```
Thought: The user wants to add caching. I should first understand the 
current data access patterns to choose the right caching strategy.

Thought: I see database queries in the service layer. These are 
read-heavy with infrequent writes — a good fit for a simple TTL cache.

Thought: The project already uses Spring Boot, so Spring Cache with 
@Cacheable annotations would be idiomatic. Let me check if a cache 
dependency is already present...
```

Explicit reasoning traces are one of the most valuable features of agent systems — treat them as structured logs.

---

## Limitations of LLM Reasoning

Let's be honest about what LLMs *can't* reliably do:

- **No true logical deduction** — LLMs are probabilistic pattern matchers, not symbolic reasoners. They approximate logic, they don't execute it.
- **Long plan fragility** — Error compounds with each step. A 20-step plan where each step has 90% success rate gives you ~12% end-to-end success.
- **Confidently wrong** — LLMs don't "know what they don't know." A plan can sound perfectly logical but contain a fundamental flaw.
- **Domain boundaries** — Plans that require knowledge outside training data are unreliable.

---

## Hybrid Approaches

The pragmatic solution: combine LLM planning with other verification:

- **LLM planning + symbolic verification** — LLM proposes, formal checker validates (e.g., type checker for code)
- **LLM planning + human approval** — Plan is generated, human reviews before execution ([[Guardrails and Safety]])
- **LLM planning + test execution** — Plans include verification steps that provide hard pass/fail signals
- **LLM planning + search** — Use exhaustive search for critical decisions, LLM for heuristic guidance

Don't trust an LLM plan blindly. Verify at checkpoints. The best agent architectures build verification into the planning loop itself.

---

**Key Takeaways**

1. Planning transforms a tool-using chatbot into an agent capable of complex multi-step tasks — without planning, agents are just greedy reactive systems.
2. Goal decomposition (breaking big goals into subtasks) is a skill in itself — better decomposition leads to better execution.
3. **Iterative planning** (plan a few steps → execute → re-plan) is the pragmatic middle ground between brittle upfront plans and purely reactive behavior.
4. Self-reflection and critique at each step help agents catch errors early — "Did that work? Am I on track? What could go wrong?"
5. LLM reasoning has real limits: no true logic, compounding errors, and confident wrongness. Always verify plans at checkpoints.
6. Hybrid approaches (LLM planning + symbolic verification + human approval) are more reliable than pure LLM planning alone.
