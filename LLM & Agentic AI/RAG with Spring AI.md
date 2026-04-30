---
title: RAG with Spring AI
date: 2025-07-18
tags:
  - spring-ai
  - rag
  - java
  - vector-database
  - llm
aliases:
  - Spring AI RAG
  - RAG Pipeline Java
status: complete
---

# RAG with Spring AI

You understand [[RAG - Retrieval Augmented Generation|what RAG is and why it matters]]. Now let's **build one** end to end in Java using [[Spring AI Framework]]. By the time you finish this note you'll have a working ingestion pipeline, a pgvector-backed [[Vector Databases|vector store]], and a streaming REST API — all inside a standard Spring Boot app.

---

## 1 · Project Setup

### Maven Dependencies

```xml
<properties>
    <java.version>21</java.version>
    <spring-ai.version>1.0.0</spring-ai.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>${spring-ai.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Core Spring AI + OpenAI -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-openai</artifactId>
    </dependency>

    <!-- PgVector store -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
    </dependency>

    <!-- Document readers (PDF, Word, etc.) -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-tika-document-reader</artifactId>
    </dependency>

    <!-- Testcontainers for integration tests -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-testcontainers</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### application.yml

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
          temperature: 0.2
      embedding:
        options:
          model: text-embedding-3-small
    vectorstore:
      pgvector:
        initialize-schema: true          # auto-create table on startup
        index-type: hnsw                  # fast approximate nearest neighbour
        distance-type: cosine_distance
        dimensions: 1536                  # must match embedding model
  datasource:
    url: jdbc:postgresql://localhost:5432/ragdb
    username: postgres
    password: postgres
```

### Docker Compose — pgvector

```yaml
# compose.yml
services:
  pgvector:
    image: pgvector/pgvector:pg16
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: ragdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

Run `docker compose up -d` and you're ready.

---

## 2 · The ETL Pipeline (Document Ingestion)

Spring AI models ingestion as an **ETL flow**: `DocumentReader → DocumentTransformer → DocumentWriter`.

### DocumentReader Implementations

```java
// PDF / Word / HTML — Apache Tika handles them all
var tikaReader = new TikaDocumentReader(
        new ClassPathResource("docs/shipping-policy.pdf"));
List<Document> docs = tikaReader.get();

// Plain text
var textReader = new TextReader(new ClassPathResource("docs/faq.txt"));
textReader.getCustomMetadata().put("source", "faq");
List<Document> textDocs = textReader.get();

// JSON — each element becomes a Document
var jsonReader = new JsonReader(
        new ClassPathResource("docs/products.json"),
        "name", "description");           // keys to extract
List<Document> jsonDocs = jsonReader.get();
```

#### Custom Reader (e.g. read from your own API)

```java
@Component
public class ShipmentApiReader implements DocumentReader {

    private final ShipmentClient client;

    public ShipmentApiReader(ShipmentClient client) {
        this.client = client;
    }

    @Override
    public List<Document> get() {
        return client.fetchAll().stream()
                .map(s -> new Document(
                        s.summary(),
                        Map.of("shipmentId", s.id(), "carrier", s.carrier())))
                .toList();
    }
}
```

### DocumentTransformer — Text Splitting

```java
// Split by token count — best general-purpose splitter
var splitter = new TokenTextSplitter(
        800,    // defaultChunkSize (tokens)
        350,    // minChunkSizeChars
        200,    // minChunkLengthToEmbed
        100,    // maxNumChunks
        true    // keepSeparator
);
List<Document> chunks = splitter.apply(docs);
```

#### Custom Transformer — Metadata Enrichment

```java
@Component
public class MetadataEnricher implements DocumentTransformer {

    @Override
    public List<Document> apply(List<Document> documents) {
        documents.forEach(doc -> {
            doc.getMetadata().put("ingestedAt", Instant.now().toString());
            doc.getMetadata().put("charCount", doc.getText().length());
        });
        return documents;
    }
}
```

### Full Ingestion Service

```java
@Service
public class DocumentIngestionService {

    private final VectorStore vectorStore;
    private final TokenTextSplitter splitter;
    private final MetadataEnricher enricher;

    public DocumentIngestionService(VectorStore vectorStore,
                                    MetadataEnricher enricher) {
        this.vectorStore = vectorStore;
        this.enricher = enricher;
        this.splitter = new TokenTextSplitter();
    }

