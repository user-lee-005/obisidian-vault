---
title: RAG in Production
date: 2025-07-14
tags:
  - rag
  - production
  - mlops
  - architecture
  - observability
aliases:
  - Production RAG
  - RAG Ops
cssclasses: []
---

# RAG in Production

> [!abstract] TL;DR
> You built a RAG pipeline that works on your laptop. Now ship it. This note covers architecture, scaling, multi-tenancy, evaluation, cost, observability, and security for production RAG systems.

**Prerequisites:** [[RAG - Retrieval Augmented Generation]], [[RAG with Spring AI]], [[RAG with LangChain]], [[Vector Databases]]
**Related:** [[Advanced RAG Patterns]], [[Production Considerations]]

---

## 1. Production Architecture

A production RAG system splits into two independent pipelines: **ingestion** (offline, async) and **query** (online, sync).

```
┌─────────────────────── INGESTION PIPELINE (async) ──────────────────────┐
│                                                                         │
│  Documents ──▶ Kafka/RabbitMQ ──▶ Chunker ──▶ Embedder ──▶ Vector DB   │
│  (S3/GCS)       (message queue)   (workers)   (batch API)  (Qdrant/    │
│                                                              Pinecone)  │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────── QUERY PIPELINE (sync) ───────────────────────────┐
│                                                                         │
│  User ──▶ Load Balancer ──▶ API Server ──┬──▶ Redis Cache              │
│           (nginx/ALB)      (stateless)   ├──▶ Vector DB (read replica) │
│                                          └──▶ LLM API (OpenAI/Bedrock)│
└─────────────────────────────────────────────────────────────────────────┘
```

> [!tip] Key Principle
> Keep ingestion and query **decoupled**. Ingestion is bursty, CPU-heavy, and tolerates latency. Query is latency-sensitive and must stay fast under load.

### Minimum viable deployment

```yaml
# docker-compose.yml (simplified)
services:
  api:
    image: rag-query-service:latest
    deploy:
      replicas: 3
    environment:
      VECTOR_DB_URL: http://qdrant:6333
      REDIS_URL: redis://cache:6379
      OPENAI_API_KEY: ${OPENAI_API_KEY}
  ingestion-worker:
    image: rag-ingestion:latest
    deploy:
      replicas: 2
    environment:
      KAFKA_BROKERS: kafka:9092
      VECTOR_DB_URL: http://qdrant:6333
  qdrant:
    image: qdrant/qdrant:latest
    volumes:
      - qdrant_data:/qdrant/storage
  cache:
    image: redis:7-alpine
```

---

## 2. Scaling the Ingestion Pipeline

### Batch vs streaming

| Approach | Use when | Tool |
|----------|----------|------|
| **Batch** | Nightly bulk loads, initial migration | Cron + Spring Batch / Airflow |
| **Streaming** | New docs arrive continuously | Kafka consumer group |
| **Hybrid** | Bulk load + incremental updates | Batch bootstrap, then stream |

### Parallel chunking and embedding

```java
// Spring AI — parallel embedding with virtual threads
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
List<Future<List<Float>>> futures = chunks.stream()
    .map(chunk -> executor.submit(() -> embeddingModel.embed(chunk)))
    .toList();
```

### Incremental updates & versioning

- Store a `doc_hash` (SHA-256 of content) alongside each chunk in metadata.
- On re-ingest: hash the new version → compare → skip unchanged docs.
- **Deletes:** delete all chunks where `source_doc_id = X` before re-inserting.
- **Deduplication:** compute `chunk_hash` and check before insert — reject exact duplicates.

### Scheduling

```yaml
# Kubernetes CronJob for nightly re-index
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rag-reindex
spec:
  schedule: "0 2 * * *"   # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: reindex
              image: rag-ingestion:latest
              command: ["java", "-jar", "ingestion.jar", "--mode=incremental"]
```

---

## 3. Scaling the Query Pipeline

### Caching strategies

**Exact match cache** — hash the query string, return cached response.

```python
# Redis exact-match cache
cache_key = hashlib.sha256(query.encode()).hexdigest()
cached = redis.get(f"rag:exact:{cache_key}")
if cached:
    return json.loads(cached)
```

**Semantic cache** — embed the query, search a cache index for similar past queries (cosine > 0.95).

```
Query ──▶ Embed ──▶ Search cache collection ──▶ Hit? Return cached answer
                                               ──▶ Miss? Run full pipeline, store result
```

**TTL & invalidation:**
- Set TTL based on content freshness (e.g., 1h for news, 24h for docs).
- Invalidate on ingestion events: when source docs change, flush related cache keys.

### Connection pooling & read replicas

```yaml
# application.yml — connection pool to Qdrant
vector-db:
  url: http://qdrant-read-replica:6333
  pool:
    max-connections: 50
    idle-timeout: 30s
```

### Horizontal scaling

Query servers are **stateless** — scale horizontally behind a load balancer. No sticky sessions needed. See [[System Design]] concepts: load balancing, horizontal scaling.

### Rate limiting

