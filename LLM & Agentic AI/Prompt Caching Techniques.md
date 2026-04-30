---
title: Prompt Caching Techniques
date: 2025-07-17
tags:
  - llm
  - caching
  - cost-optimization
  - performance
  - spring-ai
  - rag
aliases:
  - LLM Caching
  - Prompt Cache
  - KV Cache
---

# Prompt Caching Techniques

Every LLM API call costs money and takes time. When you build agents or [[RAG - Retrieval Augmented Generation|RAG]] pipelines, those calls multiply fast. Caching is how you fight back — turning repeated work into near-instant, near-free lookups. This guide covers **every layer** of caching available to you, from the model's internal mechanisms to application-level strategies you build yourself.

---

## 1. Why Cache? (The Cost Problem)

LLM APIs charge **per token** — both input and output. That sounds manageable until you consider how real applications work:

- **Agents** make 3-10+ LLM calls per user request (planning → tool calls → synthesis)
- **RAG** prepends 2,000-8,000 tokens of retrieved context to every prompt
- **Multi-turn chat** resends the entire conversation history each time

> [!danger] Cost Compounding
> A single RAG query might send 4,000 context tokens + 100 question tokens. Do that 10,000 times/day and you're sending **41 million input tokens/day** — most of it identical context.

### The Math: Without Cache vs With Cache

| Metric | Without Cache | With Cache |
|:-------|:-------------|:-----------|
| Requests/day | 10,000 | 10,000 |
| Avg input tokens/request | 4,100 | 4,100 |
| Total input tokens/day | 41,000,000 | 41,000,000 |
| Cached token ratio | 0% | ~75% (prefix + semantic) |
| Effective billable tokens | 41,000,000 | ~12,300,000 |
| GPT-4o input cost ($2.50/1M) | **$102.50/day** | **$30.75/day** |
| Monthly savings | — | **~$2,150/month** |

> [!tip] Rule of Thumb
> If your prompts share repeated structure (system prompts, context, few-shot examples), caching can cut input costs by ==50-90%==.

Latency matters too. A typical LLM call takes 500ms–5s. A cache hit returns in **5-50ms**. For user-facing apps, that's the difference between snappy and sluggish.

---

## 2. Types of Caching in LLM Systems

Caching in the LLM world happens at **three levels** — model-internal, provider-level, and application-level. Understanding all three lets you stack them for maximum savings.

### 2.1 KV-Cache (Internal Model Cache)

This is the cache **inside the model itself** during a single generation. You don't control it, but understanding it is key to appreciating prefix caching.

**What happens during generation:**
1. The transformer processes your prompt through its attention layers
2. Each layer produces **Key** and **Value** tensors for every token
3. When generating token N+1, the model needs the K/V pairs from **all** previous tokens
4. Without caching, it would recompute K/V for tokens 1…N every single time
5. The **KV-cache** stores these tensors so each new token only computes its own K/V

````mermaid
graph LR
    subgraph "Without KV-Cache"
        A1["Token 1"] -->|"Compute K,V"| B1["Token 2"]
        A1 -->|"Recompute K,V"| C1["Token 3"]
        B1 -->|"Recompute K,V"| C1
        A1 -->|"Recompute K,V"| D1["Token 4"]
        B1 -->|"Recompute K,V"| D1
        C1 -->|"Recompute K,V"| D1
    end

    subgraph "With KV-Cache"
        A2["Token 1"] -->|"Compute & Store K,V"| Cache["KV Cache"]
        B2["Token 2"] -->|"Compute & Store K,V"| Cache
        C2["Token 3"] -->|"Compute & Store K,V"| Cache
        Cache -->|"Read cached K,V"| D2["Token 4 ✅"]
    end
````

> [!info] Why This Matters to You
> KV-cache is why **prefix caching** works. If the inference engine already has K/V states for your system prompt from a previous request, it can skip recomputing them entirely.

### 2.2 Prefix Caching (Provider-Level)

The big idea: if many requests share the **same prefix** (system prompt + context + few-shot examples), the provider caches the computed KV states across requests. The model only processes **new tokens** after the cached prefix.

#### OpenAI Prompt Caching

OpenAI's caching is **automatic** — no code changes needed.

