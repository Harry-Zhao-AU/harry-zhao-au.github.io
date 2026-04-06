---
layout: post
title: "I Built a GraphRAG App. The Hard Part Wasn't the AI."
date: 2026-04-06
categories: [engineering]
tags: [ai, graphrag, neo4j, llamaindex, mcp, embeddings]
excerpt: "I built an HR skill graph with Neo4j, LlamaIndex, and an MCP server. The LLM was the easy part. How you treat your data is what actually matters."
---

Everyone's building AI apps. Most of them are wrappers around an LLM call with a prompt and a prayer. I wanted to build something that actually requires engineering — where the AI is one component, not the whole thing.

So I built [ResumeGraph](https://github.com/Harry-Zhao-AU/ResumeGraph): an HR tool that ingests employee resumes, extracts skills and relationships into a knowledge graph, and lets you query it with natural language. It connects to my agent framework [NanoBotTS](https://github.com/Harry-Zhao-AU/NanoBotTS) via both HTTP and MCP.

The LLM integration took an afternoon. Getting the data right took everything else.

## What It Does

Upload resume PDFs. The system extracts who knows what, where they worked, what they studied — and stores it all as a graph:

```
(:Employee) ──HAS_SKILL──▶ (:Skill)
(:Employee) ──WORKED_AT──▶ (:Company)
(:Employee) ──STUDIED_AT──▶ (:University)
(:Employee) ──HAS_CERTIFICATION──▶ (:Certification)
(:Employee) ──LOCATED_IN──▶ (:City)
(:Skill) ──RELATED_TO──▶ (:Skill)
```

Then you can ask:

```
"Who knows both Python and Docker?"
"Find AWS-certified engineers in Melbourne"
"Who is most similar to Daniel Chen?"
"What's the skill gap between these two engineers?"
```

The graph handles structural queries (exact relationships). Embeddings handle semantic queries (related skills). The LLM translates natural language into Cypher. Three layers, each solving a different problem.

## The Architecture

```
Resume PDFs
    │
    ▼
PyMuPDFReader (extract text — no LLM)
    │
    ▼
SimpleLLMPathExtractor (LLM extracts entities)
    │   "Priya Nair, HAS_SKILL, Java"
    │   "Priya Nair, WORKED_AT, Canva"
    │   "Priya Nair, LOCATED_IN, Sydney"
    │
    ▼
Neo4j (graph store — 302 nodes, 1,270 edges)
    │
    ▼
sentence-transformers (embed skill names locally)
    │   compute cosine similarity between all skills
    │   Docker ↔ Kubernetes: 0.95 → RELATED_TO edge
    │
    ▼
FastAPI (2 endpoints: structured search + natural language)
    │
    ├──▶ MCP Server (stdio wrapper for Claude Code)
    ├──▶ NanoBotTS (HTTP tool integration)
    └──▶ curl / browser
```

**Tech stack:** Python, Neo4j, LlamaIndex, FastAPI, sentence-transformers, MCP SDK.

## Why GraphRAG, Not Just RAG

Plain RAG is: chunk documents → embed chunks → vector search → feed results to LLM.

It works for "find me something similar to this query." It fails at:

- **"Who knows both Kafka AND AWS?"** — that's a graph intersection, not a similarity search
- **"What's the skill gap between these two people?"** — that's a set difference across graph paths
- **"Find everyone who worked at Atlassian and now knows Kubernetes"** — that's a multi-hop traversal

GraphRAG combines both approaches:

| Query type | Who handles it |
|-----------|----------------|
| Exact: "who has Python?" | Neo4j graph traversal |
| Semantic: "who does monitoring?" | Embeddings find "monitoring" → "monitoring and alerting" via RELATED_TO |
| Natural language: "best candidates for a microservices role" | LLM generates Cypher from your question + graph schema |

The graph gives you **structure**. The embeddings give you **fuzzy matching**. The LLM gives you **natural language interface**. Each layer handles what it's good at.

## The Hard Part: Data, Not AI

Here's what I learned building this.

### Entity Extraction Is Messy

The LLM reads a resume and extracts triplets like `(Priya Nair, HAS_SKILL, Java)`. Sounds simple. But:

- One resume says "Docker", another says "docker containers", another says "Docker/Kubernetes" — are these the same skill?
- "Senior Backend Engineer" and "Sr. Backend Eng." — same role or different nodes?
- The LLM confidently extracts "Python" as a skill from a resume that only mentions it in a certification name

I started with `SchemaLLMPathExtractor` for structured extraction, but Azure OpenAI rejected the JSON schema LlamaIndex generated. Switched to `SimpleLLMPathExtractor` with a custom prompt — more flexible, but loses entity properties like skill level and years. Trade-offs everywhere.

### Graph Schema Design Matters More Than Model Choice

I spent more time designing the 6 node types and 6 relationship types than on any LLM configuration. Get the schema wrong and your queries return nonsense — no model upgrade fixes a bad graph structure.

The key decision: making `(:Skill) ──RELATED_TO──▶ (:Skill)` a computed edge based on embedding similarity rather than an LLM-extracted relationship. This is more reliable — the LLM might not know that "Docker" and "Kubernetes" are related, but their embeddings (cosine similarity 0.95) definitely do.

### Embeddings Are the Unsung Hero

Everyone talks about LLMs. Nobody talks about embeddings. In this project, embeddings do the critical work:

- `sentence-transformers/all-MiniLM-L6-v2` — a tiny local model, runs in milliseconds
- Embeds each skill name into a 384-dimension vector
- Computes cosine similarity between all skill pairs
- Creates RELATED_TO edges where similarity exceeds 0.65
- Result: 94 new edges that make semantic skill matching possible

When someone searches for "monitoring," the graph traverses RELATED_TO to also find "monitoring and alerting." No LLM call needed — just graph traversal over pre-computed similarity.

This is cheaper, faster, and more reliable than asking an LLM "what skills are related to monitoring?" every time someone queries.

### Your Data Pipeline Is Your Moat

The whole ingestion pipeline:

```
PDF → raw text (no LLM) → entity extraction (LLM) → graph store → embeddings (local model) → similarity edges
```

Each step transforms the data into a more useful shape. The LLM is involved in exactly one step. The rest is traditional data engineering — parsing, deduplication, schema design, similarity computation.

**The quality of your AI app is determined by how you treat your data before it reaches the LLM.** This is the same principle as context engineering in Claude Code: what you feed the model matters more than which model you use.

## MCP: Making It Agent-Accessible

The MCP server is intentionally thin — a stdio wrapper that translates MCP tool calls into HTTP requests to the FastAPI server:

```
Claude Code / Claude Desktop
    → MCP server (stdio, JSON-RPC)
        → HTTP request to FastAPI (localhost:3100)
            → LlamaIndex + Neo4j
```

Two tools:
- `search_employees` — structured search (skill, company, city, certification)
- `query_graph` — natural language, LLM generates Cypher

The design choice: **API first, MCP second.** The FastAPI server is the real backend. The MCP server is one of several clients — NanoBotTS connects via HTTP, Claude Code connects via MCP, and you can use curl directly. Same data, multiple interfaces.

This is important. If you build MCP-first, you're locked into one consumption pattern. If you build API-first, MCP is just another adapter — exactly like the Channel pattern I learned building [NanoBotTS](/blog/2026/04/05/built-my-own-ai-agent-from-scratch/).

## The Takeaway

Building a GraphRAG app taught me something that applies to all AI engineering:

**The model is a commodity. Your data pipeline is the product.**

Anyone can call an LLM API. The engineering is everything around it:
- How you extract structure from unstructured data
- How you design your graph schema
- How you compute and store embeddings
- How you handle the messy edge cases the LLM gets wrong
- How you expose it all through clean interfaces

This is why I believe AI won't replace engineers. The LLM is one component in a system. Someone still has to design the pipeline, make the schema decisions, handle the data quality issues, and build the infrastructure that makes it all work.

That someone is an engineer.

---

*ResumeGraph is on GitHub: [github.com/Harry-Zhao-AU/ResumeGraph](https://github.com/Harry-Zhao-AU/ResumeGraph). It integrates with [NanoBotTS](https://github.com/Harry-Zhao-AU/NanoBotTS) — see [how I built that from scratch](/blog/2026/04/05/built-my-own-ai-agent-from-scratch/).*
