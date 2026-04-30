# What is AI and ML

> Imagine teaching a child to recognize cats — you don't write a rulebook, you show them hundreds of cats until they "get it." That's the essence of machine learning.

---

## What is Artificial Intelligence?

**Artificial Intelligence** is the broad field of making machines do things that would require intelligence if done by a human. That's it. Any system that perceives its environment and takes actions to achieve a goal can be called AI.

**Simple examples you already use:**
- Your phone's face unlock
- Google Maps finding the fastest route
- Email spam filters
- Netflix recommendations

AI isn't magic — it's math, data, and clever engineering.

---

## A Brief History

| Era | Milestone |
|-----|-----------|
| **1950** | Alan Turing asks "Can machines think?" — proposes the Turing Test |
| **1956** | The term "Artificial Intelligence" is coined at Dartmouth Conference |
| **1960s-70s** | Early optimism — rule-based "expert systems" |
| **1980s** | First "AI Winter" — hype dies, funding dries up |
| **1990s** | Statistical methods gain ground, IBM Deep Blue beats Kasparov (1997) |
| **2012** | Deep Learning breakthrough — AlexNet crushes image recognition competition |
| **2017** | Google's "Attention is All You Need" paper — Transformers are born |
| **2020s** | GPT, DALL-E, ChatGPT — large language models go mainstream |

The field has had cycles of hype and disappointment, but we're now in the most sustained period of progress ever.

---

## AI vs ML vs Deep Learning

Think of it as nested circles:

- **AI** (outermost) — any technique that enables machines to mimic human intelligence
  - **Machine Learning** (middle) — a *subset* of AI where machines learn from data instead of being explicitly programmed
    - **Deep Learning** (innermost) — a *subset* of ML using [[Neural Networks from Scratch|neural networks]] with many layers

**Key insight:** All deep learning is ML, all ML is AI — but not the other way around. A rule-based chess engine is AI but *not* ML. A spam filter that learns from examples is ML. A language model with billions of parameters is deep learning.

---

## Types of AI

- **Narrow AI (Weak AI)** — designed for one specific task. This is *everything* that exists today. Siri, AlphaGo, GPT-4 — they're all narrow AI, even if impressively capable.
- **General AI (AGI)** — hypothetical. Would match human-level intelligence across *all* domains. Doesn't exist yet.
- **Super AI (ASI)** — theoretical. Would surpass human intelligence in every way. Science fiction territory for now.

Don't let the hype confuse you — we live in the age of narrow AI, and it's already transforming industries.

---

## Why Did AI Explode Recently?

Three ingredients came together at the right time:

1. **Data** — the internet created oceans of text, images, and video to learn from
2. **Compute** — GPUs (originally for gaming) turned out to be perfect for matrix math in neural networks
3. **Algorithms** — breakthroughs like backpropagation, dropout, attention mechanisms, and Transformers

It's not that the ideas are new — many were proposed decades ago. We just finally had the data and hardware to make them work at scale.

---

## Real-World Examples

| Domain | AI Application |
|--------|---------------|
| **E-commerce** | Recommendation systems (Amazon, YouTube) |
| **Transportation** | Self-driving cars (Tesla, Waymo) |
| **Healthcare** | Medical image diagnosis, drug discovery |
| **Language** | Translation, chatbots, large language models (GPT, Claude) |
| **Finance** | Fraud detection, algorithmic trading |
| **Logistics** | Route optimization, demand forecasting |

These aren't future possibilities — they're deployed *today* in production systems.

---

## **Key Takeaways**

1. AI is any system that mimics human intelligence; ML is a subset that learns from data rather than following hard-coded rules.
2. Deep Learning is ML with many-layered [[Neural Networks from Scratch|neural networks]] — it's behind most recent breakthroughs.
3. All current AI is "narrow" — excellent at specific tasks, but not generally intelligent.
4. The explosion in AI progress comes from the convergence of big data, powerful GPUs, and algorithmic breakthroughs.
5. Understanding [[How Machines Learn]] is your next step to see *how* these systems actually work under the hood.