    /** Ingest an uploaded file (PDF, DOCX, TXT). */
    public int ingest(Resource resource) {
        var reader = new TikaDocumentReader(resource);
        List<Document> raw = reader.get();
        List<Document> chunks = splitter.apply(raw);
        List<Document> enriched = enricher.apply(chunks);

        vectorStore.add(enriched);                 // embeds + stores
        return enriched.size();
    }

    /** Batch ingest — useful for bootstrap / re-index jobs. */
    public int ingestAll(List<Resource> resources) {
        return resources.stream()
                .mapToInt(this::ingest)
                .sum();
    }
}
```

> [!tip] Batch Size
> PgVectorStore sends one INSERT per `add()` call. For very large corpora, batch your documents in groups of ~200 to avoid oversized transactions.

---

## 3 · Embedding Models

### OpenAI Embeddings

| Model | Dimensions | Cost | Best For |
|-------|-----------|------|----------|
| `text-embedding-3-small` | 1536 | Cheapest | Most apps — good accuracy/cost balance |
| `text-embedding-3-large` | 3072 | 6× more | When precision really matters |
| `text-embedding-ada-002` | 1536 | Legacy | Don't use for new projects |

Spring AI auto-configures `OpenAiEmbeddingModel` from `application.yml`.

### Ollama Local Embeddings

If you want to keep everything on-prem, swap the starter:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-ollama</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      embedding:
        options:
          model: nomic-embed-text        # 768 dimensions
```

> [!warning] Dimension Mismatch
> If you switch embedding models you **must re-index** all documents. Vectors from different models live in different spaces and can't be compared.

---

## 4 · Vector Store Integration

### PgVectorStore — Schema

With `initialize-schema: true`, Spring AI creates this for you:

```sql
CREATE TABLE IF NOT EXISTS vector_store (
    id        UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    content   TEXT,
    metadata  JSON,
    embedding VECTOR(1536)     -- dimension matches your model
);

CREATE INDEX ON vector_store
    USING hnsw (embedding vector_cosine_ops);
```

### Similarity Search with Metadata Filters

```java
// Simple search — top 5 nearest neighbours
List<Document> results = vectorStore.similaritySearch(
        SearchRequest.builder()
                .query("What is the cancellation policy?")
                .topK(5)
                .build()
);

// Filtered search — only docs from a specific source
List<Document> filtered = vectorStore.similaritySearch(
        SearchRequest.builder()
                .query("shipping deadlines")
                .topK(5)
                .similarityThreshold(0.75)
                .filterExpression("source == 'faq'")
                .build()
);
```

The filter expression language supports `==`, `!=`, `>`, `<`, `in`, `nin`, `&&`, `||`.

### Other Vector Store Options (Quick Config Comparison)

| Store | Starter Artifact | Managed | Notes |
|-------|-----------------|---------|-------|
| **PgVector** | `spring-ai-starter-vector-store-pgvector` | Self-host / RDS | Best if you already run Postgres |
| **ChromaDB** | `spring-ai-starter-vector-store-chroma` | Self-host | Good for prototyping |
| **Milvus** | `spring-ai-starter-vector-store-milvus` | Zilliz Cloud | High-scale workloads |
| **Redis** | `spring-ai-starter-vector-store-redis` | Redis Cloud | If you already have Redis Stack |
| **Pinecone** | `spring-ai-starter-vector-store-pinecone` | Fully managed | Zero-ops, pay-per-query |

All implement the same `VectorStore` interface — swap the dependency and config, the rest of your code stays identical.

---

## 5 · The RAG Query Pipeline

### QuestionAnswerAdvisor — Quickest Path

Spring AI ships a built-in RAG advisor. One line wires retrieval into your chat:

```java
@Service
public class SimpleRagService {

    private final ChatClient chatClient;

    public SimpleRagService(ChatClient.Builder builder, VectorStore vectorStore) {
        this.chatClient = builder
                .defaultAdvisors(new QuestionAnswerAdvisor(vectorStore,
                        SearchRequest.builder().topK(5).build()))
                .build();
    }

    public String ask(String question) {
        return chatClient.prompt()
                .user(question)
                .call()
                .content();
    }
}
```

Under the hood, `QuestionAnswerAdvisor` embeds the question, retrieves top-K documents, and appends them to the system prompt before calling the LLM.

### Custom RAG Chain — Full Control

