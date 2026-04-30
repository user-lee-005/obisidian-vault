---
tags:
  - llm
  - spring-ai
  - java
  - phase-4
---

# Spring AI Framework

If you're a Java/Spring developer, you don't need to abandon your ecosystem to build AI-powered applications. **Spring AI** is the official Spring project that brings LLM integration into the world of auto-configuration, dependency injection, and familiar abstractions. Let's dive in.

---

## What is Spring AI?

Spring AI is a Spring portfolio project that provides a consistent, Spring-idiomatic API for integrating with AI models. Think of it as what Spring Data did for databases — a clean abstraction over multiple providers.

**Why does it matter for us?**
- No need to learn Python/LangChain for production LLM apps
- Leverages Spring Boot auto-configuration — swap models via `application.yml`
- Dependency injection for AI components — testable, mockable
- First-class support for [[RAG - Retrieval Augmented Generation|RAG]], function calling, and structured outputs

---

## Setup

### Maven Dependency

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

For other providers, swap the artifact:
- `spring-ai-ollama-spring-boot-starter` — local models
- `spring-ai-azure-openai-spring-boot-starter` — Azure OpenAI
- `spring-ai-anthropic-spring-boot-starter` — Claude

### application.yml

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
```

> [!tip] Use environment variables
> Never hardcode API keys. Use `${OPENAI_API_KEY}` and set it in your deployment environment.

---

## Core Abstractions

### ChatClient

The primary interface for interacting with LLMs. Think of it like `RestTemplate` or `WebClient` but for AI models.

```java
@RestController
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @GetMapping("/chat")
    public String chat(@RequestParam String message) {
        return chatClient.prompt()
            .user(message)
            .call()
            .content();
    }
}
```

### ChatModel

The underlying model implementation. You rarely interact with it directly — `ChatClient` wraps it. But it's there if you need low-level control.

### Prompt and Message Types

```java
var systemMessage = new SystemMessage("You are a logistics expert.");
var userMessage = new UserMessage("What causes port congestion?");

Prompt prompt = new Prompt(List.of(systemMessage, userMessage));
```

This maps directly to the [[OpenAI API Deep Dive#The Messages Array|messages array]] concept.

---

## Code Examples

### Simple Chat Completion

```java
@Service
public class ShipmentAdvisor {

    private final ChatClient chatClient;

    public ShipmentAdvisor(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("You are a freight logistics advisor. Be concise.")
            .build();
    }

    public String getAdvice(String question) {
        return chatClient.prompt()
            .user(question)
            .call()
            .content();
    }
}
```

### Streaming Responses

For real-time UX with [[Spring WebFlux]]:

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> streamChat(@RequestParam String message) {
    return chatClient.prompt()
        .user(message)
        .stream()
        .content();
}
```

### Using System Prompts with Parameters

```java
@Value("classpath:prompts/shipment-analysis.st")
private Resource systemPromptResource;

public String analyzeShipment(String shipmentData) {
    return chatClient.prompt()
        .system(s -> s.text(systemPromptResource)
            .param("domain", "ocean freight"))
        .user(shipmentData)
        .call()
        .content();
}
```

### Structured Output with BeanOutputParser

This is where Spring AI really shines — map LLM responses directly to POJOs:

```java
public record ShipmentRisk(
    String riskLevel,
    String description,
    List<String> mitigationSteps
) {}

public ShipmentRisk assessRisk(String shipmentDetails) {
    return chatClient.prompt()
        .user(u -> u.text("""
            Assess the risk for this shipment and respond with:
            - riskLevel (LOW, MEDIUM, HIGH)
            - description of the main risk
            - mitigationSteps as a list
            
            Shipment: {details}
            """)
            .param("details", shipmentDetails))
        .call()
        .entity(ShipmentRisk.class);
}
```

Spring AI handles the JSON schema generation and parsing automatically. No more fragile regex!

---

## Function Calling in Spring AI