```yaml
# Per-tenant rate limits (env vars or config)
RATE_LIMIT_DEFAULT: 100    # requests/min
RATE_LIMIT_PREMIUM: 500
RATE_LIMIT_WINDOW: 60s
```

---

## 4. Multi-Tenancy

### Data isolation approaches

| Approach | Isolation | Cost | Complexity |
|----------|-----------|------|------------|
| **Metadata filter** (`tenant_id` on every chunk) | Low | Low | Low |
| **Separate collection/namespace** per tenant | Medium | Medium | Medium |
| **Separate vector DB instance** per tenant | High | High | High |

> [!warning] Metadata filtering is the cheapest but the riskiest
> A bug in your filter logic leaks data across tenants. Always add integration tests that assert tenant isolation.

### Access control at query time

```java
// Spring AI — inject tenant filter into every retrieval
SearchRequest request = SearchRequest.query(userQuery)
    .withFilterExpression("tenant_id == '" + currentTenant + "'")
    .withTopK(5);
```

### Cost allocation

Tag each LLM/embedding API call with `tenant_id`. Aggregate usage in your billing system. Most LLM providers don't support this natively — you'll need a proxy or wrapper that logs per-tenant token counts.

---

## 5. Chunking at Scale

### Strategy by document type

| Document Type | Strategy | Chunk Size |
|---------------|----------|------------|
| Legal contracts | Section/clause-based splitting | ~1000 tokens |
| Source code | Function/class-level splitting | Entire function |
| API docs | Endpoint-level (one chunk per endpoint) | Variable |
| Chat transcripts | Turn-based (user + assistant pair) | ~200-500 tokens |
| Long-form articles | Paragraph-based with overlap | ~500 tokens, 50 overlap |

### Metadata enrichment

Attach metadata **at chunk time** — you can't retroactively add it without re-indexing.

```json
{
  "content": "Section 4.2: Liability shall not exceed...",
  "metadata": {
    "source": "contract_2024_acme.pdf",
    "page": 12,
    "section": "4.2 Liability",
    "author": "Legal Team",
    "created_at": "2024-06-15",
    "tenant_id": "acme-corp",
    "doc_type": "legal",
    "tags": ["liability", "terms"]
  }
}
```

### Chunk quality validation

- **Min/max length gates:** reject chunks < 20 tokens or > 2000 tokens.
- **Language detection:** flag chunks that are mostly gibberish (OCR artifacts, binary content).
- **Entropy check:** extremely low-entropy chunks (repeated text) are usually useless — drop them.

---

## 6. Index Management

### When to re-index

| Trigger | Action |
|---------|--------|
| Embedding model changed | **Full re-index** — all vectors are incompatible |
| Chunk strategy changed | **Full re-index** — chunk boundaries shifted |
| New documents added | **Incremental** — add new, skip existing |
| Documents updated | **Partial** — delete old chunks, insert new |
| Documents deleted | **Partial** — delete matching chunks |

### Blue-green index deployment

```
1. Build new index in collection "docs_v2" (while "docs_v1" serves traffic)
2. Run validation queries against "docs_v2"
3. Swap alias: "docs_live" → "docs_v2"
4. Keep "docs_v1" for 24h rollback window
5. Delete "docs_v1"
```

This is the same blue-green pattern from [[Production Considerations]] — applied to vector indexes.

### Monitoring & cleanup

```yaml
# Alerts (Prometheus + Grafana)
- alert: VectorIndexTooLarge
  expr: qdrant_collection_points_count > 5000000
  labels:
    severity: warning
- alert: StaleDocumentsDetected
  expr: rag_docs_older_than_90d_count > 1000
  labels:
    severity: info
```

**Backup:** Qdrant and most vector DBs support snapshots. Schedule daily snapshots to object storage.

---

## 7. Evaluation in Production

### Online evaluation (real users)

- **Thumbs up/down** on every answer — simplest signal, highest volume.
- **Source click-through** — if users click cited sources, retrieval was relevant.
- **Follow-up rate** — if the user immediately asks a follow-up, the first answer was likely insufficient.

### Offline evaluation (automated)

- **Golden test set:** 50-200 curated `(question, expected_answer, expected_sources)` tuples.
- **Regression tests:** run the golden set on every pipeline change in CI.
- **A/B testing:** split traffic between retrieval strategies, compare metrics.

### Metrics dashboard

| Metric | Target | How to measure |
|--------|--------|----------------|
| Retrieval latency (p95) | < 200ms | OpenTelemetry span |
| LLM generation latency (p95) | < 3s | OpenTelemetry span |
| Cache hit rate | > 30% | Redis `INFO stats` |
| Faithfulness score | > 0.8 | LLM-as-judge (automated) |
| User satisfaction | > 80% positive | Thumbs up/down ratio |

> [!example] LLM-as-Judge for Faithfulness
> Send `(context, answer)` to a judge LLM with prompt: *"Does the answer contain only information supported by the context? Score 0-1."* Run nightly on a sample of production queries.

---

## 8. Cost Optimization

### Embedding costs