| Detail | Value |
|:-------|:------|
| Activation | Automatic after first request |
| Discount | ==50% off== cached input tokens |
| Min prefix length | 1,024 tokens |
| Cache TTL | 5–10 minutes of inactivity |
| Supported models | GPT-4o, GPT-4o-mini, o1, o1-mini |

**Cache-friendly prompt structure (Java — [[Spring AI Framework]]):**

```java
// ✅ GOOD: Static content first, dynamic content last
var systemPrompt = """
    You are a logistics assistant for a freight forwarding company.
    You have access to shipment tracking, rate calculation, and 
    customs documentation tools.
    
    Rules:
    1. Always verify shipment IDs before providing status.
    2. Quote rates in USD unless otherwise specified.
    3. Flag any shipments with customs holds.
    
    Here are example interactions:
    User: Track shipment SHP-12345
    Assistant: Shipment SHP-12345 is currently in transit...
    [... more few-shot examples totaling 1,500+ tokens ...]
    """; // ← This entire block gets cached after first call

var response = chatClient.prompt()
    .system(systemPrompt)          // Static — cached ✅
    .user(userQuestion)            // Dynamic — processed fresh
    .call()
    .content();
```

**How to verify caching is active (check response headers):**

```java
// OpenAI returns cache usage in the response
// usage.prompt_tokens_details.cached_tokens > 0 means cache hit
ChatResponse response = chatClient.prompt()
    .system(systemPrompt)
    .user("Track shipment SHP-99881")
    .call()
    .chatResponse();

log.info("Cached tokens: {}", 
    response.getMetadata().getUsage().get("cached_tokens"));
```

#### Anthropic Prompt Caching (`cache_control`)

Anthropic's approach is **explicit** — you tell the API which blocks to cache using `cache_control`. This gives you precise control but requires code changes.

| Detail | Value |
|:-------|:------|
| Activation | Explicit via `cache_control` annotation |
| Cache hit discount | ==90% off== input price (0.1×) |
| Cache write surcharge | 25% above normal input price |
| Cache TTL | 5 minutes (extended on each hit) |
| Min cacheable block | 1,024 tokens (Haiku: 2,048) |
| Max cache breakpoints | 4 per request |

**API request with caching (raw JSON):**

```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 1024,
  "system": [
    {
      "type": "text",
      "text": "You are a logistics domain expert. Here is the complete customs regulation handbook for US imports:\n\n[... 3,000 tokens of regulation text ...]",
      "cache_control": {"type": "ephemeral"}
    }
  ],
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "<shipment_manifest>\n[... 2,000 tokens of manifest data ...]\n</shipment_manifest>",
          "cache_control": {"type": "ephemeral"}
        },
        {
          "type": "text",
          "text": "Does this shipment comply with USDA regulations?"
        }
      ]
    }
  ]
}
```

**Response with cache metrics:**

```json
{
  "usage": {
    "input_tokens": 50,
    "cache_creation_input_tokens": 5000,
    "cache_read_input_tokens": 0
  }
}
```

On the **second** request with the same prefix:

```json
{
  "usage": {
    "input_tokens": 50,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 5000
  }
}
```

> [!success] Multi-Turn Conversations
> Anthropic caching shines for multi-turn conversations on the same document. Cache the document once, then ask 10 follow-up questions — each one reads from cache at 90% discount.

#### Google Gemini Context Caching

Gemini takes a different approach — you **explicitly create a cache object** with a configurable TTL, then reference it in subsequent requests.

| Detail | Value |
|:-------|:------|
| Activation | Explicit cache creation via API |
| Cost | Per-hour storage fee + reduced input token cost |
| TTL | Configurable (default 1 hour, max varies) |
| Min tokens | 32,768 |
| Best for | Very long documents queried repeatedly |

```python
import google.generativeai as genai

# Create a cached context (pays storage cost)
cache = genai.caching.CachedContent.create(
    model="gemini-1.5-flash-001",
    display_name="customs-regulations",
    system_instruction="You are a customs compliance expert.",
    contents=[regulations_document],  # Large document
    ttl=datetime.timedelta(hours=2),
)

# Use cached context (reduced input cost)
model = genai.GenerativeModel.from_cached_content(cached_content=cache)
response = model.generate_content("Is HS code 8471.30 correct for laptops?")
```

### 2.3 Prompt Structure for Maximum Cache Hits

