# What are LLMs

Now that we understand [[The Transformer Architecture]], [[Attention Mechanism]], and [[Tokenization and Vocabulary]], let's zoom out and look at what happens when you scale transformers up — way up. Welcome to the world of **Large Language Models**.

---

## Definition

A **Large Language Model (LLM)** is a transformer-based neural network with billions of parameters, trained on massive amounts of text data. The goal? Learn the statistical structure of language so well that the model can generate, understand, and reason about text in ways that feel almost human.

Think of it this way: if a transformer is the engine, an LLM is an entire rocket ship built around that engine — fueled by internet-scale data and trained with enormous compute.

---

## The "Large" in Large Language Models

The word "large" refers to **parameter count** — the number of learnable weights in the model:

- **Millions** (early era): BERT had 110M–340M parameters
- **Billions** (current mainstream): GPT-3 had 175B, Llama 3 has 8B–70B
- **Trillions** (frontier): GPT-4 is rumoured to be ~1.8T (mixture of experts)

**Why does scale matter?** Because of **emergent abilities** — capabilities that appear suddenly as models grow larger. A 1B parameter model might struggle with basic reasoning, but a 100B+ model can suddenly do chain-of-thought math, write code, and translate between languages it was barely trained on. Nobody explicitly programmed these abilities — they *emerged* from scale.

---

## Three Architectures

Not all LLMs are built the same. The original transformer had both an encoder and decoder, but modern LLMs typically use one of three variations:

### Encoder-Only (e.g., BERT)

- **Direction**: Bidirectional — sees the full context at once
- **Strength**: Understanding and classification tasks
- **Use cases**: Sentiment analysis, named entity recognition, search ranking
- **How it works**: Masks random tokens, learns to fill in blanks (Masked Language Modeling)

### Decoder-Only (e.g., GPT, Llama, Claude)

- **Direction**: Autoregressive — reads left to right, generates one token at a time
- **Strength**: Text generation, conversation, reasoning
- **Use cases**: Chatbots, code generation, creative writing
- **How it works**: Predicts the next token given all previous tokens

### Encoder-Decoder (e.g., T5, BART)

- **Direction**: Encoder reads input bidirectionally, decoder generates output autoregressively
- **Strength**: Tasks with clear input → output structure
- **Use cases**: Translation, summarization, question answering

> Most modern LLMs (GPT-4, Claude, Llama) are **decoder-only**. The field converged on this architecture because it scales best and is the most general.

---

## How Generation Works

When an LLM "writes" text, it's doing **next-token prediction** in a loop:

1. Take the input (prompt) as a sequence of tokens
2. Predict a probability distribution over the entire vocabulary for the next token
3. Sample from that distribution (picking the next word)
4. Append that token to the sequence
5. Repeat until a stop condition

**Controlling randomness:**

- **Temperature**: Controls how "creative" the model is. Low (0.1) = deterministic and safe. High (1.5) = wild and creative.
- **Top-k sampling**: Only consider the top k most likely tokens (e.g., top-40)
- **Top-p (nucleus) sampling**: Only consider tokens whose cumulative probability reaches p (e.g., 0.95)

These aren't magic — they're just ways to shape which tokens get picked from the probability distribution.

---

## Emergent Abilities

One of the most fascinating (and debated) aspects of LLMs is **emergence**. As models scale, they suddenly develop abilities nobody trained them on:

- **In-context learning**: Learning new tasks from examples in the prompt
- **Chain-of-thought reasoning**: Working through problems step by step
- **Code generation**: Writing functional programs
- **Translation**: Even between language pairs with minimal training data
- **Analogical reasoning**: Understanding metaphors and analogies

This is what makes LLMs feel different from traditional ML — they're not single-purpose tools.

---

## Foundation Models

Traditional ML: you train one model per task (spam classifier, sentiment model, translation model, etc.). Each needs its own data, pipeline, and maintenance.

**Foundation models** flip this: train one massive model on broad data, then adapt it to many downstream tasks through [[Prompt Engineering]] or [[How LLMs are Trained|fine-tuning]]. One model → many applications.

This is why companies like OpenAI, Anthropic, and Google invest billions in training a single model — the payoff is a general-purpose reasoning engine that can be steered to do almost anything with language.

---

**Key Takeaways**

1. LLMs are transformer-based models scaled to billions of parameters and trained on internet-scale text data.
2. Scale unlocks emergent abilities — capabilities that weren't explicitly programmed.
3. Most modern LLMs use a **decoder-only** architecture (autoregressive, left-to-right generation).
4. Text generation is next-token prediction in a loop, controlled by temperature and sampling strategies.
5. Foundation models represent a paradigm shift: one model, many tasks — replacing the traditional "one model per task" approach.
6. Understanding LLMs requires understanding both the transformer architecture underneath and the training process on top — which we'll cover in [[How LLMs are Trained]].
