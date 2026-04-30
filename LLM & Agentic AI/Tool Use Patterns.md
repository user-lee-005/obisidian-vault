---
title: Tool Use Patterns
tags:
  - llm
  - tools
  - patterns
  - phase-5
date: 2024-01-20
---

# Tool Use Patterns

## What Is a "Tool" in the LLM Context?

A **tool** is any function the model can invoke to interact with the outside world. It's the bridge between "thinking" and "doing." When we say an LLM "uses tools," we mean it outputs a structured request to call a function, and our application executes it (as we covered in [[Function Calling in LLMs]]).

Tools are how we give LLMs **superpowers** — access to real-time data, computation, and the ability to take actions in the world.

---

## Common Tool Categories

### Information Retrieval
- **Web search** — find current information the model wasn't trained on
- **Database queries** — look up customer records, order status, inventory
- **API lookups** — stock prices, weather, exchange rates, shipping status
- **Document search** — query a [[RAG (Retrieval-Augmented Generation)]] knowledge base

### Computation
- **Calculators** — precise math (LLMs are notoriously bad at arithmetic)
- **Code execution** — run Python/JS snippets for data analysis
- **Data transformation** — convert formats, parse dates, aggregate numbers

### Actions
- **Send email/SMS** — notifications, confirmations
- **Create tickets** — JIRA, ServiceNow, internal systems
- **Update database** — modify records, change status
- **Call REST APIs** — trigger workflows, update external systems
- **Schedule events** — calendar entries, reminders

### File Operations
- **Read/write files** — generate reports, parse uploads
- **Parse documents** — extract data from PDFs, CSVs, Excel
- **Generate files** — create exports, invoices, manifests

---

## Design Principles for Good Tools

The model **reads your tool descriptions** to decide which tool to use. This makes design critical:

### Clear, Descriptive Names and Descriptions

```json
{
  "name": "search_shipments",
  "description": "Search for shipments by tracking number, origin, destination, or date range. Returns shipment status, ETA, and carrier details."
}
```

The description is like a docstring for the model. Be specific about what the tool does, what it returns, and when to use it.

### Well-Defined Input Schemas

```json
{
  "parameters": {
    "type": "object",
    "properties": {
      "tracking_number": {
        "type": "string",
        "description": "Shipment tracking number (e.g., TRK-2024-00123)"
      },
      "status": {
        "type": "string",
        "enum": ["in_transit", "delivered", "pending", "cancelled"],
        "description": "Filter by shipment status"
      },
      "date_from": {
        "type": "string",
        "format": "date",
        "description": "Start date for search range (YYYY-MM-DD)"
      }
    },
    "required": ["tracking_number"]
  }
}
```

Use **enums** for constrained values — the model will pick from the list rather than guessing.

### Atomic Operations

One tool = one action. Don't create a `manage_shipment` tool that creates, updates, deletes, and queries. Instead:
- `create_shipment`
- `update_shipment_status`
- `cancel_shipment`
- `get_shipment_details`

### Idempotent Where Possible

If the model retries a tool call (due to [[Error Handling and Retries]]), it shouldn't create duplicate records. Use upsert patterns or idempotency keys.

### Return Structured Results

```json
{
  "shipment_id": "TRK-2024-00123",
  "status": "in_transit",
  "eta": "2024-01-22T14:00:00Z",
  "carrier": "DHL",
  "last_location": "Singapore Hub"
}
```

Return data the model can **reason about**, not raw HTML or binary blobs.

---

## Tool Selection: Less Is More

Giving the model too many tools **degrades performance**. The model has to read and evaluate every tool description. Research shows:

- **5-10 tools** — works great, high accuracy
- **10-20 tools** — still good, slight degradation
- **50+ tools** — significant accuracy drop, model gets confused

Strategies for large tool sets:
- **Category-based routing** — first call picks a category, second call picks specific tool
- **Dynamic tool injection** — only provide relevant tools based on conversation context
- **Tool descriptions as retrieval** — use [[RAG (Retrieval-Augmented Generation)]] to find relevant tools

---

## Tool Composition: Chains of Tools

Real-world tasks often need multiple tools in sequence:

```
User: "Find the cheapest flight from Chennai to London next week and email me the details"

1. search_flights(from="Chennai", to="London", date_range="next_week")
2. sort_results(by="price", order="ascending")
3. send_email(to="user@company.com", subject="Flight Options", body=formatted_results)
```

The model orchestrates this chain, using the output of one tool as input to the next. This is where things start looking like [[Agentic AI]] behavior.

---

## Examples

### Weather Lookup Tool (Python)

```python
def get_weather(city: str, units: str = "celsius") -> dict:
    """Get current weather for a city."""
    response = requests.get(
        f"https://api.weatherapi.com/v1/current.json",
        params={"key": API_KEY, "q": city}
    )
    data = response.json()
    temp = data["current"]["temp_c"] if units == "celsius" else data["current"]["temp_f"]
    return {
        "city": city,
        "temperature": temp,
        "condition": data["current"]["condition"]["text"],
        "humidity": data["current"]["humidity"]
    }
```

### Database Query Tool (Spring / Java)

```java
@Bean
@Description("Search shipments by tracking number or status")
public Function<ShipmentQuery, List<ShipmentDTO>> searchShipments(
        ShipmentRepository repository) {
    return query -> {
        if (query.trackingNumber() != null) {
            return repository.findByTrackingNumber(query.trackingNumber())
                .stream().map(ShipmentDTO::from).toList();
        }
        return repository.findByStatus(query.status())
            .stream().map(ShipmentDTO::from).toList();
    };
}

record ShipmentQuery(String trackingNumber, String status) {}
```

### REST API Call Tool (Python)

```python
tools = [{
    "type": "function",
    "function": {
        "name": "create_jira_ticket",
        "description": "Create a JIRA ticket for tracking an issue",
        "parameters": {
            "type": "object",
            "properties": {
                "title": {"type": "string", "description": "Ticket title"},
                "description": {"type": "string", "description": "Detailed description"},
                "priority": {"type": "string", "enum": ["low", "medium", "high", "critical"]},
                "assignee": {"type": "string", "description": "Username to assign to"}
            },
            "required": ["title", "priority"]
        }
    }
}]
```

---

## Security Considerations

> [!danger] Never Let the Model Construct Raw SQL or Code
> The model might try to inject malicious queries or code. Always:
> - Use parameterized queries, never string concatenation
> - Validate and sanitize all arguments before execution
> - Limit tool permissions (read-only where possible)
> - Log all tool calls for audit

```python
# BAD - model controls the query
def run_query(sql: str):
    cursor.execute(sql)  # SQL injection risk!

# GOOD - model provides parameters, you control the query
def search_orders(customer_id: str, status: str):
    cursor.execute(
        "SELECT * FROM orders WHERE customer_id = %s AND status = %s",
        (customer_id, status)
    )
```

Think of tools as an **API surface** — apply the same security practices you'd use for a public REST API.

---

## Key Takeaways

1. A "tool" is any function the model can request to invoke — it bridges thinking and doing
2. Design tools with **clear descriptions** — the model reads them to decide what to call
3. Use typed schemas with enums for constrained values — reduces model errors
4. Keep tools **atomic** (one action each) and **idempotent** (safe to retry)
5. Fewer tools = better accuracy — 5-10 is the sweet spot; use routing for larger sets
6. Tool composition enables multi-step workflows — output of one feeds into the next
7. **Never** let models construct raw SQL or executable code — always validate and parameterize
8. Return structured data the model can reason about, not raw blobs