```java
@Service
public class RagService {

    private final VectorStore vectorStore;
    private final ChatClient chatClient;

    private static final String SYSTEM_PROMPT = """
            You are a helpful assistant. Answer the user's question using ONLY
            the provided context. If the context doesn't contain the answer,
            say "I don't have enough information."

            Context:
            {context}
            """;

    public RagService(VectorStore vectorStore, ChatClient.Builder builder) {
        this.vectorStore = vectorStore;
        this.chatClient = builder.build();
    }

    public String ask(String question) {
        // 1. Retrieve relevant chunks
        List<Document> docs = vectorStore.similaritySearch(
                SearchRequest.builder()
                        .query(question)
                        .topK(5)
                        .similarityThreshold(0.7)
                        .build());

        // 2. Build context string
        String context = docs.stream()
                .map(Document::getText)
                .collect(Collectors.joining("\n\n---\n\n"));

        // 3. Augment prompt & generate
        return chatClient.prompt()
                .system(s -> s.text(SYSTEM_PROMPT).param("context", context))
                .user(question)
                .call()
                .content();
    }
}
```

### Streaming RAG Responses

```java
public Flux<String> askStream(String question) {
    List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.builder().query(question).topK(5).build());

    String context = docs.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n\n---\n\n"));

    return chatClient.prompt()
            .system(s -> s.text(SYSTEM_PROMPT).param("context", context))
            .user(question)
            .stream()
            .content();                  // returns Flux<String>
}
```

### Conversational RAG (Chat History + Retrieval)

```java
@Service
public class ConversationalRagService {

    private final RagService ragService;
    private final ChatMemory chatMemory;

    public ConversationalRagService(RagService ragService) {
        this.ragService = ragService;
        this.chatMemory = new InMemoryChatMemory();
    }

    public String chat(String sessionId, String userMessage) {
        chatMemory.add(sessionId, new UserMessage(userMessage));

        String answer = ragService.ask(userMessage);

        chatMemory.add(sessionId, new AssistantMessage(answer));
        return answer;
    }
}
```

---

## 6 · Advanced Patterns

### Custom Similarity Threshold

Reject low-confidence retrievals before they pollute the prompt:

```java
List<Document> docs = vectorStore.similaritySearch(
        SearchRequest.builder()
                .query(question)
                .topK(10)
                .similarityThreshold(0.78)       // drop anything below 0.78
                .build());

if (docs.isEmpty()) {
    return "Sorry, I couldn't find relevant information for that question.";
}
```

### Re-ranking with a Second LLM Call

Retrieve broadly, then ask the LLM to pick the most relevant chunks:

```java
public List<Document> rerank(String question, List<Document> candidates) {
    String prompt = """
            Given the question: "%s"
            Rank these passages by relevance (most relevant first).
            Return ONLY the passage numbers as a comma-separated list.
            %s
            """.formatted(question, formatNumbered(candidates));

    String ranking = chatClient.prompt().user(prompt).call().content();

    List<Integer> order = Arrays.stream(ranking.split(","))
            .map(String::trim)
            .map(Integer::parseInt)
            .toList();

    return order.stream()
            .filter(i -> i >= 0 && i < candidates.size())
            .map(candidates::get)
            .toList();
}
```

### Multi-tenant RAG via Metadata Filters

Store a `tenantId` during ingestion, then filter at query time:

```java
public String askForTenant(String tenantId, String question) {
    List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.builder()
                    .query(question)
                    .topK(5)
                    .filterExpression("tenantId == '%s'".formatted(tenantId))
                    .build());
    // ... build prompt and call LLM
}
```

### Multi-index Routing

Route questions to different vector stores based on intent:

```java
@Service
public class RoutingRagService {

    private final Map<String, VectorStore> stores; // "policies", "products"
    private final ChatClient chatClient;

    public String ask(String question) {
        String category = classify(question);        // use LLM or keyword match
        VectorStore store = stores.getOrDefault(category, stores.get("policies"));
        List<Document> docs = store.similaritySearch(
                SearchRequest.builder().query(question).topK(5).build());
        // ... augment and generate
    }
}
```

See [[Advanced RAG Patterns]] for deeper dives on query rewriting, hierarchical retrieval, and graph-augmented RAG.

---

## 7 · REST API Layer

```java
public record QuestionRequest(String question, String sessionId) {}

@RestController
@RequestMapping("/api/rag")
public class RagController {

    private final RagService ragService;
    private final DocumentIngestionService ingestionService;

    public RagController(RagService ragService,
                         DocumentIngestionService ingestionService) {
        this.ragService = ragService;
        this.ingestionService = ingestionService;
    }

    @PostMapping("/ask")
    public Flux<String> ask(@RequestBody QuestionRequest request) {
        return ragService.askStream(request.question());
    }

    @PostMapping(value = "/ingest", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<Map<String, Object>> ingest(
            @RequestParam("file") MultipartFile file) throws IOException {

        var resource = new InputStreamResource(file.getInputStream());
        int chunks = ingestionService.ingest(resource);

        return ResponseEntity.ok(Map.of(
                "status", "ingested",
                "fileName", file.getOriginalFilename(),
                "chunks", chunks));
    }
}
```