The order of your prompt content **directly affects** cache hit rates. Providers match prefixes from the **beginning** of the prompt.

```
┌─────────────────────────────────────────────┐
│  ✅ CACHE-OPTIMIZED STRUCTURE               │
│                                              │
│  ┌───────────────────────────────────┐       │
│  │ System Prompt (static)       ████ │ ← Cached│
│  ├───────────────────────────────────┤       │
│  │ Few-Shot Examples (static)   ████ │ ← Cached│
│  ├───────────────────────────────────┤       │
│  │ Document Context (per-session)███ │ ← Cached│
│  ├───────────────────────────────────┤       │
│  │ User Question (dynamic)      ░░░ │ ← Fresh │
│  └───────────────────────────────────┘       │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│  ❌ CACHE-BUSTING STRUCTURE                  │
│                                              │
│  ┌───────────────────────────────────┐       │
│  │ System Prompt (static)       ████ │ ← Cached│
│  ├───────────────────────────────────┤       │
│  │ "Current time: 2025-07-17"   ░░░ │ ← BREAKS│
│  ├───────────────────────────────────┤       │
│  │ Few-Shot Examples (static)   ░░░ │ ← Miss! │
│  ├───────────────────────────────────┤       │
│  │ Document Context             ░░░ │ ← Miss! │
│  ├───────────────────────────────────┤       │
│  │ User Question                ░░░ │ ← Miss! │
│  └───────────────────────────────────┘       │
└─────────────────────────────────────────────┘
```

> [!warning] Golden Rules for Prefix Caching
> 1. Put **static** content **first** (system prompt, few-shot examples, document context)
> 2. Put **dynamic** content **last** (user question, current timestamp)
> 3. **Don't** change the order between requests
> 4. **Don't** insert dynamic content (timestamps, request IDs) in the middle of static content

---

## 3. Application-Level Caching

Provider-level caching saves on token processing costs. **Application-level caching** eliminates LLM calls entirely — the fastest and cheapest call is the one you never make.

### 3.1 Exact Match Cache

The simplest approach: hash the full prompt, store the response, return it on identical prompts.

**Spring Boot + Redis implementation ([[Spring AI Framework]]):**

```java
@Configuration
@EnableCaching
public class LlmCacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        var config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(24))
            .serializeValuesWith(
                SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer())
            );

        return RedisCacheManager.builder(factory)
            .withCacheConfiguration("llm-exact", config)
            .withCacheConfiguration("llm-embeddings",
                config.entryTtl(Duration.ofDays(7)))  // Embeddings rarely change
            .build();
    }
}
```

```java
@Service
public class CachedLlmService {

    private final ChatClient chatClient;

    @Cacheable(value = "llm-exact", 
               key = "T(java.util.Objects).hash(#prompt, #model, #temperature)")
    public String ask(String prompt, String model, double temperature) {
        return chatClient.prompt()
            .user(prompt)
            .call()
            .content();
    }

    @CacheEvict(value = "llm-exact", allEntries = true)
    public void clearCache() {
        log.info("LLM exact-match cache cleared");
    }
}
```

**Python + Redis implementation:**

```python
import hashlib, json, redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def cached_llm_call(prompt: str, model: str = "gpt-4o", ttl: int = 3600):
    cache_key = hashlib.sha256(
        json.dumps({"prompt": prompt, "model": model}, sort_keys=True).encode()
    ).hexdigest()
    
    cached = r.get(f"llm:{cache_key}")
    if cached:
        return json.loads(cached)  # Cache hit
    
    response = openai.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}]
    )
    result = response.choices[0].message.content
    
    r.setex(f"llm:{cache_key}", ttl, json.dumps(result))
    return result
```

| Pros | Cons |
|:-----|:-----|
| Simple to implement | Low hit rate (exact same question is rare) |
| Guaranteed correctness | Doesn't handle paraphrased questions |
| Fast lookup (O(1) hash) | Cache grows linearly with unique queries |

> [!tip] Best for
> FAQ bots, repeated analytics queries, status check endpoints — anywhere the same question appears verbatim.

### 3.2 Semantic Cache

Instead of matching on exact text, match on **meaning**. Two questions that mean the same thing get the same cached answer.

**How it works:**

