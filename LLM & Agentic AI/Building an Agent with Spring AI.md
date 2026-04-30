---
title: Building an Agent with Spring AI
tags:
  - agentic-ai
  - phase-7
  - java
  - spring-boot
  - spring-ai
  - agents
date: 2025-07-15
---

# Building an Agent with Spring AI

## Goal: Build a Tool-Using Agent in Java/Spring Boot

We've seen how agents work [[Building a Simple Agent in Python|in Python from scratch]]. Now let's build one using **Spring AI** — the official Spring framework for AI integration. Spring AI handles the tool-call loop internally, letting us focus on defining tools and business logic while leveraging the Spring ecosystem we already know.

---

## Project Setup

Add these dependencies to your `pom.xml`:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

Configure your `application.yml`:

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

---

## Defining Tools as Spring Beans

In Spring AI, tools are just `Function` beans annotated with `@Description`. The framework auto-converts them to the OpenAI function-calling format:

```java
@Configuration
public class AgentToolsConfig {

    @Bean
    @Description("Get current weather for a city. Returns temperature and conditions.")
    public Function<WeatherRequest, WeatherResponse> weatherFunction(WeatherService weatherService) {
        return request -> weatherService.getWeather(request.city());
    }

    @Bean
    @Description("Search a knowledge base for company policies and documentation.")
    public Function<SearchRequest, SearchResponse> searchFunction(VectorStoreService vectorStore) {
        return request -> vectorStore.search(request.query(), request.topK());
    }

    @Bean
    @Description("Execute a read-only SQL query against the analytics database.")
    public Function<QueryRequest, QueryResponse> databaseQuery(JdbcTemplate jdbcTemplate) {
        return request -> {
            List<Map<String, Object>> results = jdbcTemplate.queryForList(request.sql());
            return new QueryResponse(results, results.size());
        };
    }
}

// Record types for requests/responses
public record WeatherRequest(String city) {}
public record WeatherResponse(String city, double temperature, String conditions) {}
public record SearchRequest(String query, int topK) {}
public record SearchResponse(List<String> results, double relevanceScore) {}
public record QueryRequest(String sql) {}
public record QueryResponse(List<Map<String, Object>> rows, int count) {}
```

The beauty here: your tools are just regular Spring beans. They can inject services, use `@Transactional`, leverage connection pools — all the Spring features you already know.

---

## ChatClient Configuration

The `ChatClient` is your agent's entry point. Configure it with a system prompt and available tools:

```java
@Configuration
public class AgentConfig {

    @Bean
    public ChatClient chatClient(ChatClient.Builder builder) {
        return builder
            .defaultSystem("""
                You are a helpful enterprise assistant. You have access to tools for:
                - Checking weather information
                - Searching company documentation
                - Querying the analytics database (read-only SQL)
                
                Think step by step. Use tools when you need real data.
                Always cite your sources when using search results.
                Never execute destructive SQL — only SELECT queries.
                """)
            .defaultFunctions("weatherFunction", "searchFunction", "databaseQuery")
            .build();
    }
}
```

---

## The Agent Loop in Spring AI

Here's the elegant part: **Spring AI handles the tool-call cycle internally**. When the LLM responds with a tool call, Spring AI automatically:
1. Parses the function call
2. Finds the matching bean
3. Invokes it with deserialized arguments
4. Sends the result back to the LLM
5. Repeats until the LLM gives a final answer

You just call `.content()` and it handles the loop:

```java
@Service
public class AgentService {

    private final ChatClient chatClient;

    public AgentService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String chat(String userMessage) {
        return chatClient.prompt()
            .user(userMessage)
            .call()
            .content();  // blocks until final answer (handles tool loops internally)
    }

    public Flux<String> chatStream(String userMessage) {
        return chatClient.prompt()
            .user(userMessage)
            .stream()
            .content();  // streams the final answer token by token
    }
}
```

---

## Adding Conversation Memory

Agents need memory to handle multi-turn conversations. Spring AI provides several options:

```java
@Configuration
public class MemoryConfig {

    @Bean
    public ChatMemory chatMemory() {
        return new InMemoryChatMemory();  // simple in-memory store
    }

    @Bean
    public ChatClient chatClientWithMemory(ChatClient.Builder builder, ChatMemory chatMemory) {
        return builder
            .defaultSystem("You are a helpful assistant with memory of past conversations.")
            .defaultFunctions("weatherFunction", "searchFunction")
            .defaultAdvisors(new MessageChatMemoryAdvisor(chatMemory))
            .build();
    }
}
```

For session-scoped memory (per-user conversations):

```java
@Service
public class SessionAgentService {

    private final ChatClient chatClient;
    private final ChatMemory chatMemory;

    public String chat(String sessionId, String userMessage) {
        return chatClient.prompt()
            .user(userMessage)
            .advisors(advisor -> advisor
                .param(ChatMemory.CONVERSATION_ID, sessionId))
            .call()
            .content();
    }
}
```

