---
tags:
  - llm
  - langchain
  - python
  - phase-4
---

# LangChain Fundamentals

Even as Java developers, understanding LangChain is valuable — it's the most popular LLM framework, its patterns influence all others (including [[Spring AI Framework]]), and many tutorials, papers, and prototypes use it. Let's learn the concepts and see how they translate.

---

## What is LangChain?

LangChain is a Python (and JavaScript) framework that abstracts common patterns in LLM application development. Instead of writing boilerplate for chaining prompts, managing memory, calling tools, and retrieving documents — LangChain gives you composable building blocks.

**Why it exists:**
- Everyone was writing the same glue code (prompt → model → parse → act)
- Memory management across conversations is tricky
- RAG pipelines have many moving parts
- Tool/function calling needs orchestration

---

## Core Concepts

### Models — LLM Wrappers

LangChain wraps different providers behind a consistent interface:

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic
from langchain_community.llms import Ollama

# All share the same interface
llm = ChatOpenAI(model="gpt-4o", temperature=0.7)
# llm = ChatAnthropic(model="claude-3-sonnet-20240229")
# llm = Ollama(model="llama3")
```

Swap providers without changing your application logic — same idea as Spring AI's `ChatModel` abstraction.

### Prompts — Templates and Composition

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a {domain} expert. Be concise."),
    ("user", "{question}")
])

# Fill in the variables
messages = prompt.invoke({
    "domain": "logistics",
    "question": "What causes port congestion?"
})
```

**Few-shot templates** let you include examples directly:

```python
from langchain_core.prompts import FewShotChatMessagePromptTemplate

examples = [
    {"input": "FCL", "output": "Full Container Load - entire container for one shipper"},
    {"input": "LCL", "output": "Less than Container Load - shared container space"},
]
```

### Output Parsers — Structure the Response

```python
from langchain_core.output_parsers import StrOutputParser, JsonOutputParser
from langchain_core.pydantic_v1 import BaseModel, Field

class ShipmentRisk(BaseModel):
    level: str = Field(description="LOW, MEDIUM, or HIGH")
    reason: str = Field(description="Primary risk factor")
    mitigation: list[str] = Field(description="Steps to mitigate")

parser = JsonOutputParser(pydantic_object=ShipmentRisk)
```

This is the Python equivalent of Spring AI's `BeanOutputParser` — map LLM text to typed objects.

### Chains — LCEL (LangChain Expression Language)

Here's where LangChain gets elegant. You compose steps with the pipe (`|`) operator:

```python
chain = prompt | llm | parser
```

That reads as: fill the prompt → send to the LLM → parse the output. Each step's output becomes the next step's input.

```python
result = chain.invoke({
    "domain": "freight forwarding",
    "question": "Assess risk of shipping to a congested port"
})
# result is a ShipmentRisk object
```

---

## Memory

LLMs are stateless — they don't remember previous messages unless you explicitly pass them. Memory solves this for multi-turn conversations.

### ConversationBufferMemory

Stores *everything*. Simple but eventually hits the context window limit.

```python
from langchain.memory import ConversationBufferMemory

memory = ConversationBufferMemory(return_messages=True)
memory.save_context(
    {"input": "What's my shipment status?"},
    {"output": "Your shipment TRK-123 is in transit, ETA March 15."}
)
```

### ConversationSummaryMemory

Summarizes older messages to stay within token limits. Great for long conversations:

```python
from langchain.memory import ConversationSummaryMemory

memory = ConversationSummaryMemory(llm=llm)
# Older messages get compressed into summaries automatically
```

### Why Memory Matters

Without memory, every API call is independent. The user says "What about the second one?" and the model has no idea what "the second one" refers to. Memory provides the conversational context that makes interactions feel natural.

> [!note] In Spring AI
> Spring AI handles this with `MessageChatMemoryAdvisor` — same concept, different implementation. See [[Spring AI Framework]].

---

## Retrieval (RAG Components)

LangChain provides the full [[RAG - Retrieval Augmented Generation|RAG]] pipeline as composable pieces:

