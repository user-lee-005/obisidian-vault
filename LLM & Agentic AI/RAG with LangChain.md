---
title: RAG with LangChain
date: 2025-07-18
tags:
  - llm
  - rag
  - langchain
  - python
  - vector-databases
aliases:
  - LangChain RAG
  - RAG Pipeline LangChain
status: complete
---

# RAG with LangChain

> [!abstract] TL;DR
> A hands-on, code-first guide to building [[RAG - Retrieval Augmented Generation]] pipelines in Python using [[LangChain Fundamentals]]. Every section has runnable code. Assumes you already understand *what* RAG is — this is about *how* to build it.

---

## 1. Setup

```bash
pip install langchain langchain-openai langchain-community langchain-chroma \
  langchain-huggingface faiss-cpu chromadb pypdf unstructured tiktoken \
  langchain-pinecone pinecone-client pgvector psycopg2-binary \
  ragas langsmith fastapi uvicorn
```

```python
import os
os.environ["OPENAI_API_KEY"] = "sk-..."
os.environ["LANGCHAIN_TRACING_V2"] = "true"          # optional: LangSmith
os.environ["LANGCHAIN_API_KEY"] = "ls__..."           # optional: LangSmith
```

**Recommended project structure:**

```
rag-project/
├── data/                  # raw documents
├── vectorstore/           # persisted index
├── chains/
│   ├── rag_chain.py       # core chain
│   └── conversational.py  # chat variant
├── loaders/
│   └── custom_loader.py
├── app.py                 # FastAPI wrapper
└── eval/
    └── evaluate.py
```

---

## 2. Document Loading

LangChain ships dozens of loaders. Each returns a list of `Document(page_content, metadata)`.

### PDF

```python
from langchain_community.document_loaders import PyPDFLoader

loader = PyPDFLoader("data/report.pdf")
docs = loader.load()            # one Document per page
print(docs[0].metadata)         # {'source': 'data/report.pdf', 'page': 0}
```

### Web Pages

```python
from langchain_community.document_loaders import WebBaseLoader

loader = WebBaseLoader("https://docs.smith.langchain.com/overview")
docs = loader.load()
```

### Entire Folders

```python
from langchain_community.document_loaders import DirectoryLoader

loader = DirectoryLoader("data/", glob="**/*.pdf", loader_cls=PyPDFLoader)
docs = loader.load()            # loads every PDF recursively
```

### Anything (Unstructured)

```python
from langchain_community.document_loaders import UnstructuredFileLoader

loader = UnstructuredFileLoader("data/messy_doc.docx")
docs = loader.load()
```

### CSV & JSON

```python
from langchain_community.document_loaders import CSVLoader, JSONLoader

csv_docs = CSVLoader("data/shipments.csv").load()

json_docs = JSONLoader(
    file_path="data/events.json",
    jq_schema=".events[]",
    text_content=False,
).load()
```

### Custom Loader (API / DB)

```python
from langchain_core.documents import Document

def load_from_api(url: str) -> list[Document]:
    import requests
    data = requests.get(url).json()
    return [
        Document(page_content=item["text"], metadata={"id": item["id"]})
        for item in data["results"]
    ]
```

> [!tip] Metadata matters
> Always propagate useful metadata (source, page, date). You'll need it later for filtering and citations.

---

## 3. Text Splitting / Chunking

Raw documents are usually too long for a single embedding. Splitting them into chunks is where most RAG quality gains (or losses) happen.

### RecursiveCharacterTextSplitter (default choice)

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=["\n\n", "\n", ". ", " ", ""],
)
chunks = splitter.split_documents(docs)
```

### TokenTextSplitter (token-aware)

```python
from langchain_text_splitters import TokenTextSplitter

splitter = TokenTextSplitter(chunk_size=512, chunk_overlap=64)
chunks = splitter.split_documents(docs)
```

### MarkdownHeaderTextSplitter

```python
from langchain_text_splitters import MarkdownHeaderTextSplitter

headers = [("#", "h1"), ("##", "h2"), ("###", "h3")]
splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers)
chunks = splitter.split_text(markdown_text)   # preserves heading hierarchy in metadata
```

### SemanticChunker (embedding-based)

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

splitter = SemanticChunker(
    OpenAIEmbeddings(),
    breakpoint_threshold_type="percentile",
)
chunks = splitter.split_documents(docs)
```

