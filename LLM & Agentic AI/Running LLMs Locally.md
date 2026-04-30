# Running LLMs Locally

We've surveyed [[The LLM Landscape]] and seen the tradeoffs between open and closed models. Now let's get practical — what does it actually take to run an LLM on your own machine? Turns out, it's more accessible than you might think. You don't need a data center; a decent laptop with enough RAM can run surprisingly capable models.

---

## Why Run Locally?

Before we get into the how, let's talk about the *why*:

- **Privacy**: Your data never leaves your machine. No third-party API sees your queries. Critical for sensitive business data, medical records, or proprietary code.
- **Cost**: No per-token charges. Once you have the hardware, inference is essentially free.
- **Offline access**: Works without internet. Useful on planes, in secure environments, or during outages.
- **Customization**: Fine-tune on your own data. Run any model variant. Experiment freely.
- **No rate limits**: No API throttling. Generate as much as you want, as fast as your hardware allows.
- **Latency**: No network round-trip. For certain use cases, local inference is faster.

The tradeoff? You're limited by your hardware, and local models are typically smaller (and less capable) than frontier API models.

---

## Hardware Requirements

Here's the reality check: **RAM/VRAM is the bottleneck** — not CPU speed, not disk space.

### The Rule of Thumb

> **~1 GB of VRAM per billion parameters** (at fp16 / half precision)

So a 7B parameter model needs ~14 GB at fp16, or ~7 GB with int8 quantization. With int4 quantization? ~4 GB.

### What You Need

| Model Size | Minimum VRAM/RAM | Practical Hardware |
|-----------|-----------------|-------------------|
| 3B (Phi-3 mini) | 2–4 GB | Any modern laptop |
| 7–8B (Llama 3 8B, Mistral 7B) | 4–8 GB | Gaming laptop, Mac M1+ (16GB) |
| 13B | 8–12 GB | Mac M2+ (32GB), RTX 3090 |
| 34B | 20–24 GB | RTX 4090, Mac M2 Pro (64GB) |
| 70B | 40–48 GB | Multi-GPU setup, Mac M2 Ultra |

**GPU vs CPU inference:**
- **GPU (CUDA)**: Much faster. NVIDIA GPUs with VRAM are the gold standard.
- **Apple Silicon (Metal)**: M1/M2/M3 Macs use unified memory — GPU and CPU share RAM. Surprisingly good for local inference.
- **CPU-only**: Works but slower. Acceptable for smaller models or if you're patient.

---

## Quantization Explained

Quantization is how we fit large models on smaller hardware by **reducing the precision** of each parameter.

### Precision Levels

| Format | Bits per weight | Memory for 7B model | Quality |
|--------|----------------|---------------------|---------|
| fp32 (full) | 32 bits | ~28 GB | Maximum (training) |
| fp16 (half) | 16 bits | ~14 GB | Near-identical to fp32 |
| int8 | 8 bits | ~7 GB | Very slight quality loss |
| int4 | 4 bits | ~3.5 GB | Noticeable but usable |

### GGUF Format and Quantization Levels

**GGUF** is the standard file format for quantized models (used by llama.cpp and Ollama). You'll see names like:

- **Q4_K_M**: 4-bit quantization, medium quality (best balance for most people)
- **Q5_K_M**: 5-bit, slightly better quality, more memory
- **Q6_K**: 6-bit, near fp16 quality
- **Q8_0**: 8-bit, minimal quality loss
- **Q2_K**: 2-bit, aggressive compression (noticeable degradation)

**The sweet spot for most users**: Q4_K_M or Q5_K_M. You lose very little quality while cutting memory requirements by 4x compared to fp16.

---

## Tools for Local Inference

### Ollama — The Simplest Way to Start

Think of Ollama as the "Docker for LLMs." Pull a model, run it. That's it.

```bash
# Install Ollama (macOS/Linux/Windows)
# Then:
ollama pull llama3:8b
ollama run llama3:8b
```

- **Pros**: Dead simple, handles quantization automatically, exposes OpenAI-compatible API
- **Cons**: Less control over quantization options, slightly more overhead
- **Best for**: Getting started, prototyping, local development