- **Batch API calls** — most providers offer 50%+ discount for batch.
- **Cache embeddings** — if the same text is embedded twice, serve from cache.
- **Smaller models for drafts** — use `text-embedding-3-small` during dev, `text-embedding-3-large` in prod.

### LLM costs

- **Shorter prompts** — trim system instructions, compress retrieved chunks.
- **Fewer chunks** — retrieve 3-5, not 10-20. More chunks ≠ better answers.
- **Tiered models** — route simple queries to `gpt-4o-mini`, complex to `gpt-4o`.

### Adaptive retrieval

Not every query needs RAG. Use a classifier or keyword check:

```
User: "What's the weather?"  → No retrieval needed, respond directly
User: "What's our refund policy?" → RAG retrieval needed
```

### Token budget

```python
MAX_CONTEXT_TOKENS = 4000
chunks = retrieve(query, top_k=10)
selected = []
token_count = 0
for chunk in chunks:
    if token_count + len(chunk.tokens) > MAX_CONTEXT_TOKENS:
        break
    selected.append(chunk)
    token_count += len(chunk.tokens)
```

---

## 9. Observability & Debugging

### End-to-end trace

Every RAG request should produce a trace like:

```
[trace_id: abc-123]
├── query_received: "How do I cancel a shipment?"
├── query_reformulated: "shipment cancellation process policy"
├── retrieval:
│   ├── vector_search: 5 chunks, 45ms
│   ├── chunk_ids: [c1, c2, c3, c4, c5]
│   └── top_score: 0.89
├── prompt_assembled: 1847 tokens
├── llm_call: gpt-4o, 2.1s, 312 output tokens
└── response_returned: 200 OK
```

**Tools:** LangSmith, OpenTelemetry + Jaeger, or custom structured logging.

### Common failure modes

| Failure | Symptom | Fix |
|---------|---------|-----|
| **Retrieval miss** | Relevant doc exists but wasn't retrieved | Improve chunking, add query expansion |
| **Context pollution** | Irrelevant chunks confuse the LLM | Raise similarity threshold, use reranker |
| **Hallucination** | Model ignores retrieved context | Strengthen system prompt, use citations |
| **Stale data** | Answer from outdated document | Enforce TTL, add `last_updated` metadata |

### Debugging workflow

```
1. Reproduce: get the exact query that failed
2. Trace: pull the trace — check retrieval scores
3. Isolate: is it a retrieval problem or a generation problem?
4. Fix: adjust chunking/embeddings (retrieval) or prompt (generation)
5. Validate: add to golden test set, run regression
```

---

## 10. Security & Compliance

### PII handling

- **Detection:** scan documents before indexing (regex for emails/phones, NER for names).
- **Redaction:** replace PII with tokens (`[REDACTED_EMAIL]`) or hash them.
- **Separation:** store PII in encrypted document store, not in vector DB chunks.

### Access control enforcement

```java
// CRITICAL: always filter at retrieval time, not after
// Never retrieve all chunks and filter in application code
SearchRequest request = SearchRequest.query(query)
    .withFilterExpression(
        "tenant_id == '" + tenantId + "' AND " +
        "access_level <= " + userAccessLevel
    );
```

### Audit logging

```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "user_id": "usr_abc",
  "tenant_id": "acme-corp",
  "query": "What is the termination clause?",
  "retrieved_docs": ["contract_2024.pdf#section4"],
  "model": "gpt-4o",
  "tokens_used": 2159,
  "trace_id": "abc-123"
}
```

### Data retention & GDPR

- **Auto-delete:** schedule cleanup jobs for chunks older than retention period.
- **Right to deletion:** when a user requests deletion, remove all their documents + chunks + cached answers. Log the deletion event for compliance.

```sql
-- Deletion query for vector DB (pseudo-SQL, actual API varies)
DELETE FROM chunks WHERE tenant_id = 'acme' AND source_doc IN (
  SELECT doc_id FROM deletion_requests WHERE status = 'pending'
);
```

---

## Key Takeaways

1. **Split ingestion and query pipelines** — they have fundamentally different scaling and latency requirements.
2. **Cache aggressively** — exact match for identical queries, semantic cache for similar ones. Aim for >30% hit rate.
3. **Metadata is everything** — enrich chunks at ingestion time with source, tenant, dates, and tags. You can't add it later without re-indexing.
4. **Multi-tenancy is a day-one decision** — retrofitting tenant isolation is painful. Pick your isolation level early.
5. **Evaluate continuously** — golden test sets in CI, user feedback in prod, LLM-as-judge for faithfulness. No metrics = flying blind.
6. **Control costs with adaptive retrieval** — not every query needs RAG. Route simple queries to smaller models.
7. **Trace every request end-to-end** — when something goes wrong, you need to see query → retrieval → prompt → response in one view.
8. **Security is a retrieval-time concern** — filter at the vector DB level, not in application code. Never retrieve-then-filter.
9. **Blue-green your indexes** — treat index updates like deployments. Build new, validate, swap, keep rollback window.
10. **Plan for deletion from day one** — GDPR right-to-deletion means you need to find and remove a user's data across vector stores, caches, and logs.
