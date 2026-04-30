---
title: MCP - Model Context Protocol
tags:
  - agentic-ai
  - phase-7
  - mcp
  - protocol
  - tools
  - anthropic
date: 2025-07-15
---

# MCP — Model Context Protocol

## What is MCP?

**Model Context Protocol (MCP)** is an open protocol created by Anthropic that standardizes how LLMs connect to external tools and data sources. Think of it like USB for AI — a universal interface that lets any LLM host talk to any tool server without custom integration code.

Before MCP, every tool integration was bespoke. Your chatbot needed custom code to talk to GitHub, custom code for databases, custom code for file systems. MCP says: "Let's agree on a protocol, and then tools only need to be written once."

---

## The Problem MCP Solves

Imagine you're building an AI assistant that needs to:
- Read files from disk
- Query a database
- Search GitHub issues
- Send Slack messages

Without MCP, you write **four separate integrations**, each with its own auth, serialization, error handling, and discovery mechanism. If you switch from OpenAI to Claude, you might need to rewrite them all.

With MCP, each capability is an **MCP server** that speaks a standard protocol. Your app (the host) connects to them all the same way. Switch LLMs? The tools still work. Share your database server with a colleague? They just connect to it.

---

## Architecture

MCP has three layers:

```
┌─────────────────────────────────────────────┐
│  MCP Host (IDE, chatbot, AI application)    │
│  ┌─────────────────────────────────────┐    │
│  │  MCP Client (protocol handler)      │    │
│  └──────┬──────────┬──────────┬────────┘    │
└─────────┼──────────┼──────────┼─────────────┘
          │          │          │
    ┌─────▼───┐ ┌────▼────┐ ┌──▼───────┐
    │MCP Server│ │MCP Server│ │MCP Server│
    │(GitHub)  │ │(Database)│ │(Files)   │
    └─────────┘ └─────────┘ └──────────┘
```

- **MCP Host** — the application using an LLM (VS Code, a chatbot, Claude Desktop)
- **MCP Client** — handles the protocol communication (built into the host)
- **MCP Server** — exposes capabilities (tools, resources, prompts) over the protocol

---

## Three Primitives

MCP defines three types of capabilities a server can expose:

### 1. Tools — Functions the Model Can Call

Like [[Function Calling and Tool Use|function calling]], but standardized:

```json
{
  "name": "get_weather",
  "description": "Get current weather for a location",
  "inputSchema": {
    "type": "object",
    "properties": {
      "city": {"type": "string", "description": "City name"},
      "units": {"type": "string", "enum": ["celsius", "fahrenheit"]}
    },
    "required": ["city"]
  }
}
```

### 2. Resources — Data the Model Can Read

Files, database records, API responses — anything the model might need as context:

```json
{
  "uri": "file:///project/src/main.py",
  "name": "Main application file",
  "mimeType": "text/x-python"
}
```

Resources are like giving the model a **filesystem** it can browse.

### 3. Prompts — Reusable Prompt Templates

Pre-built prompts that servers can offer:

```json
{
  "name": "code_review",
  "description": "Review code for bugs, security issues, and best practices",
  "arguments": [
    {"name": "code", "description": "The code to review", "required": true},
    {"name": "language", "description": "Programming language"}
  ]
}
```

---

## How It Works

The lifecycle of an MCP interaction:

1. **Discovery** — Host discovers available MCP servers (via config)
2. **Connection** — Client establishes connection to each server (stdio or HTTP/SSE)
3. **Capability advertisement** — Servers tell the client what tools/resources/prompts they offer
4. **Model invocation** — User asks a question, model sees available tools
5. **Tool call** — Model decides to use a tool → host routes the call to the correct server
6. **Execution** — Server executes the tool and returns the result
7. **Continuation** — Result goes back to the model, which continues reasoning

This is similar to the [[Building a Simple Agent in Python|agent loop]] we built — but the tool discovery and routing is standardized.

---

## Building an MCP Server (Python)

Let's build a simple MCP server that provides weather and time tools:

```python
from mcp.server.fastmcp import FastMCP
from datetime import datetime
import httpx

# Create the server
mcp = FastMCP("my-tools")

@mcp.tool()
async def get_weather(city: str) -> str:
    """Get current weather for a city. Returns temperature and conditions."""
    async with httpx.AsyncClient() as client:
        resp = await client.get(
            f"https://wttr.in/{city}",
            params={"format": "%t %C"},
            headers={"User-Agent": "mcp-weather-server"}
        )
        return f"Weather in {city}: {resp.text.strip()}"

@mcp.tool()
async def get_current_time(timezone: str = "UTC") -> str:
    """Get the current date and time."""
    now = datetime.now()
    return f"Current time: {now.strftime('%Y-%m-%d %H:%M:%S')} ({timezone})"

@mcp.resource("file://config")
async def get_config() -> str:
    """Application configuration."""
    return "debug=true\nlog_level=info\nmax_retries=3"

@mcp.prompt()
async def analyze_data(data: str) -> str:
    """Analyze a dataset and provide insights."""
    return f"Please analyze the following data and provide key insights:\n\n{data}"

# Run the server
if __name__ == "__main__":
    mcp.run()  # starts stdio transport by default
```

Configure it in your MCP host (e.g., Claude Desktop `claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "my-tools": {
      "command": "python",
      "args": ["path/to/server.py"]
    }
  }
}
```

---

## Building an MCP Server (TypeScript)

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({
  name: "my-tools",
  version: "1.0.0"
});

// Define a tool
server.tool(
  "search_issues",
  "Search GitHub issues by query",
  { query: z.string(), repo: z.string() },
  async ({ query, repo }) => {
    const response = await fetch(
      `https://api.github.com/search/issues?q=${query}+repo:${repo}`
    );
    const data = await response.json();
    const results = data.items.slice(0, 5).map(
      (i: any) => `#${i.number}: ${i.title}`
    );
    return { content: [{ type: "text", text: results.join("\n") }] };
  }
);

// Define a resource
server.resource(
  "repo-readme",
  "file://readme",
  async () => ({
    contents: [{ uri: "file://readme", text: "# My Project\n..." }]
  })
);

// Start the server
const transport = new StdioServerTransport();
await server.connect(transport);
```

---

## Why MCP Matters

### Universal Tool Interface
Write a tool once, use it with any MCP-compatible host. No more rewriting integrations for every new AI app.

### Composability
Mix and match servers. Need GitHub + database + Slack? Connect three servers. Need to add file search? Plug in another server. No code changes to your host.

### Ecosystem
The MCP ecosystem is growing fast. Pre-built servers already exist for:
- **Filesystem** — read/write files, search directories
- **GitHub** — issues, PRs, code search
- **PostgreSQL / MySQL** — query databases
- **Slack** — send messages, read channels
- **Google Drive** — read/search documents
- **Brave Search** — web search
- **Docker** — manage containers

---

## MCP vs Direct Function Calling

| Aspect | Direct Function Calling | MCP |
|--------|------------------------|-----|
| **Integration** | Custom per tool | Standardized protocol |
| **Discovery** | Hardcoded in app | Dynamic capability advertisement |
| **Reusability** | Tied to your app | Shareable across any MCP host |
| **Transport** | In-process | stdio, HTTP/SSE (can be remote) |
| **Ecosystem** | Build everything yourself | Growing library of servers |
| **Complexity** | Lower for simple cases | Higher initial setup, lower at scale |

For simple agents with a few tools, direct [[Function Calling and Tool Use|function calling]] is fine. MCP shines when you need **many tools**, **cross-application reuse**, or when you want to leverage the **ecosystem** of pre-built servers.

---

## MCP in the Broader Agent Architecture

MCP fits naturally into the [[Agent Architecture Patterns|agent architecture]] we've discussed:

- The **agent loop** (think → act → observe) remains the same
- MCP standardizes the **"act"** step — how tools are discovered, invoked, and how results return
- Frameworks like [[LangGraph and Agent Frameworks|LangGraph]] can use MCP servers as tool providers
- [[Building an Agent with Spring AI|Spring AI]] can connect to MCP servers for tool discovery

MCP doesn't replace agents — it gives them a **universal plug system** for tools.

---

## Key Takeaways

1. MCP is a **universal protocol** for connecting LLMs to tools — like USB for AI applications
2. Architecture is three layers: **Host** (your app) → **Client** (protocol handler) → **Server** (tool provider)
3. Three primitives: **Tools** (actions), **Resources** (data to read), **Prompts** (reusable templates)
4. Building an MCP server is simple — decorate functions with `@mcp.tool()` and run the server
5. The key value is **write once, use everywhere** — a tool server works with any MCP-compatible host
6. MCP enables **composability** — plug and unplug capabilities without changing your application code
7. The ecosystem is growing fast — check for existing servers before building your own (GitHub, databases, search, etc.)