```
┌──────────────┐    ┌───────────────┐    ┌────────────────┐
│ User Question │───▶│ Embed Query   │───▶│ Search Vector  │
│               │    │ (384-dim vec) │    │ Cache (cosine) │
└──────────────┘    └───────────────┘    └───────┬────────┘
                                                  │
                                    ┌─────────────┴─────────────┐
                                    │                           │
                              similarity ≥ 0.95           similarity < 0.95
                                    │                           │
                              Return cached               Call LLM, cache
                              response ✅                 result for next time
```

**Python implementation with FAISS:**

```python
import numpy as np
import faiss
from openai import OpenAI

client = OpenAI()

class SemanticCache:
    def __init__(self, threshold: float = 0.95, embedding_dim: int = 1536):
        self.threshold = threshold
        self.index = faiss.IndexFlatIP(embedding_dim)  # Inner product (cosine)
        self.responses: list[str] = []
        self.queries: list[str] = []
    
    def _embed(self, text: str) -> np.ndarray:
        resp = client.embeddings.create(model="text-embedding-3-small", input=text)
        vec = np.array(resp.data[0].embedding, dtype=np.float32)
        faiss.normalize_L2(vec.reshape(1, -1))  # Normalize for cosine sim
        return vec
    
    def get_or_generate(self, question: str, generate_fn) -> tuple[str, bool]:
        query_vec = self._embed(question).reshape(1, -1)
        
        if self.index.ntotal > 0:
            scores, indices = self.index.search(query_vec, 1)
            if scores[0][0] >= self.threshold:
                return self.responses[indices[0][0]], True  # Cache hit
        
        # Cache miss — generate and store
        response = generate_fn(question)
        self.index.add(query_vec)
        self.responses.append(response)
        self.queries.append(question)
        return response, False

# Usage
cache = SemanticCache(threshold=0.95)

answer, was_cached = cache.get_or_generate(
    "What's the transit time from Shanghai to Rotterdam?",
    lambda q: call_llm(q)
)
# Next time someone asks "How long does shipping take from Shanghai to Rotterdam?"
# → Cache hit! Same meaning, different words.
```

**Spring AI implementation pattern:**

```java
@Service
public class SemanticCacheService {

    private final EmbeddingModel embeddingModel;
    private final VectorStore vectorStore;
    private final ChatClient chatClient;

    private static final double SIMILARITY_THRESHOLD = 0.95;

    public String queryWithCache(String question) {
        // 1. Embed the question
        float[] queryEmbedding = embeddingModel
            .embed(question);

        // 2. Search for similar cached queries
        var results = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(question)
                .topK(1)
                .similarityThreshold(SIMILARITY_THRESHOLD)
                .build()
        );

        if (!results.isEmpty()) {
            log.info("Semantic cache HIT for: {}", question);
            return results.get(0).getMetadata().get("response").toString();
        }

        // 3. Cache miss — call LLM
        log.info("Semantic cache MISS for: {}", question);
        String response = chatClient.prompt()
            .user(question)
            .call()
            .content();

        // 4. Store in cache
        var doc = new Document(question, Map.of("response", response));
        vectorStore.add(List.of(doc));

        return response;
    }
}
```

**Tuning the similarity threshold:**

| Threshold | Hit Rate | Risk |
|:----------|:---------|:-----|
| 0.99 | Very low | Almost no false matches |
| ==0.95== | **Balanced** | **Good default** |
| 0.90 | High | May return wrong answers for similar-but-different questions |
| 0.85 | Very high | Dangerous — "track shipment" ≈ "cancel shipment" |

**Libraries:** GPTCache, [[LangChain Fundamentals|LangChain]] `CacheBackedEmbeddings`, Momento Vector Index

### 3.3 Response Fragment Cache

Don't just cache complete responses — cache **reusable pieces**.

**What to cache separately:**

| Fragment | Cache TTL | Why |
|:---------|:----------|:----|
| Document embeddings | Days–weeks | Embeddings don't change unless the doc changes |
| Retrieved RAG chunks | Hours | Same query retrieves same chunks |
| Tool/API results | Varies | Weather: 30min, exchange rates: 1hr, DB schemas: 1day |
| Entity extractions | Hours–days | "Extract entities from this contract" → reusable |