---

## Building a REST API Endpoint

Let's expose our agent as a REST API with both blocking and streaming endpoints:

```java
@RestController
@RequestMapping("/api/agent")
public class AgentController {

    private final SessionAgentService agentService;

    public AgentController(SessionAgentService agentService) {
        this.agentService = agentService;
    }

    @PostMapping("/chat")
    public ResponseEntity<ChatResponse> chat(
            @RequestBody ChatRequest request,
            @RequestHeader("X-Session-Id") String sessionId) {
        
        String response = agentService.chat(sessionId, request.message());
        return ResponseEntity.ok(new ChatResponse(response));
    }

    @PostMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> chatStream(
            @RequestBody ChatRequest request,
            @RequestHeader("X-Session-Id") String sessionId) {
        
        return agentService.chatStream(sessionId, request.message());
    }
}

public record ChatRequest(String message) {}
public record ChatResponse(String reply) {}
```

---

## Adding Custom Tools

Need a domain-specific tool? Just define it as a bean. Here's a tool that calls an internal REST API:

```java
@Bean
@Description("Get shipment status by tracking number. Returns current location and ETA.")
public Function<TrackingRequest, TrackingResponse> shipmentTracker(RestClient restClient) {
    return request -> {
        var status = restClient.get()
            .uri("/api/shipments/{id}/status", request.trackingNumber())
            .retrieve()
            .body(TrackingResponse.class);
        return status;
    };
}

@Bean
@Description("Create a support ticket for escalation. Requires subject and description.")
public Function<TicketRequest, TicketResponse> createTicket(TicketService ticketService) {
    return request -> {
        var ticket = ticketService.create(request.subject(), request.description(), request.priority());
        return new TicketResponse(ticket.getId(), ticket.getStatus());
    };
}
```

---

## Error Handling

Tools fail — APIs timeout, databases go down. Use Spring's retry mechanism:

```java
@Bean
@Description("Get current weather for a city")
public Function<WeatherRequest, WeatherResponse> weatherFunction(WeatherService weatherService) {
    return request -> {
        try {
            return weatherService.getWeather(request.city());
        } catch (WebClientResponseException e) {
            return new WeatherResponse(request.city(), 0, "Service unavailable: " + e.getStatusCode());
        } catch (Exception e) {
            return new WeatherResponse(request.city(), 0, "Error: " + e.getMessage());
        }
    };
}
```

For retry logic, annotate service methods:

```java
@Service
public class WeatherService {
    
    @Retryable(maxAttempts = 3, backoff = @Backoff(delay = 1000))
    public WeatherResponse getWeather(String city) {
        // API call that might fail
    }
    
    @Recover
    public WeatherResponse fallback(Exception e, String city) {
        return new WeatherResponse(city, 0, "Weather service temporarily unavailable");
    }
}
```

---

## Testing

Use `MockChatModel` to test your agent logic without real API calls:

```java
@SpringBootTest
class AgentServiceTest {

    @MockBean
    private ChatModel chatModel;

    @Autowired
    private AgentService agentService;

    @Test
    void shouldUseWeatherToolForWeatherQuestions() {
        // Setup mock response
        when(chatModel.call(any(Prompt.class)))
            .thenReturn(new ChatResponse(List.of(
                new Generation(new AssistantMessage("The weather in London is 15°C and cloudy."))
            )));

        String response = agentService.chat("What's the weather in London?");
        
        assertThat(response).contains("London");
        verify(chatModel).call(any(Prompt.class));
    }
}
```

---

## Deployment Considerations

When moving to production, think about:

- **Token costs** — each agent loop iteration costs tokens; set `maxTokens` limits
- **Rate limiting** — use a token bucket per session to prevent abuse
- **Observability** — Spring AI integrates with Micrometer for metrics; trace tool calls
- **Timeouts** — set `spring.ai.openai.chat.options.timeout` to prevent hanging requests
- **API key rotation** — use Spring Cloud Vault for secret management

See [[Production Considerations]] for the full production readiness checklist.

---

## Key Takeaways

1. Spring AI lets you define tools as **regular Spring beans** — inject services, use transactions, leverage the full ecosystem
2. The **tool-call loop is handled internally** by `ChatClient` — you just call `.content()` and it orchestrates everything
3. Memory is added via **advisors** — `MessageChatMemoryAdvisor` handles session-scoped conversation history
4. Error handling should return **graceful error messages** in the response object rather than throwing exceptions
5. Streaming with `Flux<String>` gives users **real-time feedback** while the agent works
6. Testing is straightforward with **MockChatModel** — no real API calls needed in unit tests
7. Spring AI is the natural choice for Java teams already in the Spring ecosystem — it follows familiar patterns (beans, DI, configuration)
