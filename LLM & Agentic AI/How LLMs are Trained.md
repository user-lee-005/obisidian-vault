# How LLMs are Trained

We know [[What are LLMs]] — massive transformer models that predict the next token. But how do you get from a randomly initialized pile of billions of weights to something that can write poetry, debug code, and explain quantum physics? The answer is a carefully staged **training pipeline**.

---

## The Training Pipeline Overview

Training an LLM isn't a single step — it's a multi-phase process, each building on the last:

```
Pre-training → Fine-tuning → Alignment
    ↓              ↓              ↓
 "Raw intelligence"  "Task skills"  "Safe & helpful behavior"
```

Think of it like raising a child: pre-training is reading every book in every library (raw knowledge), fine-tuning is going to school (structured learning), and alignment is developing values (being helpful, not harmful).

---

## Phase 1: Pre-training

This is where the magic (and the money) happens. The model learns language from scratch by reading the internet.

### The Data

Pre-training uses **internet-scale text corpora**:

- **Common Crawl**: Billions of web pages
- **Books**: Fiction and non-fiction corpora
- **Wikipedia**: Structured factual knowledge
- **Code**: GitHub repositories (this is why LLMs can code!)
- **Academic papers, forums, news articles**

We're talking **trillions of tokens**. Llama 3 was trained on 15 trillion tokens. GPT-4 likely used even more.

### The Objective

For decoder-only models (GPT, Llama, Claude), the training objective is simple but powerful:

> **Predict the next token**, given all previous tokens.

That's it. No labels, no human annotation — just pure **self-supervised learning**. The model reads text, tries to predict what comes next, gets corrected, and adjusts its weights. Billions of times.

### The Compute

This is not something you do on a laptop:

- **Thousands of GPUs** (NVIDIA A100s, H100s) running in parallel
- **Weeks to months** of continuous training
- **Millions of dollars** in compute costs (GPT-4 estimated at $100M+)
- Sophisticated distributed training: model parallelism, data parallelism, pipeline parallelism

### Data Quality Matters

**Garbage in, garbage out** applies here more than anywhere. Pre-training data is filtered and cleaned:

- Remove duplicates (deduplication)
- Filter toxic/harmful content
- Balance domains (not too much Reddit, enough books)
- Quality scoring (prefer well-written text)

The curation of pre-training data is one of the most impactful (and secretive) parts of building an LLM.

---

## Phase 2: Fine-tuning

After pre-training, you have a model that's great at completing text — but terrible at following instructions or being helpful. It'll happily continue any prompt, including harmful ones. Fine-tuning shapes it into something useful.

### Supervised Fine-Tuning (SFT)

Human annotators write **example conversations**: ideal prompt-response pairs showing how the model should behave.

```
Prompt: "Explain recursion in simple terms"
Response: "Recursion is when a function calls itself to solve a smaller 
           version of the same problem..."
```

The model is trained on thousands of these examples, learning the *format* and *style* of helpful responses.

### Instruction Tuning

A specific flavor of SFT focused on teaching the model to **follow instructions**:

- "Summarize this article in 3 bullet points"
- "Translate this to Spanish"
- "Write a Python function that..."

This is what turns a text-completion engine into an instruction-following assistant. Models like FLAN-T5 and InstructGPT pioneered this approach.

### Domain Fine-tuning

Specializing a general model for specific fields:

- **Medical**: Trained on clinical notes, medical literature
- **Legal**: Contract analysis, case law
- **Code**: Additional training on high-quality code + documentation
- **Finance**: SEC filings, financial reports

This is where [[Running LLMs Locally|local models]] shine — you can fine-tune for your specific use case.

---

## Phase 3: Alignment (RLHF)

Here's the problem: after pre-training and SFT, your model is **smart but potentially misaligned**. It might be helpful sometimes but also toxic, sycophantic, or confidently wrong. We need to align it with human values.

### Reinforcement Learning from Human Feedback (RLHF)

The breakthrough approach (used for ChatGPT, Claude, etc.):

**Step 1: Train a Reward Model**
- Show the LLM multiple prompts, collect several responses for each
- **Human annotators rank the outputs** from best to worst
- Train a separate model (the reward model) to predict these human preferences
- Now you have an automated "quality scorer"

**Step 2: Optimize with RL**
- Use the reward model as a signal to improve the LLM
- **PPO (Proximal Policy Optimization)**: The classic RL algorithm used to update the LLM to produce outputs the reward model scores highly
- **DPO (Direct Preference Optimization)**: A newer, simpler approach that skips the separate reward model and directly trains on preference pairs

The result: a model that's not just capable, but also *helpful, harmless, and honest*.

---

## Constitutional AI

**Anthropic's approach** (used for Claude) adds self-critique:

1. Define a set of **principles** ("be helpful", "don't assist with harm", "be honest about uncertainty")
2. Have the model **critique its own outputs** against these principles
3. Use the critiques to generate training signal — no (or less) human ranking needed

This scales better than pure RLHF because you don't need humans to rank every output. The model internalizes the principles and can self-correct.

---

## Why Alignment is Hard

Alignment isn't a solved problem. Two key challenges:

**Goodhart's Law**: "When a measure becomes a target, it ceases to be a good measure."
- If you reward the model for *seeming* helpful, it might learn to be sycophantic (always agreeing) rather than genuinely helpful
- If you reward safety, it might become overly cautious and refuse harmless requests

**Reward Hacking**: The model finds unexpected ways to maximize its reward signal without actually doing what you want. Like a student who learns to game the rubric instead of learning the material.

This is an active area of research and one reason why companies like Anthropic invest heavily in safety — as models get more capable, alignment becomes more critical.

---

**Key Takeaways**

1. LLM training has three phases: **pre-training** (raw language learning), **fine-tuning** (task skills), and **alignment** (safe behavior).
2. Pre-training is self-supervised next-token prediction on trillions of tokens — costing millions of dollars in compute.
3. Fine-tuning (SFT) turns a text-completion engine into an instruction-following assistant using human-written examples.
4. RLHF aligns models with human preferences by training a reward model on human rankings, then optimizing the LLM against it.
5. Constitutional AI (Anthropic) uses principle-based self-critique as a more scalable alignment approach.
6. Alignment remains hard due to Goodhart's Law and reward hacking — making models helpful without being sycophantic or overly cautious is an unsolved challenge.
7. Data quality at every stage (pre-training corpus, SFT examples, human rankings) is the single biggest determinant of model quality.
