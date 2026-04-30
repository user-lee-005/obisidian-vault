---
title: Structured Output and Parsing
tags:
  - llm
  - structured-output
  - json
  - parsing
  - phase-5
date: 2024-01-20
---

# Structured Output and Parsing

## The Problem: Free-Text Outputs Are Hard to Use

LLMs generate beautiful prose — but downstream systems need **typed data**, not paragraphs. If you ask an LLM to extract customer info, you might get:

> "The customer's name is John Smith, he lives in Chennai, and his order number is ORD-2024-456."

That's great for a human, but your Java service needs a `CustomerInfo` object with `name`, `city`, and `orderNumber` fields. How do we reliably get structured output from a model that fundamentally generates free text?

This is where **structured output** techniques come in — and they range from "politely asking" to "mathematically guaranteeing" the format.

---

## Why Structured Output Matters

- **API responses** need typed JSON, not prose
- **Database inserts** need exact field mapping
- **Downstream services** need predictable schemas
- **Validation** is impossible on unstructured text
- **Tool calling** (from [[Function Calling in LLMs]]) is itself a form of structured output

---

## Approaches: From Least to Most Reliable

### 1. Prompt-Based (Unreliable)

```
Extract the following as JSON:
{"name": "...", "city": "...", "order_number": "..."}
```

This works *most* of the time, but the model might:
- Add markdown code fences around the JSON
- Include extra explanation text
- Produce invalid JSON (missing quotes, trailing commas)

**Verdict:** Fine for prototypes, too fragile for production.

### 2. JSON Mode (Better)

OpenAI and other providers offer a `response_format` parameter:

```python
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Extract customer info as JSON..."}],
    response_format={"type": "json_object"}
)
```

This **guarantees valid JSON** — but doesn't guarantee it matches your schema. You might get `{"customer_name": "..."}` when you expected `{"name": "..."}`.

### 3. Function Calling (Reliable)

Use [[Function Calling in LLMs]] not to *call* a function, but to *extract* structured data. Define a "function" whose parameters match your desired output schema:

```python
tools = [{
    "type": "function",
    "function": {
        "name": "extract_customer_info",
        "description": "Extract structured customer information from text",
        "parameters": {
            "type": "object",
            "properties": {
                "name": {"type": "string"},
                "city": {"type": "string"},
                "order_number": {"type": "string", "pattern": "^ORD-\\d{4}-\\d+$"}
            },
            "required": ["name", "city", "order_number"]
        }
    }
}]
```

The model's output will match this schema — it's the most reliable approach for most use cases.

### 4. Constrained Generation (Guaranteed)

Some inference engines (like vLLM, llama.cpp) support **grammar-based decoding** — the model is physically constrained to only generate tokens that form valid output matching a grammar. Zero chance of format violations.

This isn't available via cloud APIs yet, but it's the gold standard for self-hosted models.

---

## OpenAI Structured Outputs

OpenAI's newest approach combines JSON mode with schema enforcement:

```python
from openai import OpenAI

client = OpenAI()

response = client.chat.completions.create(
    model="gpt-4",
    messages=[
        {"role": "system", "content": "Extract shipment data from the user's message."},
        {"role": "user", "content": "Shipment TRK-99 from Chennai to London, 500kg, departing Jan 25"}
    ],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "shipment_extraction",
            "schema": {
                "type": "object",
                "properties": {
                    "tracking_id": {"type": "string"},
                    "origin": {"type": "string"},
                    "destination": {"type": "string"},
                    "weight_kg": {"type": "number"},
                    "departure_date": {"type": "string", "format": "date"}
                },
                "required": ["tracking_id", "origin", "destination"]
            }
        }
    }
)
```

The output is **guaranteed** to match your schema. This is the recommended approach for OpenAI-based applications.

---

## Spring AI OutputParser (Java)

[[Spring AI Framework]] provides parsers that handle the prompt engineering and output parsing for you:

### BeanOutputParser — Map to a POJO

```java
public record ShipmentExtraction(
    String trackingId,
    String origin,
    String destination,
    Double weightKg,
    LocalDate departureDate
) {}

@Service
public class ExtractionService {

    private final ChatClient chatClient;

    public ShipmentExtraction extractShipment(String text) {
        var outputParser = new BeanOutputParser<>(ShipmentExtraction.class);
        
        String prompt = """
            Extract shipment information from the following text.
            
            Text: %s
            
            %s
            """.formatted(text, outputParser.getFormat());
        
        String response = chatClient.prompt()
            .user(prompt)
            .call()
            .content();
        
        return outputParser.parse(response);
    }
}
```