```java
// Cache embeddings separately — these are expensive to compute
@Cacheable(value = "llm-embeddings", key = "#documentId")
public float[] embedDocument(String documentId, String content) {
    return embeddingModel.embed(content);
}

// Cache tool results separately from LLM responses
@Cacheable(value = "tool-results", key = "#toolName + '_' + #argsHash")
public String executeToolCached(String toolName, String argsHash, 
                                 Supplier<String> toolCall) {
    return toolCall.get();
}
```

> [!example] RAG Fragment Caching
> Instead of caching "question → answer", cache:
> - `query → retrieved_chunks` (vector search result)
> - `chunk_id → embedding` (avoid re-embedding)
> - `(chunks + question) → answer` (final response)
> 
> This way, even if the question changes slightly, you still save on embedding and retrieval costs.

---

## 4. Multi-Layer Caching Strategy

The most effective approach combines all layers. Each layer catches what the previous one missed:

```
User Request
     │
     ▼
┌─────────────────────────────┐
│ Layer 1: Exact Match (Redis)│  ← 5ms, free
│ Hash(prompt) → response     │
└──────────────┬──────────────┘
               │ miss
               ▼
┌─────────────────────────────┐
│ Layer 2: Semantic Cache     │  ← 20ms, embedding cost only
│ Embed(query) → similar resp │
└──────────────┬──────────────┘
               │ miss
               ▼
┌─────────────────────────────┐
│ Layer 3: Prefix Cache       │  ← 200ms, 50-90% token discount
│ Provider caches KV states   │
└──────────────┬──────────────┘
               │ miss
               ▼
┌─────────────────────────────┐
│ Layer 4: Full LLM Call      │  ← 1-5s, full price
│ Cold call, no caching       │
└─────────────────────────────┘
```

| Layer | Latency | Cost | Hit Rate | Scope |
|:------|:--------|:-----|:---------|:------|
| Exact Match | ~5ms | Free (after first call) | 5-15% | Identical questions |
| Semantic Cache | ~20ms | Embedding cost only | 20-40% | Similar questions |
| Prefix Cache | ~200ms | 50-90% discount | 60-80% | Shared prompt prefix |
| Full LLM Call | 1-5s | Full price | 100% (fallback) | Everything else |

---

## 5. Cache Invalidation

> [!quote] The Two Hard Problems
> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

### Time-Based (TTL)

The simplest strategy. Set a TTL and let entries expire.

```java
// Different TTLs for different content types
@Bean
public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
    return RedisCacheManager.builder(factory)
        .withCacheConfiguration("llm-exact",
            config().entryTtl(Duration.ofHours(1)))       // Short: answers may evolve
        .withCacheConfiguration("llm-embeddings",
            config().entryTtl(Duration.ofDays(30)))        // Long: embeddings are stable
        .withCacheConfiguration("tool-results",
            config().entryTtl(Duration.ofMinutes(30)))     // Medium: external data changes
        .build();
}
```

### Event-Based Invalidation

Invalidate when the source data changes.

```java
@Service
public class DocumentUpdateListener {

    private final CacheManager cacheManager;

    @EventListener
    public void onDocumentUpdated(DocumentUpdatedEvent event) {
        // Clear cached embeddings for this document
        cacheManager.getCache("llm-embeddings")
            .evict(event.getDocumentId());
        
        // Clear any semantic cache entries that used this document
        cacheManager.getCache("rag-responses")
            .evict(event.getDocumentId());
        
        log.info("Invalidated caches for document: {}", event.getDocumentId());
    }
}
```

### Version-Based Invalidation

New model version? New prompt template? Flush relevant caches.

```java
// Include model version in cache key
@Cacheable(value = "llm-exact",
    key = "#prompt.hashCode() + '_' + @modelConfig.version")
public String ask(String prompt) {
    return chatClient.prompt().user(prompt).call().content();
}
```

> [!warning] The Hardest Problem
> Knowing ==when== cached answers become stale. A cached answer about "current shipping rates" was correct yesterday but wrong today. Use short TTLs for time-sensitive data and event-based invalidation when possible.

---

## 6. Implementation Patterns

### 6.1 Spring Boot Caching Layer (Full Implementation)