> [!warning] Tuning chunk_size & chunk_overlap
> - **Small chunks** (200-500 tokens): better precision, more noise
> - **Large chunks** (800-1500 tokens): more context, may dilute relevance
> - **Overlap** (10-20% of chunk_size): prevents splitting mid-sentence
> - Start with `chunk_size=1000, chunk_overlap=200` and iterate with evaluation.

---

## 4. Embeddings

Embeddings convert text into vectors. Pick one provider and stick with it — mixing models means incompatible vector spaces.

### OpenAI (best quality per dollar)

```python
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")   # 1536 dims, cheap
# embeddings = OpenAIEmbeddings(model="text-embedding-3-large") # 3072 dims, better
```

### HuggingFace (free, local)

```python
from langchain_huggingface import HuggingFaceEmbeddings

embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/all-MiniLM-L6-v2"
)
```

### Ollama (local with Ollama server)

```python
from langchain_community.embeddings import OllamaEmbeddings

embeddings = OllamaEmbeddings(model="nomic-embed-text")
```

### Caching Embeddings (save cost on re-runs)

```python
from langchain.embeddings import CacheBackedEmbeddings
from langchain.storage import LocalFileStore

store = LocalFileStore("./embedding_cache/")
cached_embeddings = CacheBackedEmbeddings.from_bytes_store(
    underlying_embeddings=OpenAIEmbeddings(model="text-embedding-3-small"),
    document_embedding_cache=store,
    namespace="text-embedding-3-small",
)
```

---

## 5. Vector Stores

See [[Vector Databases]] for deeper theory. Here we focus on practical setup with LangChain.

### Chroma (easiest for development)

```python
from langchain_chroma import Chroma

# In-memory (ephemeral)
vectorstore = Chroma.from_documents(chunks, embeddings)

# Persistent (survives restarts)
vectorstore = Chroma.from_documents(
    chunks, embeddings, persist_directory="./vectorstore/chroma_db"
)

# Load existing
vectorstore = Chroma(
    persist_directory="./vectorstore/chroma_db",
    embedding_function=embeddings,
)
```

#### CRUD Operations

```python
# Add
vectorstore.add_documents(new_chunks)

# Search
results = vectorstore.similarity_search("shipping delay causes", k=4)
results_with_scores = vectorstore.similarity_search_with_score("query", k=4)

# Filter by metadata
results = vectorstore.similarity_search(
    "delays", k=4, filter={"source": "data/q4_report.pdf"}
)

# Delete
vectorstore.delete(ids=["doc-id-1", "doc-id-2"])
```

### FAISS (fast, local, no server)

```python
from langchain_community.vectorstores import FAISS

vectorstore = FAISS.from_documents(chunks, embeddings)

# Save / Load
vectorstore.save_local("./vectorstore/faiss_index")
vectorstore = FAISS.load_local(
    "./vectorstore/faiss_index", embeddings,
    allow_dangerous_deserialization=True,
)
```

### Pinecone (managed, production)

```python
from langchain_pinecone import PineconeVectorStore
from pinecone import Pinecone

pc = Pinecone(api_key=os.environ["PINECONE_API_KEY"])
index = pc.Index("rag-index")

vectorstore = PineconeVectorStore(index=index, embedding=embeddings)
vectorstore.add_documents(chunks)

# Metadata filtering
results = vectorstore.similarity_search(
    "freight costs", k=5, filter={"region": "APAC"}
)
```

### pgvector (PostgreSQL)

```python
from langchain_community.vectorstores import PGVector

CONNECTION = "postgresql+psycopg2://user:pass@localhost:5432/ragdb"

vectorstore = PGVector.from_documents(
    chunks, embeddings,
    connection_string=CONNECTION,
    collection_name="shipment_docs",
)
```

---

## 6. Building the RAG Chain (LCEL)

This is the core. LangChain Expression Language (LCEL) makes chains composable and streamable.

### Simple RAG Chain

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain_openai import ChatOpenAI

retriever = vectorstore.as_retriever(search_kwargs={"k": 4})
model = ChatOpenAI(model="gpt-4o-mini", temperature=0)

prompt = ChatPromptTemplate.from_template("""
Answer the question based only on the following context.
If you cannot answer from the context, say "I don't know."

Context:
{context}

Question: {question}
""")

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | model
    | StrOutputParser()
)

# Use it
answer = rag_chain.invoke("What caused the Q4 shipping delays?")
print(answer)
```

### With Source Documents

```python
from langchain_core.runnables import RunnableParallel

