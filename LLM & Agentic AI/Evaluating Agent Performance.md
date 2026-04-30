---
title: Evaluating Agent Performance
tags:
  - agentic-ai
  - phase-7
  - evaluation
  - benchmarks
  - testing
date: 2025-07-15
---

# Evaluating Agent Performance

## Why Evaluation Is Hard

Evaluating an agent isn't like evaluating a function. Functions are deterministic — same input, same output. Agents are **non-deterministic**, **multi-step**, and **open-ended**. The same question might get answered in 3 steps one time and 7 steps the next. Both might be "correct" but with different efficiency.

This is one of the most important (and most neglected) parts of building agents. Without good evaluation, you're flying blind — you don't know if your changes made things better or worse.

---

## What to Measure

### Task Completion Rate
Did the agent achieve the goal? This is the most important metric. If you ask "book a flight to London" and it books a flight to Leeds — that's a failure regardless of how elegant the process was.

### Accuracy / Correctness
Was the output factually correct? For knowledge tasks, are the facts right? For code generation, does the code work?

### Efficiency
How many steps, tokens, and tool calls did it take? An agent that takes 15 steps to do what should take 3 is wasteful (and expensive). Measure:
- Number of LLM calls per task
- Total tokens consumed
- Number of tool calls
- Wall-clock time

### Reliability
Does it work consistently? Run the same task 10 times. If it succeeds 7/10, that's 70% reliability. For production agents, you want 95%+.

### Safety
Did it stay within boundaries? Did it try to access tools it shouldn't? Did it follow [[Guardrails and Safety Layers|guardrails]]? This matters especially for agents with write access to real systems.

---

## Evaluation Approaches

### Benchmark Suites

Standard benchmarks let you compare your agent against baselines:

| Benchmark | Domain | What It Tests |
|-----------|--------|---------------|
| **SWE-bench** | Coding | Fix real GitHub issues (repo-level tasks) |
| **GAIA** | General | Multi-step real-world assistant tasks |
| **AgentBench** | Multi-domain | Web, code, database, OS tasks |
| **HumanEval** | Code generation | Function-level code synthesis |
| **MATH** | Mathematics | Competition-level math problems |
| **WebArena** | Web navigation | Complete tasks on real websites |

SWE-bench is particularly interesting for software engineers — it tests whether an agent can read a GitHub issue, understand a codebase, and produce a working fix.

### Custom Evaluation

For your specific use case, build a test suite:

```python
test_cases = [
    {
        "input": "What's the weather in London?",
        "expected_tool": "get_weather",
        "expected_args": {"city": "London"},
        "expected_contains": ["London", "temperature"],
        "max_steps": 3
    },
    {
        "input": "Calculate 15% tip on $84.50",
        "expected_tool": "calculate",
        "expected_answer_contains": "12.67",
        "max_steps": 2
    },
    {
        "input": "What's the capital of the country with the largest population?",
        "expected_tools_used": ["search"],
        "expected_answer_contains": "Beijing",
        "max_steps": 5
    }
]

def evaluate_agent(agent, test_cases, runs_per_case=5):
    results = []
    for case in test_cases:
        passes = 0
        for _ in range(runs_per_case):
            result = agent.run(case["input"])
            if validates(result, case):
                passes += 1
        results.append({
            "input": case["input"],
            "pass_rate": passes / runs_per_case,
            "avg_steps": result.steps,
            "avg_tokens": result.total_tokens
        })
    return results
```

### LLM-as-Judge

For open-ended tasks where there's no single "right" answer, use another LLM to grade:

```python
def llm_judge(question: str, agent_answer: str, reference: str = None) -> dict:
    prompt = f"""Rate this agent response on a scale of 1-5 for:
    - Correctness: Is the information accurate?
    - Completeness: Did it fully address the question?
    - Conciseness: Was it appropriately brief?
    
    Question: {question}
    Agent Answer: {agent_answer}
    {"Reference Answer: " + reference if reference else ""}
    
    Return JSON: {{"correctness": N, "completeness": N, "conciseness": N, "reasoning": "..."}}"""
    
    response = openai.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}],
        response_format={"type": "json_object"}
    )
    return json.loads(response.choices[0].message.content)
```