This is the equivalent of [[OpenAI API Deep Dive#Function Calling / Tool Use|OpenAI's function calling]] — but with Spring beans.

### Define a Tool Function

```java
@Component
@Description("Get current shipment status by tracking number")
public class ShipmentStatusFunction implements Function<ShipmentStatusRequest, ShipmentStatus> {

    private final ShipmentRepository repository;

    public ShipmentStatusFunction(ShipmentRepository repository) {
        this.repository = repository;
    }

    @Override
    public ShipmentStatus apply(ShipmentStatusRequest request) {
        return repository.findByTrackingNumber(request.trackingNumber())
            .map(s -> new ShipmentStatus(s.getStatus(), s.getLocation(), s.getEta()))
            .orElse(new ShipmentStatus("NOT_FOUND", null, null));
    }
}

public record ShipmentStatusRequest(String trackingNumber) {}
public record ShipmentStatus(String status, String location, LocalDate eta) {}
```

### Register and Use

```java
public String chatWithTools(String userMessage) {
    return chatClient.prompt()
        .user(userMessage)
        .functions("shipmentStatusFunction")
        .call()
        .content();
}
```

When a user asks "Where is shipment TRK-12345?", the LLM will:
1. Recognize it needs shipment data
2. Call your `ShipmentStatusFunction` Spring bean
3. Receive the result
4. Generate a natural language response with the data

> [!note] Your existing services become AI tools
> Any Spring bean can be exposed as a function the LLM can call. This is incredibly powerful for integrating AI into existing applications.

---

## RAG Integration

Spring AI provides first-class support for [[RAG - Retrieval Augmented Generation]]:

```java
@Configuration
public class RagConfig {

    @Bean
    public VectorStore vectorStore(EmbeddingModel embeddingModel) {
        return new PgVectorStore(jdbcTemplate, embeddingModel);
    }
}
```

### ETL Pipeline — Load, Transform, Store

```java
@Service
public class DocumentIngestionService {

    private final VectorStore vectorStore;

    public void ingestDocuments(Resource pdfResource) {
        var reader = new PagePdfDocumentReader(pdfResource);
        var transformer = new TokenTextSplitter();

        List<Document> documents = reader.get();
        List<Document> chunks = transformer.apply(documents);

        vectorStore.add(chunks);
    }
}
```

### Query with Context Retrieval

```java
public String askWithContext(String question) {
    return chatClient.prompt()
        .advisors(new QuestionAnswerAdvisor(vectorStore))
        .user(question)
        .call()
        .content();
}
```

The `QuestionAnswerAdvisor` automatically retrieves relevant documents and injects them into the prompt. Clean and declarative.

---

## Comparison: Spring AI vs LangChain4j vs DIY

| Aspect | Spring AI | LangChain4j | DIY (HTTP Client) |
|--------|-----------|-------------|-------------------|
| Spring integration | ★★★ Native | ★★ Good | ★ Manual |
| Provider support | Growing fast | Extensive | You build it |
| Abstraction level | High | High | None |
| Community | Spring ecosystem | Active | N/A |
| Production-ready | Yes (GA) | Yes | Depends on you |
| Learning curve | Low (if you know Spring) | Moderate | Low (but more work) |

**When to use Spring AI:** You're already in Spring Boot, want clean abstractions, and value the Spring ecosystem.

**When to use LangChain4j:** You need features Spring AI doesn't have yet, or want closer parity with Python's LangChain.

**When to roll your own:** Simple use case, one model, no RAG — just call the REST API directly.

---

## Key Takeaways

1. **Spring AI** makes LLM integration feel native to Java/Spring — auto-config, DI, familiar patterns
2. **ChatClient** is your main interface — it supports prompting, streaming, structured output, and tool calling
3. **Structured output** with `.entity(MyClass.class)` eliminates manual JSON parsing of LLM responses
4. **Function calling** turns your existing Spring beans into tools the LLM can invoke — no rewrites needed
5. **RAG** is built-in via `VectorStore` + `QuestionAnswerAdvisor` — wire it up, and your LLM has domain knowledge
6. Swap providers by changing a **Maven dependency and YAML config** — your code stays the same
7. If you're a Spring developer, this is your **on-ramp to AI** — no Python required