The `BeanOutputParser` automatically:
1. Generates format instructions (tells the model the expected JSON structure)
2. Parses the response into your Java record/class
3. Handles type conversion (String → LocalDate, String → Double, etc.)

### ListOutputParser and MapOutputParser

```java
// Extract a list of items
var listParser = new ListOutputParser(new DefaultConversionService());
String format = listParser.getFormat(); // instructs model to return comma-separated list

// Extract key-value pairs
var mapParser = new MapOutputParser();
String format = mapParser.getFormat(); // instructs model to return JSON map
```

---

## Python: Pydantic-Based Parsing

### Using the `instructor` Library

The `instructor` library patches the OpenAI client to return Pydantic models directly:

```python
import instructor
from pydantic import BaseModel, Field
from openai import OpenAI

client = instructor.from_openai(OpenAI())

class ShipmentInfo(BaseModel):
    tracking_id: str = Field(description="Shipment tracking number")
    origin: str = Field(description="Origin city")
    destination: str = Field(description="Destination city")
    weight_kg: float = Field(description="Weight in kilograms")

# Returns a validated Pydantic model — not raw text!
shipment = client.chat.completions.create(
    model="gpt-4",
    response_model=ShipmentInfo,
    messages=[
        {"role": "user", "content": "Ship TRK-99 from Chennai to London, 500kg"}
    ]
)

print(shipment.tracking_id)  # "TRK-99"
print(shipment.weight_kg)    # 500.0
```

This handles:
- Schema generation from the Pydantic model
- Response parsing and validation
- **Automatic retries** if the output doesn't match

### LangChain PydanticOutputParser

```python
from langchain.output_parsers import PydanticOutputParser
from langchain.prompts import PromptTemplate

parser = PydanticOutputParser(pydantic_object=ShipmentInfo)

prompt = PromptTemplate(
    template="Extract shipment info:\n{text}\n{format_instructions}",
    input_variables=["text"],
    partial_variables={"format_instructions": parser.get_format_instructions()}
)
```

---

## Validation and Retry

What happens when the output doesn't match? You need a retry strategy:

```python
import instructor

# instructor handles retries automatically
shipment = client.chat.completions.create(
    model="gpt-4",
    response_model=ShipmentInfo,
    max_retries=3,  # retry up to 3 times on validation failure
    messages=[...]
)
```

In Spring AI, you can wrap parsing with retry logic:

```java
public ShipmentExtraction extractWithRetry(String text) {
    for (int attempt = 0; attempt < 3; attempt++) {
        try {
            return extractShipment(text);
        } catch (OutputParserException e) {
            log.warn("Parse failed on attempt {}, retrying...", attempt + 1);
        }
    }
    throw new ExtractionFailedException("Failed after 3 attempts");
}
```

See [[Error Handling and Retries]] for more robust retry patterns.

---

## When to Use Which Approach

| Scenario | Recommended Approach |
|----------|---------------------|
| Simple extraction (1-2 fields) | JSON mode |
| Complex typed objects | Function calling or Structured Outputs |
| Python apps with validation | Pydantic + `instructor` |
| Java/Spring apps | Spring AI `BeanOutputParser` |
| Self-hosted models | Constrained generation (grammar-based) |
| Unreliable model | Function calling (most constrained via API) |

---

## Key Takeaways

1. LLM outputs are free-text by default — structured output techniques force predictable formats
2. Reliability spectrum: prompt-based (worst) → JSON mode → function calling → constrained generation (best)
3. OpenAI's `json_schema` response format **guarantees** schema compliance
4. Spring AI's `BeanOutputParser` maps LLM output directly to Java POJOs — include format instructions in the prompt
5. Python's `instructor` library + Pydantic gives you type-safe extraction with automatic retries
6. Function calling is a reliable "structured output" trick — define a function whose parameters match your desired schema
7. Always implement **validation and retry** — even the best approaches occasionally fail
8. Choose your approach based on your stack: `instructor` for Python, `BeanOutputParser` for Spring, structured outputs for OpenAI-native
