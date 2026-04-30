---
tags:
  - llm
  - vector-database
  - embeddings
  - phase-4
---

# Vector Databases

We've seen how [[OpenAI API Deep Dive#Embeddings API|embeddings]] turn text into vectors. But where do you *store* millions of these vectors and search them efficiently? Enter vector databases — the infrastructure that makes [[RAG - Retrieval Augmented Generation|RAG]] possible at scale.

---

## The Problem

Imagine you have 100,000 support documents. A user asks: "How do I handle a customs hold for perishable goods?"

**Traditional search** (keyword/SQL LIKE): Looks for exact words. Misses documents that talk about "import delays for temperature-sensitive cargo" — same meaning, different words.

**Semantic search** (vector): Converts the question to a vector, finds documents with *similar meaning* regardless of exact wording. This is what we want.

---

## Embeddings Recap

An embedding model (like OpenAI's `text-embedding-3-small`) converts text into a dense vector — a list of floats (e.g., 1536 dimensions). Related concepts end up *close together* in this high-dimensional space.

```
"container shipping"  → [0.12, -0.34, 0.56, ..., 0.78]  (1536 dims)
"freight transport"   → [0.11, -0.32, 0.55, ..., 0.77]  (very similar!)
"chocolate cake"      → [-0.45, 0.89, -0.12, ..., 0.23] (very different)
```

The connection to [[Word Embeddings]] is direct — same concept, but these are produced by large pre-trained models and capture much richer semantics.

---

## Similarity Search

Given a query vector, how do we find the most similar vectors in our database?

| Method | Formula intuition | Best for |
|--------|------------------|----------|
| **Cosine similarity** | Angle between vectors (ignores magnitude) | Normalized embeddings |
| **Euclidean distance** | Straight-line distance in space | When magnitude matters |
| **Dot product** | Combines angle and magnitude | Pre-normalized vectors |

In practice, **cosine similarity** is the most common for text embeddings. Two vectors pointing in the same direction = semantically similar.

---

## What Vector Databases Do

A vector database is optimized for three operations:
1. **Store** — Persist vectors alongside their metadata (source doc, page number, etc.)
2. **Index** — Build data structures for fast approximate search
3. **Search** — Given a query vector, find the top-k most similar vectors quickly

The challenge: brute-force search (compare against every vector) is O(n). With millions of vectors, that's too slow. We need smarter indexing.

---

## Indexing Algorithms

You don't need to implement these — just understand the tradeoffs.

### Flat (Brute Force)
- Compare query against *every* vector
- **Exact** results, but O(n) — doesn't scale
- Fine for < 10,000 vectors

### IVF (Inverted File Index)
- Partition vectors into clusters (using k-means)
- At search time, only scan the nearest clusters
- **Faster** but might miss results in distant clusters
- Tune `nprobe` (how many clusters to check)

### HNSW (Hierarchical Navigable Small World)
- Build a graph where similar vectors are connected
- Navigate the graph layer by layer to find neighbors
- **Fast and accurate** — the most popular algorithm
- Higher memory usage (stores the graph)

### PQ (Product Quantization)
- Compress vectors into smaller representations
- Trade accuracy for **memory efficiency**
- Great when you have billions of vectors and limited RAM

> [!tip] In practice
> Most vector databases default to HNSW. It's the best general-purpose choice — fast with high recall.

---

## Popular Vector Databases

### Pinecone
- **Fully managed** — no infrastructure to maintain
- Simple REST/gRPC API
- Serverless option (pay per query)
- Great for: teams that don't want to operate infrastructure

### Weaviate
- **Open-source** with managed cloud option
- Hybrid search: combine vector similarity + BM25 keyword search
- Built-in vectorization (can call embedding models for you)
- Great for: teams wanting both semantic and keyword search

### Milvus / Zilliz
- **Open-source** (Zilliz is the managed version)
- Designed for massive scale (billions of vectors)
- Multiple index types, GPU acceleration
- Great for: high-performance, large-scale deployments

### pgvector
- **PostgreSQL extension** — add vector search to your existing Postgres!
- SQL interface: `SELECT * FROM docs ORDER BY embedding <=> query_vector LIMIT 5`
- Great for: teams already using PostgreSQL who want to avoid another database

```sql
-- pgvector example
CREATE EXTENSION vector;

CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding vector(1536)
);

-- Create HNSW index
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);

-- Similarity search
SELECT content, 1 - (embedding <=> '[0.12, -0.34, ...]') AS similarity
FROM documents
ORDER BY embedding <=> '[0.12, -0.34, ...]'
LIMIT 5;
```

### Chroma
- **Lightweight**, Python-native
- Runs in-memory or with SQLite persistence
- Zero configuration to get started
- Great for: prototyping, local development, small datasets

---

## Metadata Filtering

Pure vector search isn't always enough. You often want to combine it with traditional filters:

```
"Find shipping documents similar to my query, 
 BUT only from 2024, AND only about ocean freight"
```

Most vector databases support **pre-filtering** or **post-filtering** on metadata:

```python
# Pinecone example
results = index.query(
    vector=query_embedding,
    top_k=10,
    filter={
        "year": {"$gte": 2024},
        "transport_mode": "ocean"
    }
)
```

This is critical for production RAG systems — you rarely want to search *everything*.

---

## Embedding Models

Your choice of embedding model determines the quality of your vector representations:

| Model | Dimensions | Strengths |
|-------|-----------|-----------|
| OpenAI `text-embedding-3-small` | 1536 | Great quality, affordable |
| OpenAI `text-embedding-3-large` | 3072 | Best quality from OpenAI |
| Cohere `embed-v3` | 1024 | Multilingual, efficient |
| `sentence-transformers` (open-source) | 384–1024 | Free, run locally |
| Voyage AI | 1024 | Strong for code and technical docs |

> [!note] Match your embedding model
> You must use the **same** embedding model for indexing and querying. You can't embed documents with OpenAI and query with Cohere — the vector spaces are different.

---

## Practical Considerations

- **Dimensionality** — Higher dimensions = more expressive but more storage/compute. 1536 is a good default.
- **Index size** — HNSW indexes can be 2–4x the raw vector data in memory. Plan accordingly.
- **Update frequency** — Some indexes (IVF) need rebuilding when data changes significantly. HNSW handles incremental updates better.
- **Cost** — Managed services charge per vector stored + per query. Self-hosted = infrastructure cost.
- **Latency** — Vector search is typically 10–50ms for millions of vectors. Fast enough for real-time RAG.

---

## Key Takeaways

1. Vector databases solve **semantic search** — finding content by meaning rather than keywords
2. They store, index, and search high-dimensional vectors produced by [[OpenAI API Deep Dive#Embeddings API|embedding models]]
3. **HNSW** is the most popular indexing algorithm — fast approximate nearest neighbor with high recall
4. **pgvector** is the easiest path if you already have PostgreSQL — add an extension and go
5. **Metadata filtering** combines vector search with traditional filters — essential for production systems
6. Always use the **same embedding model** for both indexing and querying
7. Vector databases are the storage layer that makes [[RAG - Retrieval Augmented Generation|RAG]] possible at scale