> [!example] cURL Test
> ```bash
> # Ingest a PDF
> curl -X POST http://localhost:8080/api/rag/ingest \
>   -F "file=@docs/shipping-policy.pdf"
>
> # Ask a question
> curl -X POST http://localhost:8080/api/rag/ask \
>   -H "Content-Type: application/json" \
>   -d '{"question": "What is the return window?"}'
> ```

---

## 8 · Testing

### Unit Test — Mock VectorStore

```java
@ExtendWith(MockitoExtension.class)
class RagServiceTest {

    @Mock VectorStore vectorStore;
    @Mock ChatClient.Builder chatClientBuilder;
    @Mock ChatClient chatClient;

    RagService ragService;

    @BeforeEach
    void setUp() {
        when(chatClientBuilder.build()).thenReturn(chatClient);
        ragService = new RagService(vectorStore, chatClientBuilder);
    }

    @Test
    void returnsAnswerFromContext() {
        var doc = new Document("Return window is 30 days.");
        when(vectorStore.similaritySearch(any(SearchRequest.class)))
                .thenReturn(List.of(doc));

        // mock the fluent ChatClient chain
        // ... assert ragService.ask("return policy?") contains "30 days"
    }
}
```

### Integration Test — Testcontainers + pgvector

```java
@SpringBootTest
@Testcontainers
class RagIntegrationTest {

    @Container
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>(
            DockerImageName.parse("pgvector/pgvector:pg16")
                    .asCompatibleSubstituteFor("postgres"))
            .withDatabaseName("testdb");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", pg::getJdbcUrl);
        registry.add("spring.datasource.username", pg::getUsername);
        registry.add("spring.datasource.password", pg::getPassword);
    }

    @Autowired DocumentIngestionService ingestionService;
    @Autowired VectorStore vectorStore;

    @Test
    void ingestAndSearch() {
        var resource = new ClassPathResource("test-docs/sample.txt");
        int chunks = ingestionService.ingest(resource);
        assertThat(chunks).isGreaterThan(0);

        List<Document> results = vectorStore.similaritySearch(
                SearchRequest.builder().query("test query").topK(3).build());
        assertThat(results).isNotEmpty();
    }
}
```

---

## 9 · Production Checklist

> [!warning] Don't Skip These

| Area | What to Do |
|------|-----------|
| **Connection pooling** | Use HikariCP (Spring Boot default). Set `maximum-pool-size` based on ingestion concurrency. |
| **Async ingestion** | Annotate `ingest()` with `@Async` or push file references onto a message queue (RabbitMQ / Kafka). |
| **Token monitoring** | Log token usage per request via `ChatResponse.getMetadata()`. Set up alerts for cost spikes. |
| **Retrieval latency** | Add Micrometer timers around `vectorStore.similaritySearch()`. Target < 100 ms p95. |
| **Vector count** | Track `SELECT count(*) FROM vector_store` on a dashboard. Know when your index grows. |
| **Re-indexing** | Keep the original `source` in metadata. To re-index: delete by filter → re-ingest. |
| **Document updates** | Use deterministic `Document.id` based on content hash. `add()` with the same id overwrites. |
| **Index tuning** | HNSW `m` and `ef_construction` params in pgvector affect recall vs speed. Benchmark with your data. |

---

## Key Takeaways

1. **Spring AI's ETL model** (`Reader → Transformer → Writer`) gives you a clean, testable ingestion pipeline — no framework lock-in.
2. **PgVectorStore** is the pragmatic default — you probably already run Postgres, just add the `pgvector` extension.
3. **`QuestionAnswerAdvisor`** gets you RAG in a single line; build a custom chain when you need control over prompts, thresholds, or re-ranking.
4. **Metadata filters** are your key to multi-tenant RAG — store `tenantId` at ingestion, filter at query time.
5. **Always match embedding dimensions** between model config and vector store schema — mismatches fail silently with garbage results.
6. **Test with Testcontainers** — spinning up a real pgvector instance in CI catches schema and query bugs that mocks never will.
7. **Monitor token usage and retrieval latency** from day one — RAG costs scale with query volume × chunk count.