```java
@Configuration
@EnableCaching
public class LlmCacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        var defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .serializeValuesWith(
                SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer())
            )
            .disableCachingNullValues();

        return RedisCacheManager.builder(factory)
            .cacheDefaults(defaultConfig.entryTtl(Duration.ofHours(1)))
            .withCacheConfiguration("llm-exact",
                defaultConfig.entryTtl(Duration.ofHours(4)))
            .withCacheConfiguration("llm-embeddings",
                defaultConfig.entryTtl(Duration.ofDays(7)))
            .withCacheConfiguration("tool-results",
                defaultConfig.entryTtl(Duration.ofMinutes(30)))
            .build();
    }
}
```

```java
@Service
@Slf4j
public class CachedChatService {

    private final ChatClient chatClient;
    private final MeterRegistry meterRegistry;

    private final Counter cacheHits;
    private final Counter cacheMisses;
    private final Timer llmLatency;

    public CachedChatService(ChatClient.Builder builder, MeterRegistry registry) {
        this.chatClient = builder.build();
        this.meterRegistry = registry;
        this.cacheHits = Counter.builder("llm.cache.hits").register(registry);
        this.cacheMisses = Counter.builder("llm.cache.misses").register(registry);
        this.llmLatency = Timer.builder("llm.call.latency").register(registry);
    }

    @Cacheable(value = "llm-exact",
        key = "T(java.util.Objects).hash(#systemPrompt, #userMessage)")
    public String chat(String systemPrompt, String userMessage) {
        cacheMisses.increment();
        return llmLatency.record(() ->
            chatClient.prompt()
                .system(systemPrompt)
                .user(userMessage)
                .call()
                .content()
        );
    }

    public double getCacheHitRate() {
        double hits = cacheHits.count();
        double total = hits + cacheMisses.count();
        return total > 0 ? hits / total : 0.0;
    }
}
```

### 6.2 Python Caching with GPTCache

```python
from gptcache import cache
from gptcache.adapter import openai as cached_openai
from gptcache.embedding import Onnx
from gptcache.manager import CacheBase, VectorBase, get_data_manager
from gptcache.similarity_evaluation.distance import SearchDistanceEvaluation

# Initialize GPTCache with semantic matching
onnx = Onnx()
cache_base = CacheBase("sqlite")
vector_base = VectorBase("faiss", dimension=onnx.dimension)
data_manager = get_data_manager(cache_base, vector_base)

cache.init(
    embedding_func=onnx.to_embeddings,
    data_manager=data_manager,
    similarity_evaluation=SearchDistanceEvaluation(),
)
cache.set_openai_key()

# Now use the cached adapter — identical API, automatic caching
response = cached_openai.ChatCompletion.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "What is prompt caching?"}],
)
```

### 6.3 LangChain Built-in Caching

```python
from langchain.globals import set_llm_cache
from langchain_community.cache import RedisSemanticCache
from langchain_openai import OpenAIEmbeddings

# Semantic cache backed by Redis
set_llm_cache(
    RedisSemanticCache(
        redis_url="redis://localhost:6379",
        embedding=OpenAIEmbeddings(),
        score_threshold=0.95,
    )
)

# All LangChain LLM calls now automatically use the cache
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o")
llm.invoke("What are the INCOTERMS for CIF shipping?")  # Cached automatically
```

### 6.4 Monitoring Cache Performance

Track these metrics to know if your caching strategy is working:

| Metric | Target | Alert Threshold |
|:-------|:-------|:----------------|
| Exact cache hit rate | >10% | <5% |
| Semantic cache hit rate | >25% | <15% |
| Prefix cache hit rate | >60% | <40% |
| Avg latency (cached) | <50ms | >200ms |
| Avg latency (uncached) | <3s | >5s |
| Monthly cost savings | >40% | <20% |

```java
// Expose cache metrics via Spring Actuator
@RestController
@RequestMapping("/api/cache")
public class CacheMetricsController {

    private final CachedChatService chatService;

    @GetMapping("/stats")
    public Map<String, Object> getStats() {
        return Map.of(
            "hitRate", chatService.getCacheHitRate(),
            "estimatedSavings", chatService.getEstimatedCostSavings(),
            "cacheSize", chatService.getCacheEntryCount()
        );
    }
}
```

---

## 7. Cost Savings Analysis

### Real-World Calculation: RAG Customer Support Bot

**Assumptions:**
- 10,000 queries/day
- System prompt: 500 tokens
- RAG context: 3,000 tokens per query
- User question: 100 tokens
- Output: 400 tokens per response
- Model: GPT-4o ($2.50/1M input, $10.00/1M output)

