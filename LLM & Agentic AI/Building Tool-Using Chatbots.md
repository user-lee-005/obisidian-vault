---
title: Building Tool-Using Chatbots
tags:
  - llm
  - chatbot
  - tools
  - spring-ai
  - phase-5
date: 2024-01-20
---

# Building Tool-Using Chatbots

## Architecture Overview

A tool-using chatbot has three core components:

1. **Chat loop** — handles user input/output and orchestrates the flow
2. **Tool execution layer** — receives structured calls, executes them, returns results
3. **Conversation memory** — maintains context including tool call history

This is where [[Function Calling in LLMs]] and [[Tool Use Patterns]] come together into a working system. Let's build one.

---

## The Conversation Loop

Here's the fundamental pattern every tool-using chatbot follows:

```python
messages = [{"role": "system", "content": "You are a helpful assistant."}]
tools = [...]  # tool definitions

while True:
    user_input = input("You: ")
    messages.append({"role": "user", "content": user_input})
    
    response = client.chat.completions.create(
        model="gpt-4", messages=messages, tools=tools
    )
    
    assistant_message = response.choices[0].message
    messages.append(assistant_message)  # keep in history
    
    # Tool call loop — model might need multiple rounds
    while assistant_message.tool_calls:
        for tool_call in assistant_message.tool_calls:
            result = execute_tool(tool_call)
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": json.dumps(result)
            })
        
        # Re-prompt with tool results
        response = client.chat.completions.create(
            model="gpt-4", messages=messages, tools=tools
        )
        assistant_message = response.choices[0].message
        messages.append(assistant_message)
    
    print(f"Assistant: {assistant_message.content}")
```

The key insight: **after executing tools, you re-prompt the model** with the results so it can generate a natural language response.

---

## Conversation Memory: Tool Calls in the Message Array

The message history includes **everything** — user messages, assistant responses, tool calls, and tool results:

```json
[
  {"role": "system", "content": "You are a logistics assistant."},
  {"role": "user", "content": "Where is shipment TRK-123?"},
  {"role": "assistant", "tool_calls": [{"id": "call_1", "function": {"name": "get_shipment", "arguments": "{\"id\": \"TRK-123\"}"}}]},
  {"role": "tool", "tool_call_id": "call_1", "content": "{\"status\": \"in_transit\", \"eta\": \"2024-01-22\"}"},
  {"role": "assistant", "content": "Shipment TRK-123 is currently in transit with an ETA of January 22nd."},
  {"role": "user", "content": "Can you delay it by 2 days?"}
]
```

This history lets the model reference previous tool results and maintain context across turns.

---

## Implementation in Spring AI (Java)

[[Spring AI Framework]] makes tool-using chatbots remarkably clean:

### Defining Tools as @Bean Functions

```java
@Configuration
public class ChatTools {

    @Bean
    @Description("Get shipment details by tracking number")
    public Function<TrackingRequest, ShipmentInfo> getShipment(
            ShipmentService shipmentService) {
        return request -> shipmentService.findByTracking(request.trackingNumber());
    }

    @Bean
    @Description("Update the estimated delivery date for a shipment")
    public Function<UpdateEtaRequest, UpdateResult> updateShipmentEta(
            ShipmentService shipmentService) {
        return request -> shipmentService.updateEta(
            request.trackingNumber(), request.newEta()
        );
    }
}

record TrackingRequest(String trackingNumber) {}
record UpdateEtaRequest(String trackingNumber, LocalDate newEta) {}
```

### ChatClient with Function Callbacks

```java
@RestController
@RequestMapping("/api/chat")
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("You are a logistics assistant for a freight company.")
            .build();
    }

    @PostMapping
    public String chat(@RequestBody ChatRequest request) {
        return chatClient.prompt()
            .user(request.message())
            .functions("getShipment", "updateShipmentEta")  // enable these tools
            .call()
            .content();
    }
}
```

Spring AI handles the entire tool call → result → re-prompt cycle **automatically**. You define the functions, and the framework orchestrates the loop.

---

## Implementation in Python (OpenAI SDK)

### Full Working Example