### Document Loaders

```python
from langchain_community.document_loaders import (
    PyPDFLoader,
    WebBaseLoader,
    TextLoader
)

# Load from various sources
docs = PyPDFLoader("shipping_manual.pdf").load()
web_docs = WebBaseLoader("https://docs.example.com/api").load()
```

### Text Splitters

Documents need to be chunked for embedding. Too big = diluted meaning. Too small = lost context.

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=["\n\n", "\n", ". ", " "]
)

chunks = splitter.split_documents(docs)
```

### Vector Stores

Store embeddings and search by similarity. See [[Vector Databases]] for deep coverage.

```python
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = FAISS.from_documents(chunks, embeddings)

# Search
results = vectorstore.similarity_search("port congestion causes", k=4)
```

### Retrieval Chains

Put it all together — question → retrieve → generate:

```python
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

retriever = vectorstore.as_retriever(search_kwargs={"k": 4})

rag_prompt = ChatPromptTemplate.from_messages([
    ("system", "Answer based on the following context:\n\n{context}"),
    ("user", "{question}")
])

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | rag_prompt
    | llm
    | StrOutputParser()
)

answer = rag_chain.invoke("What are the main causes of shipping delays?")
```

---

## Complete Code Examples

### Simple Chain

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

llm = ChatOpenAI(model="gpt-4o", temperature=0.7)

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a concise logistics expert."),
    ("user", "{question}")
])

chain = prompt | llm | StrOutputParser()

response = chain.invoke({"question": "Explain demurrage in 2 sentences"})
print(response)
```

### RAG Chain with Document Retrieval

```python
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import Chroma
from langchain_community.document_loaders import PyPDFLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

# 1. Load and chunk
docs = PyPDFLoader("freight_handbook.pdf").load()
splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
chunks = splitter.split_documents(docs)

# 2. Embed and store
vectorstore = Chroma.from_documents(chunks, OpenAIEmbeddings())
retriever = vectorstore.as_retriever()

# 3. Build RAG chain
prompt = ChatPromptTemplate.from_messages([
    ("system", "Answer using ONLY the provided context. If unsure, say so.\n\nContext: {context}"),
    ("user", "{question}")
])

chain = (
    {"context": retriever | (lambda docs: "\n".join(d.page_content for d in docs)),
     "question": RunnablePassthrough()}
    | prompt
    | ChatOpenAI(model="gpt-4o")
    | StrOutputParser()
)

answer = chain.invoke("What are Incoterms 2020?")
```

---

## LangChain vs Doing It Manually

**When LangChain helps:**
- Complex chains with multiple steps (retrieval → reranking → generation)
- You need memory management across conversations
- Rapid prototyping — get something working fast
- You want tracing/observability (LangSmith integration)

**When it hurts:**
- Simple single API calls — the abstraction adds complexity for no benefit
- You need full control over HTTP requests and retry logic
- Version churn — LangChain's API changes frequently (the v0.1 → v0.2 migration was painful)
- Debugging — abstraction layers make it harder to see what's actually being sent to the model

> [!tip] The pragmatic approach
> Use LangChain for prototyping and complex orchestration. For production with simple flows, consider calling APIs directly or using [[Spring AI Framework]] which is more stable.

---

## Key Takeaways

1. **LangChain** abstracts common LLM patterns — prompts, models, parsers, memory, retrieval — into composable pieces
2. **LCEL** (the pipe `|` operator) lets you chain steps declaratively: `prompt | model | parser`
3. **Memory** solves the statelessness problem — `ConversationBufferMemory` for short chats, `ConversationSummaryMemory` for long ones
4. **RAG in LangChain** = Document Loaders → Text Splitters → Vector Store → Retrieval Chain
5. The concepts translate directly to [[Spring AI Framework]] — `ChatClient`, `QuestionAnswerAdvisor`, `VectorStore`
6. LangChain is **great for prototyping**, but evaluate whether the abstraction is worth it in production
7. Understanding LangChain patterns helps you read the majority of LLM tutorials and papers out there
