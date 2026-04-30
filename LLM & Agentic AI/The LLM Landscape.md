# The LLM Landscape

We've covered [[What are LLMs]], [[How LLMs are Trained]], and their [[LLM Capabilities and Limitations|capabilities and limitations]]. Now let's map the actual players. The LLM space moves fast — models that were state-of-the-art six months ago get surpassed regularly. But understanding the landscape helps you make informed decisions about which models to use and when.

---

## Closed-Source / API-First Models

These are models you access through an API. You can't see the weights, modify the architecture, or run them on your own hardware. What you get: cutting-edge capabilities, managed infrastructure, and rapid iteration.

### OpenAI (GPT Family)

- **GPT-4 / GPT-4o**: The flagship. Strong reasoning, multimodal (text + image), large context (128K)
- **o1 / o1-mini**: "Reasoning" models — they spend more compute at inference time to think through problems (chain-of-thought internally)
- **Strengths**: General-purpose excellence, strong coding, massive ecosystem
- **Pricing**: Token-based (input and output tokens priced separately). GPT-4 is expensive; GPT-4o is cheaper with similar quality
- **Best for**: Complex reasoning, multimodal tasks, when quality matters more than cost

### Anthropic (Claude Family)

- **Claude 3.5 Sonnet**: Best balance of quality, speed, and cost. Excellent at code and analysis
- **Claude 3 Opus**: Highest capability tier. Deep reasoning and nuanced understanding
- **Claude 3 Haiku**: Fast and cheap for simpler tasks
- **Strengths**: Safety focus, long context (200K tokens), strong instruction-following, excellent at structured outputs
- **Best for**: Long document analysis, code generation, tasks requiring careful reasoning and safety

### Google (Gemini Family)

- **Gemini Pro**: Competent general-purpose model, multimodal from day one
- **Gemini Ultra**: Frontier model competing with GPT-4 and Opus
- **Gemini Flash**: Lightweight, fast, and cost-effective
- **Strengths**: Native multimodality (text, image, video, audio), enormous context windows (1M+ tokens), integration with Google ecosystem
- **Best for**: Multimodal tasks, very long context scenarios, Google Cloud integrations

---

## Open-Source / Open-Weights Models

These models release their weights publicly. You can download them, run them locally, fine-tune them, and deploy them on your own infrastructure. The tradeoff: generally slightly less capable than frontier closed models, but you get full control.

### Meta — Llama (2, 3, 3.1)

- **Llama 3.1 8B / 70B / 405B**: The open-source heavyweight
- **Truly open**: Weights available, commercially usable (with license)
- **Strengths**: Strong general performance, large community, extensive fine-tuned variants
- **Why it matters**: Proved that open models can approach closed-model quality, especially at 70B+

### Mistral (Mixtral, Mistral Large)

- **Mistral 7B**: Punched way above its weight — better than Llama 2 13B at half the size
- **Mixtral 8x7B**: Mixture of Experts architecture — 47B total params but only uses ~12B per token
- **Mistral Large**: Their closed/API model competing with GPT-4
- **Strengths**: Efficiency, European company (GDPR-friendly), innovative architectures
- **Why it matters**: Pioneered efficient open models and made MoE mainstream

### Others Worth Knowing

- **Falcon** (TII, UAE): Early open model, large parameter counts
- **Phi** (Microsoft): Tiny models (1.3B–14B) trained on high-quality data, surprisingly capable for their size
- **Qwen** (Alibaba): Strong multilingual model, competitive at various sizes
- **DeepSeek** (China): Impressive reasoning, efficient training, open weights

---

## Open vs Closed: The Tradeoffs

| Factor | Closed (API) | Open (Self-hosted) |
|--------|-------------|-------------------|
| **Capability** | Usually highest | Catching up fast |
| **Cost** | Per-token pricing, can get expensive at scale | Fixed infra cost, predictable |
| **Privacy** | Data goes to third party | Data stays on your servers |
| **Control** | Limited to API parameters | Full control: fine-tune, modify, deploy anywhere |
| **Latency** | Network overhead | Can be optimized for your workload |
| **Customization** | Prompt-only (mostly) | Fine-tune on your domain data |
| **Maintenance** | Managed for you | You own the infra and updates |

For a logistics company handling sensitive shipment data? Privacy and control might push you toward open models. For a quick prototype? API-first is faster to ship.

---

## Model Selection Criteria

When choosing a model for a task, consider:

1. **Task fit**: Does the model excel at your specific use case? (coding, analysis, conversation, etc.)
2. **Context length**: How much input do you need to process? (8K might not cut it for document analysis)
3. **Cost**: At your expected volume, what's the monthly bill? Open models have fixed infra costs; APIs scale linearly.
4. **Latency**: How fast does the response need to be? Smaller models and API-optimized models are faster.
5. **Privacy**: Can your data leave your infrastructure? Regulated industries often can't use external APIs.
6. **Quality threshold**: Is "good enough" acceptable, or do you need frontier quality?

There's no universal best model — only the best model for *your* constraints.

---

## The Trend: Smaller Models Getting Smarter

One of the most exciting trends in the LLM landscape:

- **Small open models** (7B–14B) are reaching the quality that 70B+ models had a year ago
- **Distillation** techniques transfer knowledge from large models to small ones
- **Better data** matters more than bigger models (Phi proved this)
- **Quantized models** run on consumer hardware with minimal quality loss (see [[Running LLMs Locally]])

The implication: you might not need GPT-4 for everything. A well-chosen 8B model might handle 80% of your tasks at 1% of the cost.

---

## Mixture of Experts (MoE)

A key architecture innovation worth understanding:

Traditional models use **all parameters** for every token. MoE models have multiple "expert" sub-networks and a **router** that selects which experts to activate for each token.

- **Mixtral 8x7B**: 8 expert networks of 7B each (47B total), but only 2 are active per token (~12B active)
- **Benefit**: Get large-model quality with small-model inference cost
- **GPT-4** is widely believed to use MoE internally

Think of it like a hospital: you have many specialists, but for each patient (token), only the relevant specialists are consulted. Efficient and effective.

---

**Key Takeaways**

1. The landscape splits into **closed-source APIs** (OpenAI, Anthropic, Google) for cutting-edge capability and **open-weights models** (Llama, Mistral, Phi) for control and privacy.
2. No single model is best for everything — selection depends on task, cost, latency, privacy, and quality requirements.
3. Open models are **catching up fast** — Llama 3.1 70B competes with earlier GPT-4 versions.
4. The trend is toward **smaller, smarter models**: better training data and techniques matter more than raw size.
5. **Mixture of Experts (MoE)** gives large-model quality at small-model cost by activating only relevant sub-networks per token.
6. For production systems, model choice is a tradeoff matrix — not a simple "pick the biggest one" decision.
7. This landscape changes every few months. Stay current, but understand the *patterns* (open vs closed, big vs small, general vs specialized) — those are more stable than any specific model name.