```python
import openai
import json

client = openai.OpenAI()

# Tool definitions
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_shipment_status",
            "description": "Get current status of a shipment by tracking number",
            "parameters": {
                "type": "object",
                "properties": {
                    "tracking_number": {
                        "type": "string",
                        "description": "The shipment tracking number"
                    }
                },
                "required": ["tracking_number"]
            }
        }
    }
]

# Tool implementation
def get_shipment_status(tracking_number: str) -> dict:
    # In reality, this calls your database or API
    return {
        "tracking_number": tracking_number,
        "status": "in_transit",
        "location": "Singapore Hub",
        "eta": "2024-01-22"
    }

# Dispatch table
tool_functions = {
    "get_shipment_status": get_shipment_status
}

def execute_tool(tool_call):
    func = tool_functions[tool_call.function.name]
    args = json.loads(tool_call.function.arguments)
    return func(**args)

# Chat loop
def chat(user_message: str, messages: list) -> str:
    messages.append({"role": "user", "content": user_message})
    
    response = client.chat.completions.create(
        model="gpt-4", messages=messages, tools=tools
    )
    
    msg = response.choices[0].message
    messages.append(msg)
    
    while msg.tool_calls:
        for tc in msg.tool_calls:
            result = execute_tool(tc)
            messages.append({
                "role": "tool",
                "tool_call_id": tc.id,
                "content": json.dumps(result)
            })
        
        response = client.chat.completions.create(
            model="gpt-4", messages=messages, tools=tools
        )
        msg = response.choices[0].message
        messages.append(msg)
    
    return msg.content
```

---

## Multi-Turn Tool Use

Sometimes a single tool call leads to more tool calls. Imagine:

1. User: "Book the cheapest flight to London next week and add it to my calendar"
2. Model calls `search_flights(destination="London", date="next_week")`
3. Results come back → Model calls `book_flight(flight_id="FL-123")`
4. Booking confirmed → Model calls `create_calendar_event(title="Flight to London", date="...")`
5. Model responds: "Done! I've booked flight FL-123 and added it to your calendar."

The inner `while` loop handles this naturally — it keeps looping as long as the model outputs tool calls.

---

## Conversation Design: When to Use Tools vs. Knowledge?

Not every question needs a tool call. The model should:

- **Use tools** when: data is time-sensitive, user-specific, or requires action
- **Answer from knowledge** when: question is general, conceptual, or about well-known facts

You can guide this in the system prompt:

```
Use the search_shipments tool for any question about specific shipments.
For general questions about shipping processes or policies, answer from your knowledge.
```

---

## UX Considerations

- **Streaming responses** — stream the final text response, but tool calls happen in a blocking step before streaming starts
- **Show "thinking" state** — while tools execute, show the user something is happening
- **Tool call transparency** — optionally show which tools were called (builds trust)

```
🔍 Looking up shipment TRK-123...
✅ Found! Here's what I know:
```

---

## Error Handling

What happens when a tool fails mid-conversation? See [[Error Handling and Retries]] for detailed patterns, but the summary:

- **Don't silently swallow errors** — feed the error back to the model
- **Let the model adapt** — it might try a different approach or inform the user
- **Set timeouts** — don't let a stuck API hang your chatbot forever

```python
try:
    result = execute_tool(tool_call)
except TimeoutError:
    result = {"error": "Service timed out. The shipment tracking system is currently slow."}
except Exception as e:
    result = {"error": f"Tool execution failed: {str(e)}"}
```

The model receives this error and can respond gracefully: "I'm having trouble reaching the tracking system right now. Can I help you with something else?"

---

## Key Takeaways

1. The chatbot loop is: user input → LLM → (optional tool calls → execute → re-prompt) → display response
2. **Conversation memory must include tool calls and results** — the model needs full context
3. Spring AI automates the tool call cycle — define `@Bean` functions and the framework handles orchestration
4. In Python, you manage the loop explicitly — check for `tool_calls`, execute, append results, re-prompt
5. Multi-turn tool use happens naturally when the inner loop continues until no more tool calls
6. Guide the model on **when** to use tools vs. answer from knowledge via system prompts
7. Always feed errors back to the model as structured messages — let it adapt gracefully
8. Show users what's happening — transparency builds trust in tool-using chatbots