rag_chain_with_sources = RunnableParallel(
    {"context": retriever, "question": RunnablePassthrough()}
).assign(
    answer=lambda x: (
        prompt.invoke({"context": format_docs(x["context"]), "question": x["question"]})
        | model
        | StrOutputParser()
    ).invoke(x["question"])
)

# Simpler approach — just return both
from langchain_core.runnables import RunnablePassthrough

def retrieve_and_answer(question: str):
    docs = retriever.invoke(question)
    context = format_docs(docs)
    answer = (prompt | model | StrOutputParser()).invoke(
        {"context": context, "question": question}
    )
    return {"answer": answer, "sources": [d.metadata for d in docs]}
```

### With Streaming

```python
for chunk in rag_chain.stream("What are the top freight routes?"):
    print(chunk, end="", flush=True)
```

### Custom Prompt Template

```python
logistics_prompt = ChatPromptTemplate.from_template("""
You are a logistics domain expert. Use the retrieved documents to answer
the user's question. Cite specific data points when possible.
If the answer is not in the context, say so.

Retrieved Documents:
{context}

User Question: {question}

Answer:
""")
```

---

## 7. Conversational RAG (Chat with History)

The problem: a user asks *"What about their pricing?"* — "their" refers to a company mentioned 3 turns ago. A plain retriever won't know what "their" means.

### History-Aware Retriever

```python
from langchain.chains import create_history_aware_retriever
from langchain_core.prompts import MessagesPlaceholder

contextualize_prompt = ChatPromptTemplate.from_messages([
    ("system", "Given the chat history and latest user question, "
               "reformulate the question to be standalone."),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
])

history_aware_retriever = create_history_aware_retriever(
    model, retriever, contextualize_prompt
)
```

### Full Conversational Chain

```python
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain

qa_prompt = ChatPromptTemplate.from_messages([
    ("system", "Answer questions using the context below.\n\n{context}"),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
])

question_answer_chain = create_stuff_documents_chain(model, qa_prompt)
conversational_rag = create_retrieval_chain(
    history_aware_retriever, question_answer_chain
)

# Usage
from langchain_core.messages import HumanMessage, AIMessage

chat_history = []

r1 = conversational_rag.invoke({
    "input": "Tell me about Maersk's shipping routes",
    "chat_history": chat_history,
})
chat_history.extend([HumanMessage(content="Tell me about Maersk's shipping routes"),
                     AIMessage(content=r1["answer"])])

r2 = conversational_rag.invoke({
    "input": "What about their pricing?",        # "their" = Maersk
    "chat_history": chat_history,
})
print(r2["answer"])
```

### Session-Based Memory

```python
from collections import defaultdict

session_store: dict[str, list] = defaultdict(list)

def chat(session_id: str, question: str) -> str:
    result = conversational_rag.invoke({
        "input": question,
        "chat_history": session_store[session_id],
    })
    session_store[session_id].extend([
        HumanMessage(content=question),
        AIMessage(content=result["answer"]),
    ])
    return result["answer"]
```

---

## 8. Advanced Patterns

See [[Advanced RAG Patterns]] for deeper dives on each.

### Multi-Query Retriever

Generates multiple query variations to improve recall.

```python
from langchain.retrievers.multi_query import MultiQueryRetriever

multi_retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(),
    llm=model,
)
docs = multi_retriever.invoke("freight cost optimization strategies")
```

### Contextual Compression

Compresses retrieved docs to only the relevant portions.

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import LLMChainExtractor

compressor = LLMChainExtractor.from_llm(model)
compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=retriever,
)
```

### Ensemble Retriever (Dense + BM25)

Combines semantic search with keyword search for better coverage.

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

bm25 = BM25Retriever.from_documents(chunks, k=4)
dense = vectorstore.as_retriever(search_kwargs={"k": 4})

ensemble = EnsembleRetriever(
    retrievers=[bm25, dense],
    weights=[0.4, 0.6],
)
```

### Self-Query Retriever

The LLM writes metadata filters from natural language.

```python
from langchain.retrievers.self_query.base import SelfQueryRetriever
from langchain.chains.query_constructor.schema import AttributeInfo

metadata_field_info = [
    AttributeInfo(name="source", description="PDF filename", type="string"),
    AttributeInfo(name="page", description="Page number", type="integer"),
    AttributeInfo(name="region", description="Geographic region", type="string"),
]

