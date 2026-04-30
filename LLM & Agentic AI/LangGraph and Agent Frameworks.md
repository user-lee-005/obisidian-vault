---
title: LangGraph and Agent Frameworks
tags:
  - agentic-ai
  - phase-7
  - langgraph
  - frameworks
  - python
  - orchestration
date: 2025-07-15
---

# LangGraph and Agent Frameworks

## Why Frameworks?

We've [[Building a Simple Agent in Python|built an agent from scratch]] — and it worked! But as requirements grow (multi-step workflows, human approval, branching logic, persistence), the simple loop gets messy fast. Frameworks handle the common patterns so we can focus on business logic.

The question isn't "should I use a framework?" but "which framework fits my problem?"

---

## LangGraph: Graph-Based Agent Orchestration

**LangGraph** (by LangChain) models agents as **state machines** — directed graphs where:
- **Nodes** are steps (LLM calls, tool execution, human input, validation)
- **Edges** are transitions between steps (including conditional routing)
- **State** flows through the graph and accumulates results

This is powerful because complex agent behavior becomes *visible* and *debuggable* — you can literally draw the agent's decision tree.

### Core Concepts

| Concept | What It Does |
|---------|--------------|
| `StateGraph` | Defines the graph structure |
| Nodes | Functions that process and update state |
| Edges | Connections between nodes |
| Conditional edges | Route based on state (branching logic) |
| State | TypedDict shared across all nodes |
| Checkpointing | Save/resume graph execution |

---

## Building a LangGraph Agent

Let's build an agent that can search the web and do math, with an explicit graph structure:

```python
from typing import TypedDict, Annotated, Sequence
from langchain_core.messages import BaseMessage, HumanMessage, AIMessage
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
import operator

# 1. Define the state that flows through our graph
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], operator.add]

# 2. Define our tools
from langchain_core.tools import tool

@tool
def search(query: str) -> str:
    """Search the web for information."""
    # Real implementation would call an API
    return f"Search results for: {query}"

@tool
def calculate(expression: str) -> str:
    """Evaluate a math expression."""
    return str(eval(expression))

tools = [search, calculate]

# 3. Create the LLM with tool binding
llm = ChatOpenAI(model="gpt-4o").bind_tools(tools)

# 4. Define node functions
def call_model(state: AgentState) -> dict:
    """The 'thinking' node — calls the LLM."""
    messages = state["messages"]
    response = llm.invoke(messages)
    return {"messages": [response]}

def should_continue(state: AgentState) -> str:
    """Conditional edge: decide whether to use tools or finish."""
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "continue"  # route to tools node
    return "end"           # route to END

# 5. Build the graph
graph = StateGraph(AgentState)

# Add nodes
graph.add_node("agent", call_model)
graph.add_node("tools", ToolNode(tools))

# Set entry point
graph.set_entry_point("agent")

# Add edges
graph.add_conditional_edges(
    "agent",
    should_continue,
    {"continue": "tools", "end": END}
)
graph.add_edge("tools", "agent")  # after tools, always go back to agent

# 6. Compile and run
app = graph.compile()

# Invoke the agent
result = app.invoke({
    "messages": [HumanMessage(content="What is 25 * 48 + 137?")]
})
print(result["messages"][-1].content)
```

---

## Key LangGraph Patterns

### Conditional Routing (Branching)

Route to different nodes based on the agent's decision:

```python
def route_by_intent(state: AgentState) -> str:
    """Route based on what the user wants."""
    last_msg = state["messages"][-1].content.lower()
    if "weather" in last_msg:
        return "weather_node"
    elif "database" in last_msg:
        return "db_node"
    return "general_node"

graph.add_conditional_edges("classifier", route_by_intent, {
    "weather_node": "weather",
    "db_node": "database",
    "general_node": "general"
})
```

### Human-in-the-Loop

Interrupt the graph for human approval before executing sensitive actions:

```python
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()
app = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["dangerous_action_node"]  # pause here for approval
)

# Run until interruption
config = {"configurable": {"thread_id": "user-123"}}
result = app.invoke({"messages": [HumanMessage("Delete all records")]}, config)
# → Graph pauses before "dangerous_action_node"

# Human approves → resume
result = app.invoke(None, config)  # continues from checkpoint
```

This is essential for [[Guardrails and Safety Layers|safety]] — you want a human to approve destructive actions.

### Persistence (Checkpointing)

Save graph state so you can resume later (critical for long-running agents):

```python
from langgraph.checkpoint.sqlite import SqliteSaver

# Persist to SQLite
checkpointer = SqliteSaver.from_conn_string("agent_state.db")
app = graph.compile(checkpointer=checkpointer)

# Every invocation saves state
config = {"configurable": {"thread_id": "conversation-456"}}
app.invoke({"messages": [HumanMessage("Start a research task")]}, config)

# Later (even after restart), resume the same conversation
app.invoke({"messages": [HumanMessage("What did you find?")]}, config)
```

