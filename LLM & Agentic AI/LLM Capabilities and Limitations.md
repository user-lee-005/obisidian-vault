# LLM Capabilities and Limitations

By now we understand [[What are LLMs]], [[How LLMs are Trained]], and how to steer them with [[Prompt Engineering]]. But let's be honest about what these models can and *cannot* do. Knowing the boundaries is just as important as knowing the strengths — especially when you're building production systems on top of them.

---

## What LLMs Are Good At

### Text Generation & Transformation

This is their home turf. LLMs excel at:

- **Summarization**: Condensing long documents to key points
- **Translation**: Between dozens of language pairs, often near-professional quality
- **Rewriting**: Changing tone, simplifying jargon, expanding bullet points into prose
- **Creative writing**: Stories, marketing copy, emails, documentation

### Code Generation & Explanation

Modern LLMs trained on code (Codex, GPT-4, Claude, Llama) can:

- Write functional code from natural language descriptions
- Explain complex code in plain English
- Debug errors, suggest fixes, refactor for readability
- Translate between programming languages

### Reasoning (with prompting help)

When guided with [[Prompt Engineering#Chain-of-Thought (CoT)|Chain-of-Thought]], LLMs can:

- Solve multi-step logic problems
- Analyze tradeoffs and make recommendations
- Work through math (with caveats — see limitations below)
- Decompose complex problems into sub-tasks

### Following Complex Instructions

Well-aligned models handle surprisingly intricate instructions:

- "Write a REST API endpoint that validates input, checks permissions, queries the database, and returns paginated JSON results with proper error codes"
- They'll follow formatting rules, constraints, and multi-part requests

### In-Context Learning

Perhaps the most remarkable ability: LLMs learn from **examples in the prompt** without any weight updates. Show it 3 examples of your desired format, and it'll generalize to new inputs. This is why few-shot prompting works.

---

## What LLMs Struggle With

### Hallucinations

The elephant in the room. LLMs **confidently generate false information** — citing papers that don't exist, inventing statistics, describing APIs with wrong method signatures.

**Why does this happen?** Because LLMs are fundamentally doing **token prediction, not truth-seeking**. They produce the *most likely next token* — and "likely" is based on training data patterns, not a verified knowledge base. If a plausible-sounding answer is statistically likely, the model will generate it regardless of factual accuracy.

Hallucinations are not bugs to be fixed — they're inherent to the architecture. We can reduce them (with [[Prompt Engineering|prompting]], RAG, and guardrails) but never fully eliminate them.

### Math and Precise Computation

LLMs process numbers as tokens, not as mathematical objects. They:

- Can solve simple arithmetic and algebra (usually)
- Struggle with multi-digit multiplication, complex calculations
- Make errors in multi-step numerical reasoning
- Cannot reliably do precise computations that a calculator handles trivially

This is why tool use (calling a calculator, running code) is essential for anything requiring mathematical precision.

### Real-Time / Current Information

LLMs have a **knowledge cutoff** — they only know what was in their training data up to a certain date. They cannot:

- Tell you today's stock price
- Know about events after their cutoff
- Access real-time data without external tools

This is a major motivator for **Retrieval-Augmented Generation (RAG)** — connecting LLMs to up-to-date knowledge bases.

### Long-Context Reasoning

Even with large context windows, LLMs can struggle with:

- Finding a needle in a haystack (specific fact buried in a long document)
- Maintaining consistency across very long outputs
- Reasoning about relationships between distant parts of the context

Though this is improving rapidly — models like Claude and Gemini now handle 100K–1M+ token contexts with increasing reliability.

### Consistency Across Conversations

LLMs are stateless by default. Each API call is independent. They:

- Don't remember previous conversations (unless you pass history)
- May contradict themselves within long outputs
- Can give different answers to the same question (sampling randomness)

---

## Context Windows

The **context window** is the maximum number of tokens the model can process in a single call (input + output combined).

**Why it matters**: The context window determines how much information you can provide (documents, examples, conversation history) and how long the response can be.

**How they've grown:**

| Model | Context Window |
|-------|---------------|
| GPT-3 (2020) | 4K tokens |
| GPT-4 (2023) | 8K / 32K tokens |
| Claude 3 (2024) | 200K tokens |
| Gemini 1.5 Pro | 1M+ tokens |

Longer context windows enable new use cases: analyzing entire codebases, processing full books, maintaining long conversations — but more context doesn't always mean better reasoning within that context.

---

## The Stochastic Nature

Same prompt ≠ same output. LLMs are **stochastic** (random) by design:

- **Temperature** introduces randomness in token selection
- Even at temperature 0, floating-point differences across hardware can produce variation
- This is a *feature* for creative tasks but a *challenge* for deterministic systems

For production systems, this means: always validate outputs, never assume reproducibility, and design for variance.

---

## Mitigation Strategies

Knowing the limitations, here's how we work around them:

| Limitation | Mitigation |
|-----------|-----------|
| Hallucinations | **RAG** — ground responses in retrieved documents |
| Math errors | **Tool use** — let the model call calculators, code interpreters |
| Knowledge cutoff | **Search/retrieval** — connect to live data sources |
| Inconsistency | **Structured outputs** — JSON schemas, validation layers |
| Safety issues | **Guardrails** — content filters, output classifiers |

These mitigations aren't just patches — they're core architectural patterns for building reliable LLM applications. We'll explore them in depth in later phases.

---

## The "Stochastic Parrot" Debate

Are LLMs just sophisticated autocomplete? Or is something deeper happening?

**The "stochastic parrot" view**: LLMs are statistical pattern matchers that remix training data — no understanding, just next-token prediction. They're mirrors reflecting our text back at us.

**The "emergent intelligence" view**: At sufficient scale, pattern matching becomes indistinguishable from reasoning. The model develops internal representations that generalize beyond training data — something understanding-like emerges.

The truth likely lies somewhere in between — and the debate matters for how much we should trust and rely on these systems. For practical purposes: **treat LLMs as very capable but fallible tools**, and design systems with appropriate verification.

---

**Key Takeaways**

1. LLMs excel at text generation, code, instruction-following, and in-context learning — but they have real, structural limitations.
2. **Hallucinations are inherent** to next-token prediction. They're not bugs — they're a fundamental property of the architecture.
3. LLMs can't do reliable math, access real-time data, or maintain state across calls without external help.
4. Context windows determine how much information a model can process — they've grown from 4K to 1M+ tokens.
5. The stochastic nature means you must design for variance — never assume the same prompt gives the same output.
6. Mitigation strategies (RAG, tools, guardrails) are essential for production systems and will be explored in later phases.
7. Understanding limitations isn't pessimism — it's engineering. Knowing what can go wrong lets you build systems that handle it gracefully.