**Without any caching:**

| Component | Tokens/Day | Cost/Day |
|:----------|:-----------|:---------|
| Input tokens | 36,000,000 | $90.00 |
| Output tokens | 4,000,000 | $40.00 |
| **Daily total** | | **$130.00** |
| **Monthly total** | | **$3,900** |

**With multi-layer caching:**

| Layer | Effect | Savings |
|:------|:-------|:--------|
| Exact match (10% hit rate) | 1,000 calls eliminated | $13.00/day |
| Semantic cache (25% of remaining) | 2,250 calls eliminated | $29.25/day |
| Prefix cache on remaining 6,750 calls (50% discount on 3,500 cached tokens) | Reduced input cost | $29.53/day |
| **Daily savings** | | **$71.78** |
| **Monthly savings** | | **~$2,153** |
| **Effective monthly cost** | | **~$1,747 (55% reduction)** |

### When Caching ISN'T Worth It

> [!warning] Skip Caching When
> - **Unique queries**: Every question is genuinely different (creative writing, brainstorming)
> - **Constantly changing context**: Source data updates every few minutes
> - **Low volume**: <100 calls/day — infrastructure cost exceeds savings
> - **High-stakes accuracy**: Medical/legal where stale answers are dangerous
> - **Temperature > 0 intentionally**: You *want* varied responses

---

## 8. Common Pitfalls

> [!danger] Pitfall 1: Caching Non-Deterministic Outputs
> If `temperature > 0`, the same prompt produces different outputs. Caching locks in one version — the user always gets the same "creative" response. **Fix:** Set `temperature = 0` for cached prompts, or only cache factual/analytical responses.

> [!danger] Pitfall 2: Stale Cache Serving Outdated Information
> "What's the current exchange rate?" cached 6 hours ago is wrong now. **Fix:** Use short TTLs for time-sensitive queries. Tag cache entries with data freshness timestamps.

> [!danger] Pitfall 3: Cache Poisoning
> A hallucinated or incorrect response gets cached and served to every subsequent user. **Fix:** Add quality checks before caching. Never cache responses that the LLM flagged as uncertain.

> [!danger] Pitfall 4: Over-Caching
> Your Redis instance grows to 50GB storing every LLM response ever generated. **Fix:** Set max memory limits, use LRU eviction, and only cache high-frequency queries.

> [!danger] Pitfall 5: Breaking Prefix Cache
> Inserting `"Current timestamp: 2025-07-17T14:30:00"` at the top of your system prompt breaks the prefix match for every single request. **Fix:** Move all dynamic content to the end of the prompt. See [[#2.3 Prompt Structure for Maximum Cache Hits]].

---

## Key Takeaways

1. **LLM caching operates at three levels** — model-internal (KV-cache), provider-level (prefix caching), and application-level (exact/semantic). Stack all three for maximum savings.
2. **Prompt structure matters** — put static content first, dynamic content last. A reordered prompt busts the prefix cache and wastes money.
3. **Anthropic gives you explicit control** with `cache_control` (90% discount on hits). OpenAI is automatic (50% discount). Choose your provider's caching model wisely.
4. **Semantic caching eliminates LLM calls entirely** by matching on meaning, not exact text. A similarity threshold of 0.95 is a safe starting point.
5. **Cache fragments, not just full responses** — embeddings, retrieved chunks, and tool results all benefit from independent caching with different TTLs.
6. **Invalidation is the hardest part** — use TTL as a baseline, event-driven invalidation for known data changes, and version keys for model upgrades.
7. **Monitor your cache hit rates** — if exact match is below 5% or semantic below 15%, your caching strategy needs tuning.
8. **A multi-layer cache can cut LLM costs by 40-60%** for typical [[RAG in Production|production RAG]] applications — but only if your queries have enough repetition to benefit.
9. **Don't cache everything** — skip caching for unique queries, time-sensitive data, and intentionally non-deterministic outputs.
10. **Start simple** — begin with exact match caching in Redis, add semantic caching when hit rates plateau, and let prefix caching work automatically in the background.

---

**Related:** [[OpenAI API Deep Dive]] · [[Spring AI Framework]] · [[LangChain Fundamentals]] · [[RAG - Retrieval Augmented Generation]] · [[RAG in Production]] · [[Production Considerations]]