### Streaming Intermediate Steps

Show the user what's happening as the agent works:

```python
async for event in app.astream_events(
    {"messages": [HumanMessage("Research quantum computing")]},
    version="v2"
):
    if event["event"] == "on_chat_model_stream":
        print(event["data"]["chunk"].content, end="", flush=True)
    elif event["event"] == "on_tool_start":
        print(f"\n🔧 Using tool: {event['name']}")
```

---

## Other Agent Frameworks

### CrewAI — Multi-Agent with Roles

Best for teams of specialized agents working together:

```python
from crewai import Agent, Task, Crew

researcher = Agent(
    role="Senior Researcher",
    goal="Find comprehensive information about the topic",
    backstory="Expert research analyst with 20 years of experience",
    tools=[search_tool, web_scraper]
)

writer = Agent(
    role="Technical Writer",
    goal="Write clear, engaging content based on research",
    backstory="Award-winning technical writer"
)

research_task = Task(description="Research {topic}", agent=researcher)
writing_task = Task(description="Write a report based on research", agent=writer)

crew = Crew(agents=[researcher, writer], tasks=[research_task, writing_task])
result = crew.kickoff(inputs={"topic": "quantum computing"})
```

### AutoGen (Microsoft) — Multi-Agent Conversations

Agents that talk to each other:

```python
from autogen import AssistantAgent, UserProxyAgent

assistant = AssistantAgent("assistant", llm_config={"model": "gpt-4o"})
user_proxy = UserProxyAgent("user_proxy", code_execution_config={"work_dir": "coding"})

user_proxy.initiate_chat(assistant, message="Write a Python script to analyze CSV data")
```

### Semantic Kernel (Microsoft) — C#/Python/Java

Best for Microsoft ecosystem integration:

```csharp
var kernel = Kernel.CreateBuilder()
    .AddOpenAIChatCompletion("gpt-4o", apiKey)
    .Build();

kernel.Plugins.AddFromType<WeatherPlugin>();
kernel.Plugins.AddFromType<DatabasePlugin>();

var result = await kernel.InvokePromptAsync("What's the weather in Seattle?");
```

### LangChain4j — Java-Native

Alternative to Spring AI with LangChain-style abstractions:

```java
var assistant = AiServices.builder(Assistant.class)
    .chatLanguageModel(model)
    .tools(new WebSearchTool(), new CalculatorTool())
    .chatMemory(MessageWindowChatMemory.withMaxMessages(20))
    .build();

String answer = assistant.chat("What is the population of Japan times 2?");
```

---

## Framework Comparison

| Framework | Language | Best For | Complexity |
|-----------|----------|----------|------------|
| **LangGraph** | Python | Complex flows, conditional logic, human-in-loop | High |
| **CrewAI** | Python | Multi-agent teams, role-based collaboration | Medium |
| **AutoGen** | Python | Multi-agent conversations, code generation | Medium |
| **Semantic Kernel** | C#/Python/Java | Microsoft ecosystem, Azure integration | Medium |
| **LangChain4j** | Java | Java developers wanting LangChain patterns | Medium |
| **Spring AI** | Java | Spring Boot applications, enterprise Java | Low-Medium |

---

## When to Use a Framework vs Build from Scratch

**Build from scratch when:**
- Your use case is simple (single tool loop)
- You need full control over every decision
- You're learning how agents work
- Framework overhead isn't justified

**Use a framework when:**
- You need complex routing/branching logic
- You want human-in-the-loop approval
- You need persistence and checkpointing
- You're building multi-agent systems
- You want built-in observability and tracing

The decision comes down to: **is the framework saving me time, or is it adding complexity I don't need?** Start simple ([[Building a Simple Agent in Python|from scratch]]), and reach for a framework when the simple approach becomes painful.

---

## Key Takeaways

1. **LangGraph** models agents as state machines — nodes are steps, edges are transitions, state flows through the graph
2. Conditional edges enable **branching logic** — route to different nodes based on agent decisions or state
3. **Human-in-the-loop** is built into LangGraph via `interrupt_before` — critical for [[Guardrails and Safety Layers|safety-sensitive]] actions
4. **Checkpointing** enables persistence — save and resume long-running agent conversations across restarts
5. **CrewAI** is the simplest path to multi-agent systems — define roles, give tasks, let agents collaborate
6. Choose frameworks based on your ecosystem: **Spring AI/LangChain4j** for Java, **LangGraph** for complex Python flows, **Semantic Kernel** for Microsoft
7. Start from scratch to learn, then adopt a framework when complexity demands it — don't reach for a framework before you need one