### A/B Testing in Production

Once deployed, compare agent versions on live traffic:
- Route 50% of requests to Agent A, 50% to Agent B
- Measure completion rate, user satisfaction, cost, latency
- Use statistical significance tests before declaring a winner

---

## Metrics Deep Dive

### Pass@1 and Pass@k

- **Pass@1** — correct on the first attempt (strictest)
- **Pass@k** — correct in at least one of k attempts (more lenient)
- For production, Pass@1 is what matters (users don't get retries)
- For development, Pass@5 shows potential (maybe the agent *can* do it, just unreliably)

### Token Efficiency

```
Token Efficiency = Successful Tasks / Total Tokens Consumed
```

An agent that solves 10 tasks using 50K tokens is more efficient than one solving 10 tasks using 200K tokens. This directly maps to cost.

### Tool Call Accuracy

Did the agent call the **right tool** with the **right arguments**?

```python
def tool_call_accuracy(expected_calls, actual_calls):
    correct = 0
    for expected in expected_calls:
        for actual in actual_calls:
            if (actual.name == expected["name"] and 
                matches_args(actual.args, expected["args"])):
                correct += 1
                break
    return correct / len(expected_calls)
```

### Recovery Rate

How often does the agent self-correct after an error?

```
Recovery Rate = (Errors followed by successful retry) / Total Errors
```

A high recovery rate means your error handling is working — the agent reads errors and adapts.

---

## Evaluation Challenges

### Non-Determinism
Same input → different outputs every time. You **must** run multiple trials. A single pass/fail tells you almost nothing. Run at least 5-10 trials per test case.

### Subjectivity
"Write a summary of this article" — what's a good summary? Different graders might disagree. Use multiple LLM judges and aggregate scores.

### Cost
Running evaluations is expensive. A 100-case test suite with 5 runs each = 500 agent executions. At maybe $0.10-$1.00 per execution, that's $50-$500 per eval run. Budget for this.

### Overfitting to Benchmarks
Optimizing for SWE-bench might not improve real-world performance. Use benchmarks as a signal, not as the objective. Always pair with your own domain-specific test cases.

### The "Almost Right" Problem
An agent that gives a 90% correct answer is hard to score. Is that a pass or fail? Define your criteria clearly before running evaluations — ambiguity in grading wastes results.

---

## Practical Approach

If you're building an agent and don't know where to start with eval:

1. **Write 10-20 representative test cases** covering your core use cases
2. **Define pass/fail criteria** for each (be specific — "contains X" or "calls tool Y")
3. **Run each test 5 times**, calculate pass rate
4. **Measure cost and latency** alongside accuracy
5. **Set a baseline**, then measure again after changes
6. **Automate it** — run your eval suite in CI on every change

Start simple. A spreadsheet tracking pass rates across 20 test cases is infinitely better than no evaluation at all.

---

## Key Takeaways

1. Agent evaluation is hard because agents are **non-deterministic and multi-step** — you must run multiple trials per test case
2. Measure five things: **task completion, accuracy, efficiency, reliability, and safety**
3. Use **benchmark suites** (SWE-bench, GAIA) for comparison, but always build **custom test cases** for your specific domain
4. **LLM-as-judge** works surprisingly well for grading open-ended outputs — use it when there's no single right answer
5. **Pass@1** is what matters in production — users don't get retries, so reliability is paramount
6. Evaluation is expensive but essential — budget for it and **automate it in CI** so it runs on every change
7. Start with 10-20 test cases and a spreadsheet — imperfect evaluation is infinitely better than no evaluation