self_query = SelfQueryRetriever.from_llm(
    llm=model,
    vectorstore=vectorstore,
    document_contents="Logistics and shipping reports",
    metadata_field_info=metadata_field_info,
)
# "Show me APAC reports from page 5 onwards" → auto-generates filters
```

### Parent Document Retriever

Embed small chunks for precision, retrieve the full parent document for context.

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore

parent_splitter = RecursiveCharacterTextSplitter(chunk_size=2000)
child_splitter = RecursiveCharacterTextSplitter(chunk_size=400)

store = InMemoryStore()
parent_retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=store,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)
parent_retriever.add_documents(docs)
```

---

## 9. Evaluation

You can't improve what you can't measure.

### LangSmith Tracing

```python
# Set env vars (from Section 1), then every chain call is automatically traced
# View traces at https://smith.langchain.com
```

### RAGAS Integration

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision
from datasets import Dataset

eval_data = {
    "question": ["What caused Q4 delays?", "Top 3 freight routes?"],
    "answer": [rag_chain.invoke(q) for q in ["What caused Q4 delays?", "Top 3 freight routes?"]],
    "contexts": [[d.page_content for d in retriever.invoke(q)]
                 for q in ["What caused Q4 delays?", "Top 3 freight routes?"]],
    "ground_truth": ["Port congestion and labor shortages", "Shanghai-LA, Rotterdam-NYC, Singapore-Sydney"],
}

dataset = Dataset.from_dict(eval_data)
result = evaluate(dataset, metrics=[faithfulness, answer_relevancy, context_precision])
print(result)
```

### Automated Evaluation Pipeline

```python
def evaluate_rag(test_cases: list[dict]) -> dict:
    """Run RAG chain over test cases and compute metrics."""
    results = []
    for case in test_cases:
        answer = rag_chain.invoke(case["question"])
        docs = retriever.invoke(case["question"])
        results.append({
            "question": case["question"],
            "expected": case["expected_answer"],
            "actual": answer,
            "num_docs": len(docs),
        })
    return results
```

---

## 10. Production Deployment

### FastAPI Wrapper

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Query(BaseModel):
    question: str
    session_id: str = "default"

class Answer(BaseModel):
    answer: str
    sources: list[dict]

@app.post("/ask", response_model=Answer)
async def ask(query: Query):
    docs = await retriever.ainvoke(query.question)
    context = format_docs(docs)
    answer = await (prompt | model | StrOutputParser()).ainvoke(
        {"context": context, "question": query.question}
    )
    return Answer(
        answer=answer,
        sources=[d.metadata for d in docs],
    )
```

```bash
uvicorn app:app --host 0.0.0.0 --port 8000
```

### Streaming Endpoint

```python
from fastapi.responses import StreamingResponse

@app.post("/ask/stream")
async def ask_stream(query: Query):
    async def generate():
        async for chunk in rag_chain.astream(query.question):
            yield chunk
    return StreamingResponse(generate(), media_type="text/plain")
```

### Caching Layer

```python
from langchain_community.cache import SQLiteCache
from langchain_core.globals import set_llm_cache

set_llm_cache(SQLiteCache(database_path=".langchain_cache.db"))
# Identical prompts now return cached responses instantly
```

### Error Handling and Retries

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=1, max=10))
def safe_invoke(question: str) -> str:
    return rag_chain.invoke(question)
```

### Monitoring with Callbacks

```python
from langchain_core.callbacks import StdOutCallbackHandler

answer = rag_chain.invoke(
    "What happened in Q4?",
    config={"callbacks": [StdOutCallbackHandler()]},
)
```

---

## Key Takeaways

1. **Start simple** — `RecursiveCharacterTextSplitter` + Chroma + OpenAI embeddings gets you 80% of the way.
2. **Metadata is your friend** — attach source, page, date to every chunk for filtering and citations.
3. **Cache embeddings** — `CacheBackedEmbeddings` prevents re-computing vectors on every restart.
4. **LCEL is the glue** — `retriever | prompt | model | parser` composes cleanly and streams natively.
5. **Conversational RAG needs question reformulation** — `create_history_aware_retriever` solves the pronoun/context problem.
6. **Combine retrieval strategies** — `EnsembleRetriever` (BM25 + dense) almost always beats a single retriever.
7. **Evaluate early** — RAGAS + LangSmith tracing before you optimize blindly.
8. **Production = async + caching + retries** — wrap your chain in FastAPI with `ainvoke` and a cache layer.
9. **Chunk tuning is the #1 lever** — experiment with size, overlap, and splitting strategy before adding complexity.
10. **Use [[Advanced RAG Patterns]] when simple RAG hits a ceiling** — parent docs, self-query, and multi-vector retrievers unlock the next level.
