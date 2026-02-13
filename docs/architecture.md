# Architecture Deep Dive

This document provides an in-depth look at the architectural decisions, system design, and engineering trade-offs behind AI Secretary.

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Why LangGraph StateGraph](#why-langgraph-stategraph)
3. [Graph Nodes — 18 Node Roles](#graph-nodes--18-node-roles)
4. [Search System — 6-Source Integration](#search-system--6-source-integration)
5. [RAG Pipeline](#rag-pipeline)
6. [Security Mode — Hybrid LLM Routing](#security-mode--hybrid-llm-routing)
7. [Modularization Strategy](#modularization-strategy)
8. [Dual Deployment Design](#dual-deployment-design)
9. [Streaming Architecture](#streaming-architecture)
10. [State Management & Checkpointing](#state-management--checkpointing)

---

## System Overview

AI Secretary is a multi-mode LLM agent system with 4 operating modes:

| Mode | Trigger | Pipeline | Avg. Time |
|------|---------|----------|-----------|
| **General Chat** | Simple questions, greetings | Gatekeeper → SearchChecker → GeneralChat | 1-5 sec |
| **Deep Research** | Complex analysis requests | Gatekeeper → Planner → ... → Publisher (11 nodes) | 2-4 min |
| **Code** | Programming questions | Gatekeeper → CodeChat → CodeFactCheck | 5-15 sec |
| **Meeting** | Audio file upload | Direct upload → Diarization → Summary | 1-3 min |

The **Gatekeeper Router** at the entry point uses an LLM to classify incoming queries and route them to the appropriate pipeline, eliminating unnecessary computation for simple queries.

---

## Why LangGraph StateGraph

### The Problem with ReAct Agents

ReAct (Reasoning + Acting) agents dynamically decide which tools to call at each step. While flexible, this approach has critical limitations for complex pipelines:

| Issue | ReAct Agent | StateGraph |
|-------|-------------|------------|
| **Execution Order** | Non-deterministic | Guaranteed node sequence |
| **Retry Logic** | Difficult to implement | Built-in conditional edges |
| **Progress Tracking** | Opaque (tool calls are hidden) | Explicit node transitions → SSE progress |
| **Debugging** | Hard to reproduce | Deterministic, reproducible |
| **State Persistence** | Manual | Built-in checkpointing |
| **Cost Control** | Can loop endlessly | Bounded by graph edges |

### Why StateGraph Fits

The deep research pipeline requires:
1. **Sequential guarantees**: Search must complete before evaluation
2. **Conditional branching**: Quality check passes → write, fails → re-search
3. **Bounded iteration**: Maximum 3 retry loops (Evaluator → Strategist → Miner)
4. **Observable state**: Frontend needs step-by-step progress updates via SSE
5. **Resumability**: Long-running research can survive server restarts

StateGraph provides all of this with explicit edge definitions, making the system behavior predictable and debuggable.

---

## Graph Nodes — 18 Node Roles

### Entry & Routing (2 nodes)

| Node | Role | Key Logic |
|------|------|-----------|
| **Gatekeeper Router** | Classifies query intent | LLM-based routing → general_chat / deep_research / code_chat / confirmation |
| **Search Checker** | Decides if search is needed | Detects tool triggers (weather/translate), search keywords, or direct-answerable |

### General Chat Path (4 nodes)

| Node | Role | Key Logic |
|------|------|-----------|
| **Simple Search** | Quick web search | Serper primary → Tavily fallback |
| **Tool Executor** | Runs real-time tools | Weather API / Papago translation / Geocoding |
| **General Chat** | Main conversation node | RAG context injection, quick fact-check, follow-up suggestions |
| **Deep Research Confirmation** | Confirms before expensive research | Shows estimated time, asks user confirmation |

### Deep Research Pipeline (11 nodes)

| Node | Role | Key Logic |
|------|------|-----------|
| **Planner** | Research planning | Analyzes intent, selects mode template, generates refined topic |
| **Outliner** | Structure generation | Creates 4-6 section outline with search queries per section |
| **Miner** | Parallel search | Searches 6 sources concurrently, collects 20-30 results |
| **Selector** | Source evaluation | Trust scoring (domain tiers), URL validation, dedup, top-K |
| **Deep Reader** | Content extraction | Full-text via Crawl4AI, long-text summarization |
| **Evaluator** | Quality assessment | Scores coverage, depth, diversity, recency → PASS or RETRY |
| **Strategist** | Gap analysis | Identifies missing topics, generates targeted re-search queries |
| **Writer** | Report generation | Structured markdown, audience-adaptive tone, section citations |
| **Fact Checker** | Claim verification | Cross-references assertions against source materials |
| **Publisher** | Output formatting | Source accordions, follow-up questions, final formatting |
| **Librarian** | Knowledge storage | Stores research in RAG vector store for future retrieval |

### Code Path (2 nodes)

| Node | Role | Key Logic |
|------|------|-----------|
| **Code Chat** | Code generation | Local (Qwen 2.5 Coder) or cloud (Gemini) model |
| **Code Fact Check** | Code verification | Extracts libraries/APIs → searches official docs + GitHub → validates |

---

## Search System — 6-Source Integration

### Source Selection Strategy

Different query types benefit from different search sources:

| Query Type | Primary Sources | Why |
|-----------|----------------|-----|
| **General/News** | Serper + Naver News | Google coverage + Korean news depth |
| **Academic** | Semantic Scholar + arXiv | Citation data + preprints |
| **Technology** | Serper + GitHub | Web docs + code repositories |
| **Korean-specific** | Naver (Blog/Ency/Kin) | Korean web ecosystem coverage |
| **Code/Library** | GitHub + Serper | Source code + documentation |

### Trust Scoring

The Selector node scores each search result using a multi-factor algorithm:

```
Trust Score = Domain Tier Weight + Snippet Relevance + Recency Bonus + URL Quality

Domain Tiers:
  T1 (Authoritative):  .gov, .edu, nature.com, arxiv.org, ...      → +3.0
  T2 (Reliable):       major news outlets, official docs, ...        → +2.0
  T3 (Standard):       blogs, forums, general web, ...               → +1.0
```

### Cost Optimization

| Source | Free Tier | Strategy |
|--------|-----------|----------|
| Serper | 2,500/month | Primary for all queries |
| Naver | 25,000/day | Korean queries + supplement |
| DuckDuckGo | Unlimited | Fallback when quota exceeded |
| Tavily | 1,000/month | Deep research only |
| Semantic Scholar | Unlimited | Academic queries only |
| GitHub | 5,000/hour | Code-related queries only |

The search router selects sources based on query type, minimizing paid API calls while maintaining coverage.

---

## RAG Pipeline

### Architecture: SQLite + NumPy (Zero Infrastructure)

The RAG system uses a lightweight, self-contained architecture:

```
Document Ingestion:
  PDF → PyMuPDF (table preservation) → chunk → embed → SQLite
  Image → EasyOCR → text → chunk → embed → SQLite
  Research → auto-store after publication → embed → SQLite
  Chat → selective storage → embed → SQLite

Retrieval:
  Query → embed → NumPy cosine similarity → top-K → [doc p.N] citations
```

### Why Not Pinecone/Weaviate/Chroma?

| Factor | External Vector DB | SQLite + NumPy |
|--------|-------------------|----------------|
| **Infrastructure** | Requires server/service | Single file |
| **Cost** | $70+/month (managed) | $0 |
| **Backup** | Complex | Copy file |
| **Performance** | Optimized for millions | Sufficient for <100K docs |
| **Deployment** | Extra dependency | Already available |

For a personal assistant or small-team tool, the <100K document scale makes external vector databases unnecessary overhead.

### Anti-Hallucination Features

1. **Exact-quote prompting**: When numbers are detected in retrieved documents, the system applies strict exact-quote prompts to prevent LLM fabrication
2. **Page-level citations**: `[doc p.N]` format traces every claim to specific document pages
3. **Precise retrieval**: `retrieve_exact()` returns source content with page metadata

---

## Security Mode — Hybrid LLM Routing

### The Privacy Problem

Users may ask questions containing sensitive information (personal data, business secrets). Sending these to cloud LLMs (Gemini) creates privacy risks.

### Solution: `is_secure_mode` Routing

```
User Query
    |
    v
[is_secure_mode = true?]
    |               |
   YES              NO
    |               |
    v               v
 Ollama           Gemini
 (EXAONE 3.5)    (2.5 Flash/Pro)
 Local GPU        Cloud API
 Private          Fast + Smart
```

### Hybrid Search in Secure Mode

Even in secure mode, search quality must be maintained. The system uses a two-step process:

1. **Step 1** (Local): EXAONE 3.5 extracts only keywords from user's question
2. **Step 2** (Cloud): Only the keywords (not the raw question) are sent to Gemini to optimize search queries

This preserves privacy (the user's actual question never leaves the local machine) while still leveraging Gemini's superior query optimization.

---

## Modularization Strategy

### Before: 6,300-Line Monolith

The original `agent_mvp.py` contained everything — constants, models, LLM initialization, search functions, utility functions, 18 graph nodes, and graph assembly — in a single file.

**Problems:**
- Impossible to test individual components
- Merge conflicts with every change
- Cloud deployment pulled in GPU dependencies
- No separation of concerns

### After: 9-Module Architecture

```
agent_mvp.py (239 lines)
  ├── Re-exports all module contents (backward compatibility)
  ├── StateGraph assembly (18 nodes + edges)
  └── Dynamic variables (rag_service, compiled_graph)

modules/
  ├── config.py (36)      ← Environment + constants
  ├── models.py (232)     ← State + schemas
  ├── utils.py (523)      ← Pure utilities
  ├── llm_gateway.py (419) ← LLM init + routing
  ├── search.py (820)     ← Search functions
  ├── tools.py (417)      ← Real-time tools
  ├── nodes_chat.py (968) ← Chat + code nodes
  └── nodes_research.py (2987) ← Research pipeline
```

### Backward Compatibility

The key constraint: `server.py` and `voice_service.py` must continue to work with `from agent_mvp import X` unchanged.

**Solution:** `agent_mvp.py` re-exports everything from modules:

```python
from modules.config import (CURRENT_DATE, OLLAMA_MODEL, ...)     # noqa: F401
from modules.llm_gateway import (ask_gemini, switch_ollama_model, ...)  # noqa: F401
# ... all other modules
```

This means zero changes to downstream consumers while gaining full module separation internally.

### Dynamic Variable Challenge

`rag_service` is initialized in `server.py`'s lifespan (not at import time), and needs to be accessible from node modules:

```python
# agent_mvp.py
rag_service = None  # Set by server.py lifespan

# modules/nodes_chat.py
import agent_mvp  # Access agent_mvp.rag_service dynamically
rag = agent_mvp.rag_service
```

This avoids circular imports while maintaining the dynamic initialization pattern.

---

## Dual Deployment Design

### Local Mode (Personal / GPU Server)

- Full feature set: all 4 modes + voice + meeting
- Local LLMs: EXAONE 3.5 (Korean chat), Qwen 2.5 Coder (code)
- GPU required: 8GB+ VRAM
- Security mode available

### Cloud Mode (SaaS)

- Chat + Research + Code modes only
- Gemini-only (no Ollama dependency)
- No GPU required
- Lightweight dependencies (no torch, pyannote)
- Security mode disabled (no local LLM available)

### Switching

A single environment variable controls the entire system:

```env
DEPLOY_MODE=local   # Full features, GPU required
DEPLOY_MODE=cloud   # Gemini-only, lightweight
```

The `llm_gateway.py` module checks `IS_CLOUD` at initialization:

```
if IS_CLOUD:
    ollama_llm = None         # Skip Ollama initialization
    ollama_json_llm = None
    # is_secure=True → falls back to Gemini with warning
```

---

## Streaming Architecture

### Server-Sent Events (SSE)

The frontend connects to `/ai_secretary/stream_v2` via EventSource for real-time updates:

```
Client (React Native)                  Server (FastAPI)
    |                                      |
    |--- POST /stream_v2 ---------------->|
    |                                      |--- LangGraph ainvoke()
    |<--- SSE: {"type": "token", ...} ----|
    |<--- SSE: {"type": "token", ...} ----|
    |<--- SSE: {"type": "progress", ...} -|  ← Step progress update
    |<--- SSE: {"type": "token", ...} ----|
    |<--- SSE: {"type": "done"} ----------|
    |                                      |
```

### Progress Tracking

Deep research sends progress events through the SSE stream, allowing the frontend to show a 6-step progress bar:

```
Step 1/6: Planning research strategy...
Step 2/6: Searching across 6 sources...
Step 3/6: Evaluating source quality...
Step 4/6: Reading full articles...
Step 5/6: Writing comprehensive report...
Step 6/6: Fact-checking and publishing...
```

---

## State Management & Checkpointing

### ResearchState

The `ResearchState` TypedDict carries all data through the graph:

| Field Group | Fields | Purpose |
|-------------|--------|---------|
| **Topic** | topic, refine_topic, sub_topics | Research subject management |
| **Mode** | mode, target_audience, is_secure_mode | Pipeline configuration |
| **Messages** | messages (add_messages reducer) | Conversation history |
| **Research** | scraped_contents, search_results | Collected data |
| **Quality** | review_status, iteration | Self-correction loop control |
| **Progress** | progress, deep_research_start_time | Frontend status updates |
| **Output** | final_report, references | Final deliverables |

### AsyncSqliteSaver

Conversation state persists in SQLite via LangGraph's checkpointing:

- **Session restoration**: Resume conversations after server restart
- **Thread-based**: Each user gets an isolated conversation thread
- **Async**: Non-blocking checkpoint writes during streaming

```
server.py lifespan:
  async with AsyncSqliteSaver.from_conn_string(DB_PATH) as checkpointer:
      compiled_graph = workflow.compile(checkpointer=checkpointer)
      agent_mvp.compiled_graph = compiled_graph
```

---

## API Endpoint Design

### Main Streaming Endpoint

```
POST /ai_secretary/stream_v2
Content-Type: application/json
X-API-Key: <api-key>

{
  "message": "Analyze the AI chip market",
  "mode": "deep",
  "thread_id": "user_123_session_1",
  "target_audience": "expert",
  "is_secure_mode": false
}

Response: SSE stream
  data: {"type": "token", "content": "# AI Chip Market..."}
  data: {"type": "progress", "step": 3, "total": 6, "label": "Evaluating..."}
  data: {"type": "sources", "items": [...]}
  data: {"type": "done"}
```

### Tool Endpoints

Direct tool access bypasses the LLM pipeline for quick results:

```
GET /tools/weather?city=Seoul        → KMA weather data
POST /tools/translate               → Papago translation
GET /tools/geocode?address=강남구    → Naver Maps coordinates
```

---

*This architecture document is part of the AI Secretary portfolio showcase. Source code is maintained in a private repository.*
