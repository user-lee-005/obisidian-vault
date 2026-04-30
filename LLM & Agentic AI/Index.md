# 🤖 LLM & Agentic AI Notes

> A complete learning path — from "what is AI?" to building autonomous agents from scratch.
> Designed for someone with zero ML background. Java/Spring + Python where needed.

---

## 🧠 Phase 1: ML & AI Foundations

Start here. No prerequisites — just curiosity.

| Topic | Description |
|-------|-------------|
| [[What is AI and ML]] | Definitions, history, types of AI, why it matters now |
| [[How Machines Learn]] | Supervised, unsupervised, reinforcement learning explained simply |
| [[Neural Networks from Scratch]] | Neurons, layers, weights, biases, activation functions |
| [[Training a Model]] | Loss functions, gradient descent, backpropagation (intuition-first) |
| [[Overfitting and Generalization]] | Train/test split, regularization, bias-variance tradeoff |
| [[Math You Actually Need]] | Linear algebra basics, probability, calculus intuition (just enough) |

---

## 📖 Phase 2: Deep Learning & NLP Foundations

How machines understand language — the building blocks of LLMs.

| Topic | Description |
|-------|-------------|
| [[Deep Neural Networks]] | Multi-layer networks, CNNs (brief), RNNs and sequence modeling |
| [[Word Embeddings]] | One-hot encoding, Word2Vec, GloVe — how machines understand text |
| [[Sequence Models and Attention]] | RNN limitations, encoder-decoder, the attention mechanism |
| [[The Transformer Architecture]] | Self-attention, positional encoding, multi-head attention — the paper that changed everything |
| [[Tokenization and Vocabulary]] | BPE, WordPiece, SentencePiece — how text becomes numbers |

---

## 🚀 Phase 3: Large Language Models (LLMs)

The main event — understanding how GPT, Claude, and friends actually work.

| Topic | Description |
|-------|-------------|
| [[What are LLMs]] | GPT, BERT, T5 — decoder vs encoder vs encoder-decoder models |
| [[How LLMs are Trained]] | Pre-training, fine-tuning, RLHF, instruction tuning |
| [[Prompt Engineering]] | Zero-shot, few-shot, chain-of-thought, system prompts |
| [[LLM Capabilities and Limitations]] | Hallucinations, context windows, reasoning, creativity |
| [[The LLM Landscape]] | OpenAI, Anthropic, Meta (Llama), Google, open-source vs closed |
| [[Running LLMs Locally]] | Ollama, llama.cpp, quantization, hardware requirements |

---

## 🔧 Phase 4: Working with LLM APIs & Tooling

Hands-on — integrating LLMs into real applications.

| Topic | Description |
|-------|-------------|
| [[OpenAI API Deep Dive]] | Chat completions, streaming, function calling, structured outputs |
| [[Spring AI Framework]] | Integrating LLMs into Spring Boot apps, chat clients, prompts |
| [[LangChain Fundamentals]] | Chains, prompts, output parsers, memory (Python) |
| [[Vector Databases]] | Embeddings, similarity search, Pinecone, Weaviate, pgvector |
| [[RAG - Retrieval Augmented Generation]] | Architecture, chunking, retrieval strategies, evaluation |
| [[Fine-Tuning vs RAG vs Prompt Engineering]] | When to use which approach |

---

## 🛠️ Phase 5: Function Calling & Tool Use

Teaching LLMs to interact with the outside world.

| Topic | Description |
|-------|-------------|
| [[Function Calling in LLMs]] | How models call external tools, OpenAI function calling spec |
| [[Tool Use Patterns]] | Search, calculators, code execution, API calls |
| [[Building Tool-Using Chatbots]] | End-to-end: LLM + tools + conversation memory |
| [[Structured Output and Parsing]] | JSON mode, Pydantic, response schemas |
| [[Error Handling and Retries]] | When tools fail, fallback strategies |

---

## 🏗️ Phase 6: Agentic AI — Concepts & Architecture

The theory behind autonomous AI agents.

| Topic | Description |
|-------|-------------|
| [[What are AI Agents]] | Definition, autonomy levels, agent vs chatbot |
| [[Agent Architecture Patterns]] | ReAct, Plan-and-Execute, Reflexion, Tree of Thoughts |
| [[Memory Systems for Agents]] | Short-term (context), long-term (vector store), episodic memory |
| [[Multi-Agent Systems]] | Agent collaboration, delegation, orchestration |
| [[Planning and Reasoning]] | Goal decomposition, self-reflection, iterative refinement |
| [[Guardrails and Safety]] | Output validation, content filtering, human-in-the-loop |

---

## 🏆 Phase 7: Building Agents from Scratch

Putting it all together — building real autonomous agents.

| Topic | Description |
|-------|-------------|
| [[Building a Simple Agent in Python]] | ReAct loop, tool calling, memory — from scratch |
| [[Building an Agent with Spring AI]] | Java-based agent with tool integration |
| [[LangGraph and Agent Frameworks]] | Stateful agents, graphs, conditional routing |
| [[MCP - Model Context Protocol]] | Anthropic's protocol for tool/resource integration |
| [[Evaluating Agent Performance]] | Benchmarks, task completion, safety metrics |
| [[Production Considerations]] | Cost, latency, observability, scaling agents |
| [[The Future of Agentic AI]] | Autonomous coding, research agents, multi-modal agents |

---

## 🗺️ Suggested Learning Path

1. Start with [[What is AI and ML]] — get the big picture
2. Work through Phase 1 sequentially — each note builds on the previous
3. Phase 2 introduces NLP — the bridge between basic ML and LLMs
4. Phase 3 is where it clicks — you'll understand how ChatGPT works
5. Phase 4 gets hands-on — start building with Spring AI and LangChain
6. Phase 5 teaches tool use — LLMs that can DO things
7. Phase 6 is architecture — how agents think and plan
8. Phase 7 is the capstone — build your own agents

---

> 💡 **Tip:** Don't skip Phase 1-2 even if you're eager to get to agents. The intuition you build here makes everything else 10x easier to understand.

> 🔗 **Cross-references:** Check [[System Design]] for how agents fit into larger systems, and [[Cloud]] for deploying AI workloads.
