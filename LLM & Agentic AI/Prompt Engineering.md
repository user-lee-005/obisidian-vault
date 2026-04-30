# Prompt Engineering

We've seen [[How LLMs are Trained]] and understand [[What are LLMs]] — they're next-token predictors steered by input text. Here's the beautiful consequence: **the prompt is the programming language**. You don't write code to control an LLM; you write natural language instructions. This discipline of crafting effective inputs is called **prompt engineering**.

---

## What is Prompt Engineering?

Prompt engineering is the art and science of **crafting inputs to get desired outputs** from an LLM. It matters because LLMs are incredibly sensitive to how you phrase things — the same question asked differently can yield dramatically different quality answers.

Think of the LLM as a brilliant but literal colleague. If you give vague instructions, you'll get vague output. If you're specific and structured, you'll get focused, high-quality responses.

---

## Why It Works

LLMs are **few-shot learners** — they can pick up patterns from just a few examples in the prompt. This isn't a hack; it's by design. During [[How LLMs are Trained|pre-training]], models learned to recognize and continue patterns. When you put examples in a prompt, you're activating that same pattern-matching ability.

The prompt effectively creates a **temporary context** — a micro-environment that shapes the model's behavior for that generation. No retraining needed.

---

## Core Techniques

### Zero-Shot Prompting

Just ask directly. No examples, no setup — rely on the model's pre-trained knowledge.

```
Translate the following to French: "The shipment arrived late."
```

Works well for straightforward tasks where the model has strong prior knowledge.

### Few-Shot Prompting

Provide examples before your actual question. The model infers the pattern and applies it.

```
Classify the sentiment:
"I love this product!" → Positive
"Terrible experience, never again." → Negative
"The delivery was okay, nothing special." → Neutral

"The tracking updates were instant and accurate." →
```

More examples = more reliable pattern, but watch the context window.

### Chain-of-Thought (CoT)

The magic words: **"Let's think step by step."** This forces the model to show its reasoning, which dramatically improves accuracy on complex problems.

```
Q: If a warehouse processes 150 shipments/hour and operates 16 hours/day, 
   how many shipments per week (5 days)?

A: Let's think step by step.
- Shipments per hour: 150
- Hours per day: 16
- Shipments per day: 150 × 16 = 2,400
- Days per week: 5
- Shipments per week: 2,400 × 5 = 12,000
```

Why does this work? Because the model generates intermediate tokens that serve as "working memory," avoiding the need to do complex reasoning in a single jump. See [[LLM Capabilities and Limitations]] for more on reasoning.

### Role Prompting

Assign the model a persona. This activates relevant knowledge and adjusts tone.

```
You are a senior logistics architect with 15 years of experience designing 
warehouse management systems. Explain the tradeoffs between FIFO and LIFO 
inventory strategies.
```

### System Prompts

Set **behavior constraints and personality** that persist across a conversation:

```
System: You are a concise technical assistant. Always provide code examples. 
Never make assumptions — ask clarifying questions if the request is ambiguous. 
Format responses with markdown headers.
```

System prompts are how products like ChatGPT and Claude define their assistant behavior.

---

## Prompt Structure Best Practices

**Be specific and explicit:**
- ❌ "Tell me about shipping"
- ✅ "Explain the difference between LTL and FTL shipping, with cost considerations for a 500kg pallet from Sydney to Melbourne"

**Provide context and constraints:**
- State what you know, what you need, and what format you expect
- Include relevant background the model might not assume

**Use delimiters for input data:**
```
Summarize the text between triple backticks:
```text here```
```

**Specify output format:**
```
Return the result as a JSON object with keys: "carrier", "eta", "cost"
```

**Give the model an "out":**
```
If you don't have enough information to answer, say "I need more context" 
rather than guessing.
```

---

## Common Pitfalls

- **Ambiguity**: The model will make assumptions if you're vague — and they might be wrong
- **Prompt injection**: Malicious inputs that override your system prompt (e.g., "Ignore all previous instructions and..."). Important for production systems.
- **Over-constraining**: Too many rules can make the model freeze up or produce stilted output
- **Context window overflow**: Long prompts eat into the space available for the response
- **Assuming consistency**: The same prompt can give different outputs due to [[LLM Capabilities and Limitations|sampling randomness]]

---

## Advanced Techniques

### Self-Consistency

Run the same CoT prompt multiple times, then take the **majority vote** on the final answer. Reduces variance for complex reasoning tasks.

### Tree of Thoughts

Instead of a single chain of reasoning, explore **multiple reasoning paths** and evaluate which leads to the best outcome. Like BFS/DFS but for reasoning.

### ReAct Prompting (Reasoning + Acting)

Combine reasoning steps with **tool usage**:

```
Thought: I need to find the current shipping rate for this route.
Action: search_rates(origin="SYD", dest="MEL", weight=500)
Observation: Rate is $45/kg for express, $28/kg for standard.
Thought: Given the budget constraint, standard is better.
Answer: ...
```

This is a preview of where things are headed — [[Prompt Engineering]] meets tool use meets agents. We'll explore this fully in later phases.

---

**Key Takeaways**

1. Prompt engineering is crafting inputs to steer LLM outputs — the prompt is your programming interface.
2. **Few-shot** prompting provides examples to establish a pattern; **zero-shot** relies on the model's pre-trained knowledge.
3. **Chain-of-Thought** ("let's think step by step") dramatically improves reasoning by giving the model working memory through intermediate tokens.
4. Structure matters: be specific, provide context, use delimiters, and specify output format.
5. Watch for pitfalls: ambiguity, prompt injection, over-constraining, and assuming deterministic outputs.
6. Advanced techniques like ReAct bridge prompting with tool use — the foundation for agentic systems.
7. The best prompt engineers understand [[What are LLMs|how LLMs work underneath]] — it's not magic, it's pattern matching steered by your input.