Ollama also exposes an API at `localhost:11434` — you can point your code at it just like you'd call OpenAI's API. This makes it trivial to swap between local and cloud models.

### llama.cpp — The Power User's Choice

A C/C++ implementation of LLM inference. Lightweight, fast, runs on CPU efficiently.

- **Pros**: Maximum performance, fine-grained control, supports GGUF directly
- **Cons**: Command-line focused, more setup required
- **Best for**: Production serving, CPU inference, embedded systems

### vLLM — High-Throughput Serving

When you need to serve models to multiple users simultaneously:

- **Pros**: PagedAttention (efficient memory), continuous batching, high throughput
- **Cons**: Requires GPU, more complex setup
- **Best for**: Self-hosted API serving, team deployments

### Text Generation WebUI — Browser Interface

A Gradio-based web interface for running models locally:

- **Pros**: Nice UI, supports many model formats, easy model switching
- **Cons**: Heavier resource usage, more moving parts
- **Best for**: Experimentation, non-technical users, visual model comparison

---

## Practical Example: Running Llama 3 8B on a Laptop

Let's walk through the simplest path — using Ollama on a MacBook with 16GB RAM (or any machine with 8GB+ available):

```bash
# 1. Install Ollama
# Download from https://ollama.ai or:
# macOS: brew install ollama
# Windows: download installer from website

# 2. Pull the model (downloads ~4.7 GB for Q4_K_M quantization)
ollama pull llama3:8b

# 3. Run interactively
ollama run llama3:8b

# 4. Or use the API (OpenAI-compatible)
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3:8b",
    "messages": [{"role": "user", "content": "Explain microservices in 3 sentences."}]
  }'
```

**What to expect:**
- First run downloads the model (~4.7 GB)
- Generation speed: ~20-40 tokens/second on Apple M2, ~10-20 on CPU-only
- Quality: Surprisingly good for general tasks, code, and conversation
- Memory usage: ~5-6 GB RAM

You can also run specialized models:

```bash
ollama pull codellama:7b    # Code-focused
ollama pull mistral:7b      # General, efficient
ollama pull phi3:mini        # Tiny but capable (3.8B)
```

---

## When Local Makes Sense vs When to Use APIs

| Scenario | Recommendation |
|----------|---------------|
| Handling sensitive/proprietary data | **Local** — data never leaves your machine |
| High volume, predictable workload | **Local** — fixed cost beats per-token pricing |
| Need frontier intelligence (GPT-4, Claude Opus) | **API** — local models can't match frontier quality yet |
| Quick prototyping, low volume | **API** — faster to start, no hardware concerns |
| Offline/air-gapped environments | **Local** — only option |
| Fine-tuning for a specific domain | **Local** — full control over the model |
| Need multimodal (image + text) | **API** — local multimodal is still catching up |
| Development & testing | **Local** — iterate fast without API costs |

A common production pattern: use local models for development and testing, then deploy with APIs for production where you need maximum quality — or use [[The LLM Landscape|open models]] on your own GPU servers for the best of both worlds.

---

**Key Takeaways**

1. Running LLMs locally gives you **privacy, zero cost per token, offline access, and full control** — at the tradeoff of hardware requirements and model size limits.
2. **VRAM/RAM is the bottleneck** — roughly 1 GB per billion parameters at fp16, much less with quantization.
3. **Quantization** (fp16 → int8 → int4) is the key enabler — Q4_K_M gives ~4x memory savings with minimal quality loss.
4. **Ollama** is the simplest starting point: `ollama pull llama3:8b && ollama run llama3:8b` gets you a working local LLM in minutes.
5. A 7–8B quantized model runs comfortably on a 16GB laptop and handles most general tasks surprisingly well.
6. Local vs API isn't binary — many teams use both: local for development and privacy-sensitive tasks, APIs for frontier quality.
7. The barrier to entry keeps dropping. Models get smaller and smarter, quantization gets better, and tools like Ollama make it trivial. If you haven't tried running a model locally, now is the time.
