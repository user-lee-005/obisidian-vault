---
tags:
  - llm
  - rag
  - retrieval
  - phase-4
---

# RAG - Retrieval Augmented Generation

This is arguably the most important pattern in applied LLM engineering. If you build one thing with LLMs, it'll probably involve RAG. Let's understand it deeply.

---

## The Problem RAG Solves

LLMs have two fundamental limitations:

1. **Knowledge cutoff** — GPT-4 doesn't know about your company's shipping procedures updated last week
2. **Hallucination** — When they don't know something, they confidently make it up

RAG solves both: **give the model relevant context at query time** so it can answer from real data.

Think of it as the difference between:
- **Closed-book exam** — The model relies only on what it memorized during training
- **Open-book exam** — The model gets reference materials to look up answers ← This is RAG

---

## Architecture

RAG has two pipelines:

### Indexing Pipeline (offline, one-time)

```
Documents → Chunk → Embed → Store in Vector DB
```

1. **Load documents** — PDFs, web pages, databases, APIs
2. **Chunk** — Split into smaller passages (see chunking strategies below)
3. **Embed** — Convert each chunk to a vector using an [[OpenAI API Deep Dive#Embeddings API|embedding model]]
4. **Store** — Save vectors + metadata in a [[Vector Databases|vector database]]

### Query Pipeline (online, per-request)

```
User Question → Embed → Retrieve → Augment Prompt → Generate
```

1. **Embed the question** — Same embedding model as indexing
2. **Retrieve** — Find the top-k most similar chunks from the vector DB
3. **Augment** — Inject retrieved chunks into the LLM prompt as context
4. **Generate** — LLM answers based on the provided context

```python
# Simplified RAG query pipeline
question = "What's our policy on hazmat shipping?"
question_vector = embed(question)
relevant_chunks = vector_db.search(question_vector, top_k=5)

prompt = f"""Answer based on the following context only.
If the answer isn't in the context, say "I don't know."

Context:
{format_chunks(relevant_chunks)}

Question: {question}"""

answer = llm.generate(prompt)
```

---

## Chunking Strategies

Chunking is where most RAG pipelines succeed or fail. The goal: create chunks that are **self-contained enough to be useful** but **small enough to be specific**.

### Fixed-Size Chunks
- Split every N characters (e.g., 1000)
- Simple but blindly cuts mid-sentence/paragraph
- Use with overlap to mitigate

### Sentence-Based Splitting
- Split on sentence boundaries
- Respects natural language structure
- Can create uneven chunk sizes

### Recursive Character Splitting
- Try splitting on `\n\n` first, then `\n`, then `. `, then ` `
- Preserves structure hierarchically
- **Most commonly used** — the default in [[LangChain Fundamentals]]

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=["\n\n", "\n", ". ", " "]
)
```

### Semantic Chunking
- Use embeddings to detect topic boundaries
- Split where the meaning shifts significantly
- Best quality but most compute-intensive

### Chunk Overlap

Always use overlap (10–20% of chunk size). Without it, important context at chunk boundaries gets lost:

```
Chunk 1: "...containers must be sealed with ISO-standard locks."
Chunk 2: "The locks must be inspected before loading..."
```

With 200-char overlap, both chunks include the complete lock policy.

> [!tip] Start with 1000 chars, 200 overlap
> This is a good default for most text documents. Tune from there based on your retrieval quality.

---

## Retrieval Strategies

### Dense Retrieval (Vector Similarity)
- Embed query → find similar vectors → return chunks
- Great for: semantic understanding, paraphrased questions
- Misses: exact terminology, names, codes

### Sparse Retrieval (BM25 / Keyword)
- Traditional keyword matching with TF-IDF weighting
- Great for: specific terms, product codes, names
- Misses: conceptual similarity, synonyms

### Hybrid Search — The Best of Both

Combine dense + sparse retrieval for superior results:

```python
# Pseudocode for hybrid search
dense_results = vector_db.similarity_search(query, k=10)
sparse_results = bm25_search(query, k=10)

# Reciprocal Rank Fusion to merge
final_results = reciprocal_rank_fusion(dense_results, sparse_results)
```

Why hybrid works: "What's the status of BOL-2024-0892?" needs keyword matching (the BOL number), while "How do we handle shipments stuck at customs?" needs semantic understanding.

### Re-ranking

After initial retrieval, use a **cross-encoder** to re-score results:

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

# Initial retrieval gets 20 candidates
candidates = retriever.get_relevant_documents(query, k=20)

# Re-ranker scores each candidate against the query
scores = reranker.predict([(query, doc.page_content) for doc in candidates])

# Take top 5 after re-ranking
top_docs = [candidates[i] for i in scores.argsort()[-5:][::-1]]
```

Re-ranking is computationally expensive but dramatically improves precision.

---

## Prompt Design for RAG

The prompt template matters. Here's a production-ready pattern:

```
You are a helpful assistant. Answer the user's question based ONLY on 
the provided context. If the context doesn't contain the answer, say 
"I don't have enough information to answer that."

Do not make up information. Cite which document the answer came from.

Context:
---
{chunk_1}
Source: shipping_manual.pdf, page 12
---
{chunk_2}
Source: customs_guide.pdf, page 45
---

Question: {user_question}
```

Key principles from [[Prompt Engineering Techniques]]:
- Explicitly constrain to the provided context
- Give the model permission to say "I don't know"
- Request citations for traceability

---

## Evaluation

How do you know if your RAG pipeline is working well? Measure both stages:

### Retrieval Quality
| Metric | What it measures |
|--------|-----------------|
| **Precision@k** | What % of retrieved chunks are relevant? |
| **Recall@k** | What % of all relevant chunks did we retrieve? |
| **MRR** (Mean Reciprocal Rank) | How high is the first relevant result? |

### Generation Quality
| Metric | What it measures |
|--------|-----------------|
| **Faithfulness** | Does the answer stick to the context? (no hallucination) |
| **Relevance** | Does the answer address the question? |
| **Correctness** | Is the answer factually accurate? |

### RAGAS Framework

RAGAS automates RAG evaluation using LLMs as judges:

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision

results = evaluate(
    dataset=test_dataset,
    metrics=[faithfulness, answer_relevancy, context_precision]
)
```

---

## Common Pitfalls

- **Wrong chunk size** — Too big: noisy context. Too small: lost meaning.
- **Poor embedding model** — Use a quality model. `text-embedding-3-small` is a good minimum.
- **No re-ranking** — Initial retrieval is noisy. Re-ranking dramatically helps precision.
- **Stuffing too much context** — More chunks ≠ better answers. 3–5 relevant chunks usually beats 20 mediocre ones.
- **Ignoring metadata** — Filter by date, source, department. Don't search everything.
- **No evaluation** — You can't improve what you don't measure.

---

## Advanced Techniques

### Multi-Hop RAG
Some questions need information from multiple documents. Retrieve → Generate intermediate answer → Retrieve again → Final answer.

### Query Decomposition
Complex questions get split into sub-questions:
- "Compare our 2023 and 2024 shipping costs" → Two retrieval queries, one per year

### Iterative Retrieval
If initial retrieval doesn't produce a good answer, reformulate the query and try again.

### Parent Document Retrieval
Embed small chunks (for precision) but retrieve the larger parent document (for context). Best of both worlds.

---

## Key Takeaways

1. RAG = **retrieve relevant context at query time** — gives LLMs access to your specific data without fine-tuning
2. Two pipelines: **indexing** (load → chunk → embed → store) and **query** (embed → retrieve → augment → generate)
3. **Chunking** is the most underrated factor — start with 1000 chars, 200 overlap, recursive splitting
4. **Hybrid search** (dense + sparse) outperforms either alone — combine semantic understanding with keyword matching
5. **Re-ranking** with a cross-encoder dramatically improves precision after initial retrieval
6. Always **constrain the prompt** — tell the model to only use provided context and to say "I don't know" when appropriate
7. **Evaluate both retrieval and generation** — use RAGAS or similar frameworks to measure and improve
8. See [[Spring AI Framework]] for Java RAG implementation and [[LangChain Fundamentals]] for Python
