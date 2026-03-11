<div align="center">

# AI Secretary

### Intelligent Multi-Mode AI Assistant with Deep Research Pipeline

An LLM agent system that automatically selects the optimal response strategy based on query complexity — from instant chat to multi-source deep research reports with company intelligence.

[**English**](./README.md) | [**日本語**](./README.ja.md) | [**한국어**](./README.ko.md)

[![Python](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.128-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![LangGraph](https://img.shields.io/badge/LangGraph-1.0-1C3C3C?logo=langchain&logoColor=white)](https://langchain-ai.github.io/langgraph/)
[![React Native](https://img.shields.io/badge/React_Native-Expo-61DAFB?logo=react&logoColor=black)](https://reactnative.dev)
[![Gemini](https://img.shields.io/badge/Gemini_2.5-Flash-4285F4?logo=google&logoColor=white)](https://ai.google.dev)

</div>

---

## Overview

**AI Secretary** is a production-ready LLM agent system built on LangGraph's StateGraph architecture (24 nodes, conditional edges). Unlike simple chatbot wrappers, it implements a **multi-stage research pipeline** with automatic quality evaluation, self-correcting search loops, and fact-checking — producing cited, structured reports from 8 search sources.

The system features **benchmark-driven hybrid LLM routing** (cloud Gemini + local Ollama) with cost-optimized node-level routing, **automatic query classification** that selects research profiles without user intervention, **Business Development research mode** with multi-national company intelligence (DART/EDGAR/EDINET), **research session caching** with URL-level trust tracking, **agentic self-correction loops** for code verification, **dynamic Human-in-the-Loop** for research strategy selection, and a **Korean law RAG system** powered by the Korea Legislation Research Institute API.

The project includes a **comprehensive test suite** (83+ unit tests) covering core logic, and a **hybrid LLM benchmark system** for data-driven routing decisions.

> **Note:** This is a portfolio showcase. Source code is maintained in a private repository.

---

## Key Features

### General Chat
- Context-aware conversation with automatic web search when needed
- Hybrid keyword extraction: local LLM extracts keywords, cloud LLM optimizes search queries (privacy-preserving)
- RAG integration for document-grounded responses with `[doc p.N]` citations
- **Law RAG**: Auto-detects legal queries and injects relevant law context with article-level citations
- AI-suggested follow-up questions for conversation continuity

### Deep Research
- **Query Classifier**: Automatically classifies queries into 7 research types (market_entry, competitor_analysis, company_profile, etc.) using local LLM — zero cloud cost for classification
- **Dynamic HITL**: Before entering the pipeline, LLM generates 2-3 analysis strategy options per query — user selects or types custom direction
- **BD Profile**: Business Development research mode with specialized prompts, company intelligence injection, and market entry analysis
- **Company Intelligence**: Multi-national company data from DART (Korea), EDGAR (US), EDINET (Japan) — automatic extraction, verification, and context injection
- **Research Cache**: Session history tracking, URL-level trust caching with auto-tagging, research snapshots for change detection
- **2-4 minute** comprehensive analysis reports from 8 search sources
- Self-correcting quality loop: Evaluator scores research quality, Strategist re-plans if insufficient (max 3 iterations)
- Automatic fact-checking against source materials before publication
- Structured markdown output with per-section source accordions

### Code Assistant
- Dedicated code generation with automatic verification
- **Agentic self-correction loop**: Code Fact Check detects issues → Code Fix auto-repairs → re-verification (max 2 iterations)
- Code fact-checker validates libraries/APIs against official docs and GitHub
- Supports local model (Qwen 2.5 Coder) and cloud model (Gemini) switching

### Meeting Intelligence
- Speaker diarization (pyannote) + speech-to-text transcription
- Automatic meeting summary generation with speaker-labeled segments

### Real-time Tools
- **Weather**: Korean Meteorological Agency API with Lambert Conformal Conic grid projection
- **Translation**: Naver Papago (KO/EN/JA/ZH)
- **Geocoding**: Naver Maps address-to-coordinate conversion

### Law RAG
- Korean law search via Korea Legislation Research Institute Open API (law.go.kr)
- Hybrid retrieval: vector similarity + BM25 with Reciprocal Rank Fusion
- Article-level chunking with `[Law Name Article N]` citation format
- Privacy HITL: user chooses secure (local) or cloud mode for legal queries
- Covers 14 laws (real estate + security/network domains)

---

## System Architecture

```mermaid
graph TD
    subgraph Frontend["Frontend (React Native / Expo)"]
        UI["4-Mode UI<br/>General | Deep | Code | Meeting"]
    end

    subgraph Backend["Backend (FastAPI + LangGraph)"]
        GK["Gatekeeper Router<br/><i>Auto-classifies query intent</i>"]

        subgraph GeneralPath["General Chat Path"]
            SC["Search Checker"]
            SS["Simple Search"]
            TE["Tool Executor<br/><i>Weather / Translate / Geocode</i>"]
            GC["General Chat Node<br/><i>+ RAG + Law RAG + Fact Check</i>"]
        end

        subgraph DeepPath["Deep Research Pipeline"]
            QC["Query Classifier<br/><i>Auto research type</i>"]
            CL["Cache Lookup<br/><i>Session history</i>"]
            HITL["Dynamic HITL<br/><i>Strategy selection</i>"]
            PL["Planner"]
            OL["Outliner"]
            MN["Miner<br/><i>8-source parallel search</i>"]
            SL["Selector<br/><i>Trust scoring + dedup</i>"]
            RD["Deep Reader<br/><i>Full-text extraction</i>"]
            EV["Evaluator<br/><i>Quality scoring</i>"]
            ST["Strategist<br/><i>Re-plan on low quality</i>"]
            WR["Writer"]
            FC["Fact Checker"]
            PB["Publisher"]
            HS["History Saver<br/><i>Cache persistence</i>"]
            LB["Librarian<br/><i>RAG storage</i>"]
        end

        subgraph BDPath["Company Intelligence (BD Mode)"]
            CE["Company Extractor<br/><i>DART / EDGAR / EDINET</i>"]
            CV["Company Verifier<br/><i>Cross-source validation</i>"]
        end

        subgraph CodePath["Code Mode"]
            CC["Code Chat"]
            CF["Code Fact Check<br/><i>Verify libs + APIs</i>"]
            FIX["Code Fix<br/><i>Auto-repair</i>"]
        end
    end

    subgraph LLM["LLM Layer"]
        GM["Gemini 2.5<br/>Flash"]
        OL2["Ollama Local<br/>EXAONE 3.5 / Qwen 2.5"]
    end

    subgraph Search["Search Sources (8)"]
        SR["Serper (Google)"]
        NV["Naver (KR)"]
        TV["Tavily"]
        DDG["DuckDuckGo"]
        SS2["Semantic Scholar"]
        GH["GitHub"]
        LAW["Law API (law.go.kr)"]
    end

    UI -->|SSE Streaming| GK
    GK -->|General| SC
    GK -->|Deep Research| QC
    GK -->|Code| CC

    QC --> CL --> HITL
    HITL -->|"User strategy selection"| PL

    SC -->|"Search needed"| SS
    SC -->|"Tool needed"| TE
    SC -->|"Direct answer"| GC
    SS --> GC
    TE --> GC

    PL --> OL --> MN --> SL --> RD --> EV
    EV -->|"Quality OK"| WR
    EV -->|"Need more data"| ST
    ST -->|"Re-search"| MN
    WR --> FC --> PB --> HS --> LB
    PL -->|"BD Profile"| CE --> CV --> OL

    CC --> CF
    CF -->|"ISSUES_FOUND"| FIX
    FIX -->|"Re-verify"| CF

    GC & LB & CF -->|END| UI

    Backend -->|"is_secure=false"| GM
    Backend -->|"is_secure=true"| OL2
    MN --> Search
```

### Architecture Highlights

| Aspect | Design Decision | Why |
|--------|----------------|-----|
| **Graph Engine** | LangGraph StateGraph (24 nodes) | Deterministic node routing > ReAct agent's unpredictable tool calls. Complex pipelines need explicit flow control |
| **Query Classifier** | Local LLM auto-classification | Classifies queries into 7 research types using EXAONE 7B — zero cloud cost, respects frontend overrides |
| **Quality Loop** | Evaluator → Strategist → Miner cycle | Self-correcting research: if source quality < threshold, re-plan and re-search (max 3x) |
| **Code Self-Correction** | Code Fact Check ↔ Code Fix loop | Agentic loop: detect issues → auto-fix → re-verify (max 2x) |
| **Dynamic HITL** | LLM-generated strategy options | 2-3 analysis strategies per query + custom text input, injected into Planner |
| **BD Profile** | Business Development research mode | Specialized prompts + company intelligence (DART/EDGAR/EDINET) for market entry analysis |
| **Company Intelligence** | Multi-national company DB | DART (Korea), EDGAR (US), EDINET (Japan) — extractor + verifier nodes with cross-source validation |
| **Research Cache** | SQLite session + URL cache | Session history tracking, URL trust caching, auto-tagging, research snapshots |
| **LLM Routing** | Benchmark-driven hybrid routing | `is_secure` > `hybrid_mode` > `default(cloud)` priority chain. Node-level local/cloud assignment |
| **LLM Gateway** | 4 unified gateway functions | All nodes route through `ask_gemini*()` — single `is_secure` parameter switches entire system |
| **Hybrid Search** | Local keyword extraction + Cloud query optimization | User's raw question never leaves local machine in secure mode |
| **Law RAG** | Vector + BM25 hybrid with article-level chunking | Korean law search with privacy HITL (user chooses secure/cloud mode) |
| **Checkpointing** | AsyncSqliteSaver | Conversation persistence across server restarts |
| **Testing** | 83+ pytest unit tests | Pure logic tests with dependency mocking — formatters, models, cache, benchmarks, URL fixing |

---

## Tech Stack

### Backend
| Technology | Role |
|-----------|------|
| **Python 3.11** | Runtime |
| **FastAPI** | Async REST API + SSE streaming |
| **LangGraph 1.0** | StateGraph workflow orchestration (24 nodes, conditional edges) |
| **LangChain** | LLM abstractions + tool integration |
| **Gemini 2.5 Flash** | Unified cloud LLM — benchmark-validated: Flash matches Pro quality (-4%) at 2x speed |
| **Ollama + EXAONE 3.5** | Local LLM (Korean-optimized, 7.8B) |
| **Qwen 2.5 Coder** | Local code generation model |

### Frontend
| Technology | Role |
|-----------|------|
| **React Native (Expo)** | Cross-platform mobile app |
| **TypeScript** | Type-safe development |
| **EventSource** | Real-time SSE streaming |
| **SimpleMarkdown** | Custom lightweight markdown renderer |

### AI / ML
| Technology | Role |
|-----------|------|
| **sentence-transformers** | Embedding (all-MiniLM-L6-v2 + ko-sroberta) |
| **Faster-Whisper** | Speech-to-text |
| **pyannote.audio** | Speaker diarization |
| **EasyOCR** | Image text extraction |
| **PyMuPDF** | PDF parsing with table preservation |

### Search & Data
| Source | Type | Free Tier |
|--------|------|-----------|
| **Serper.dev** | Google Search | 2,500/month |
| **Naver Search** | Korean web/blog/news/encyclopedia | 25,000/day |
| **Tavily** | News + academic | 1,000/month |
| **DuckDuckGo** | General web | Unlimited |
| **Semantic Scholar** | Academic papers | Unlimited |
| **arXiv** | CS/AI/Math preprints | Unlimited |
| **GitHub** | Code repositories | 5,000/hour |
| **Law API** | Korean legislation (law.go.kr) | Unlimited |

---

## Module Architecture

The system was refactored from a **6,300-line monolith** into **14+ specialized modules**:

| Module | Lines | Responsibility |
|--------|-------|---------------|
| `config.py` | 89 | Environment variables, deployment mode, hybrid routing config |
| `models.py` | 323 | State definitions (HITL + BD profile + query classifier fields), schemas, domain trust tiers |
| `utils.py` | 547 | URL validation, trust scoring, text processing |
| `llm_gateway.py` | 485 | Unified LLM gateway, Gemini/Ollama routing, hybrid mode switching |
| `search.py` | 865 | 11 search functions across 8 sources |
| `tools.py` | 417 | Weather / Translation / Geocoding APIs |
| `nodes_chat.py` | 1,156 | General chat, code mode, query classifier, HITL nodes |
| `nodes_research.py` | 3,401 | Deep research pipeline (11 nodes, all gateway-integrated) |
| `nodes_company.py` | 491 | Company extractor + verifier nodes (BD profile) |
| `research_cache.py` | 798 | Research session cache, URL trust tracking, auto-tagging |
| `company_store.py` | 791 | Company intelligence DB with multi-national connectors |
| `company_connectors/` | 1,362 | DART (Korea), EDGAR (US), EDINET (Japan), Web enrichment |
| `law_rag.py` | 1,440 | Korean law RAG (Vector + BM25 hybrid, article-level chunking) |
| `tag_enums.py` | 304 | Research tag classification system |
| `agent_mvp.py` | 304 | Re-exports + StateGraph assembly (24 nodes, conditional edges) |

### Dual Deployment Design

```
                    .env: DEPLOY_MODE=local          .env: DEPLOY_MODE=cloud
                    ========================         ========================
LLM Gateway         Gemini + Ollama (GPU)            Gemini only
Security Mode        Available (local LLM)            Disabled
Voice/Meeting        Enabled (GPU STT)                Disabled
Code Model           Qwen 2.5 Coder (local)           Gemini
Dependencies         Full (torch, pyannote...)         Lightweight
```

A single `DEPLOY_MODE` environment variable switches the entire system between personal (GPU) and cloud (SaaS) configurations.

---

## Agentic Reasoning

### 1. Code Mode — Self-Correction Loop

After code generation, an **agentic loop** automatically verifies, detects issues, fixes, and re-verifies:

```mermaid
flowchart LR
    CC["Code Chat"] --> CF{"Code Fact Check"}
    CF -->|PASS / WARNING| END(("END"))
    CF -->|ISSUES_FOUND| FIX["Code Fix"]
    FIX -->|"Re-verify (max 2x)"| CF
```

- **Code Fact Check**: Extracts libraries/APIs from generated code → validates against official docs + GitHub
- **Code Fix**: On ISSUES_FOUND, LLM auto-repairs using verification results as context
- **Re-verification**: Fixed code goes back through Fact Check — max 2 iterations before delivery

### 2. Dynamic HITL (Human-in-the-Loop)

Before deep research, LLM dynamically generates analysis strategy options tailored to each query:

1. User question → LLM generates 2-3 strategy options (JSON)
2. Options sent via SSE to frontend
3. Frontend renders dynamic buttons + free-text input (TextInput)
4. User selection → `selected_option` injected into Planner
5. Planner adjusts `refined_topic` and research direction based on selection

**Custom input**: `custom:` prefix convention distinguishes LLM-generated options from user-typed free text.

### 3. LLM Gateway Integration

All LLM calls across all nodes (11 deep research + chat + code) are unified through 4 gateway functions:

| Function | Purpose |
|----------|---------|
| `ask_gemini()` | General text response |
| `ask_gemini_json()` | JSON structured response |
| `ask_gemini_high()` | Flash (formerly Pro — benchmark-validated equivalent) |
| `ask_gemini_high_json()` | Flash + JSON |

A single `is_secure` parameter switches the entire system between Gemini (cloud) and Ollama (local).

### 4. Law RAG with Privacy HITL

When a legal query is detected, the system offers a privacy choice before proceeding:

1. Legal keyword detected in user query
2. If security mode is off → presents privacy HITL buttons
3. User chooses: Secure Mode (local LLM) or Cloud Mode (Gemini)
4. RAG search runs locally regardless → law context injected into response
5. Final answer generated with article-level citations and disclaimer

### 5. Query Classifier — Automatic Research Profiling

Before deep research begins, a local LLM automatically classifies the query into one of 7 research types:

| Research Type | Description |
|--------------|-------------|
| `market_entry` | New market entry feasibility analysis |
| `competitor_analysis` | Competitive landscape comparison |
| `company_profile` | Single company deep dive |
| `market_size` | Market sizing and growth projections |
| `partnership` | Partnership/M&A opportunity evaluation |
| `trend_analysis` | Industry trend and technology analysis |
| `general_research` | Open-ended research (default) |

- Uses **EXAONE 7B** (local) — zero cloud cost for classification
- Respects frontend overrides: if user manually selects a profile, classifier is bypassed
- Classification result selects the appropriate **prompt profile** (e.g., `bd_generic` for business development)

### 6. Company Intelligence — Multi-National Data Pipeline

For BD profile research, the system automatically extracts and verifies company data:

```mermaid
flowchart LR
    PL["Planner"] -->|"BD Profile"| CE["Company Extractor"]
    CE -->|"Extract from query"| DB[("Company DB<br/>DART / EDGAR / EDINET")]
    DB --> CV["Company Verifier"]
    CV -->|"Verified context"| WR["Writer"]
```

- **Company Extractor**: Identifies company mentions → queries DART (Korea), EDGAR (US), EDINET (Japan)
- **Company Verifier**: Cross-validates data across sources, assigns data quality scores
- **Writer Integration**: Verified company context (revenue, funding, products) injected into report generation

### 7. Research Cache & Session History

Research sessions are persisted for reuse and tracking:

- **Session Cache**: Each research run creates a session with topic, mode, profile, and status
- **URL Trust Cache**: Crawled URLs are cached with domain, trust score, tags, and HTTP status
- **Auto-tagging**: 3-tier tag system classifies sources (Official/Academic/News/Blog/etc.)
- **Research Snapshots**: Enables change detection across repeated research on the same topic

### 8. Benchmark-Driven Hybrid LLM Routing

A benchmark system measures local vs cloud LLM performance per task type, enabling data-driven routing:

```
Priority: is_secure_mode > hybrid_mode > default(cloud)

Hybrid Node Routing:
  _classify_query        → local (EXAONE)
  _generate_search_query → local
  search_checker         → local
  generate_followup      → local
  Everything else        → cloud (Gemini)
```

The benchmark suite (`benchmark_hybrid.py`) evaluates JSON parsing, query classification, search query generation, search checking, and follow-up generation — producing accuracy scores that inform routing decisions.

---

## Deep Research Pipeline — In Detail

The deep research mode is the core differentiator. Here's how a single research query flows through the specialized pipeline:

```
User: "Analyze the current state of AI chip market competition"
 |
 v
[Query Classifier] - Auto-classifies query into research type (e.g., "competitor_analysis")
                   - Uses local LLM (EXAONE 7B) — zero cloud cost
                   - Selects prompt profile (default / bd_generic)
 |
 v
[Cache Lookup] - Checks session history for similar past research
               - Loads cached URL trust scores for faster selection
 |
 v
[Dynamic HITL] - LLM generates 2-3 analysis strategy options
               - User selects strategy or types custom direction
 |
 v
[1. Planner] - Reads selected_option + research_type, adjusts direction
             - Generates refined topic and target structure
             - (BD mode) Routes to Company Extractor first
 |
 v
[BD: Company Extractor] - Extracts company mentions from query
                        - Queries DART/EDGAR/EDINET for company data
[BD: Company Verifier]  - Cross-validates data, assigns quality scores
                        - Injects verified company context into state
 |
 v
[2. Outliner] - Creates structured outline with 4-6 sections
              - Generates optimized search queries per section
 |
 v
[3. Miner] - Parallel search across 8 sources (Serper + Naver + Tavily + ...)
           - Collects 20-30 raw results per iteration
 |
 v
[4. Selector] - Trust scoring: domain tier (T1/T2/T3) + snippet relevance
              - URL validation (async batch HEAD requests)
              - Deduplication + top-K selection
              - Cached URL trust scores accelerate repeated research
 |
 v
[5. Deep Reader] - Full-text extraction via Crawl4AI
                 - Long text summarization for context window management
 |
 v
[6. Evaluator] - Scores: coverage, depth, source diversity, recency
               - Decision: PASS (→ Writer) or RETRY (→ Strategist)
 |
 v
[7. Strategist] (if RETRY) - Identifies coverage gaps
                            - Generates new targeted search queries
                            - Routes back to Miner (max 3 iterations)
 |
 v
[8. Writer] - Generates structured markdown report
            - Per-section source citations
            - (BD mode) Integrates verified company data
            - Audience-adaptive tone (expert / general)
 |
 v
[9. Fact Checker] - Cross-references claims against source materials
                  - Flags unsupported statements
 |
 v
[10. Publisher] - Generates executive insight summary (infographic)
               - Restores reference URLs to clickable markdown links
               - Composes final output structure
 |
 v
[11. History Saver] - Persists session metadata + URL trust scores
                    - Enables cache hits for future research
 |
 v
[12. Librarian] - Stores research results in RAG vector store
               - Enables future retrieval and cross-referencing
```

---

## Performance Optimization

| Metric | Before (All Deep Research) | After (Smart Routing) | Improvement |
|--------|---------------------------|----------------------|-------------|
| Avg Response Time | 2-4 min for every query | 1-5 sec for general chat | **95% faster** for simple queries |
| Search API Calls | 14+ per query (Tavily) | 0-3 per query (Serper) | **~80% fewer** |
| LLM Token Usage | Full pipeline every time | Proportional to complexity | Significant reduction |

The **Gatekeeper Router** classifies incoming queries into 2 primary paths:
1. **General Chat** — Simple questions, fact checks, and searchable queries get instant responses. Search and quick fact-checking are handled inline within the chat node
2. **Deep Research** — Only complex, multi-aspect analysis triggers the full 11-node pipeline

This 2-layer routing (Gatekeeper → Search Checker) eliminates unnecessary computation for the majority of queries.

---

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/health` | GET | Server health check |
| `/ai_secretary/stream_v2` | POST | Main chat endpoint (SSE streaming) |
| `/ai_secretary/switch_model` | POST | Switch Ollama models (code mode) |
| `/voice/chat` | POST | Voice input → STT → LLM → response |
| `/meeting/upload` | POST | Meeting audio → diarization + summary |
| `/rag/upload` | POST | Ingest PDF/image for RAG |
| `/rag/search` | GET | Vector similarity search |
| `/rag/stats` | GET | Vector store statistics |
| `/tools/weather` | GET | Weather forecast (KMA API) |
| `/tools/translate` | POST | Text translation (Papago) |
| `/tools/geocode` | GET | Address → coordinates (Naver Maps) |
| `/law/crawl` | POST | Crawl Korean legislation |
| `/law/search` | GET | Law search (Vector + BM25 hybrid) |
| `/law/update` | POST | Re-crawl updated laws only |

---

## Design Decisions

### Why LangGraph StateGraph over ReAct Agent?

ReAct agents choose tools dynamically, which is powerful but **unpredictable** for complex multi-step workflows. A 24-node workflow needs:
- Guaranteed execution order (search → evaluate → write)
- Conditional branching with retry logic (Evaluator → Strategist → Miner, max 3x)
- Agentic loops: code self-correction (code_fact_check ↔ code_fix, max 2x)
- SSE streaming for real-time progress updates + dynamic HITL option delivery
- State persistence across server restarts (AsyncSqliteSaver checkpointing)

StateGraph provides **deterministic routing** with explicit conditional edges, making the pipeline debuggable and reproducible.

### Why Hybrid Search (8 Sources)?

No single search API covers all needs:
- **Serper** (Google): Best general coverage, but limited free tier
- **Naver**: Essential for Korean-language sources (25K free/day)
- **Semantic Scholar**: Academic papers with citation data
- **GitHub**: Code-related research
- **DuckDuckGo**: Unlimited fallback
- **Tavily**: Good for news, but expensive at scale
- **Law API** (law.go.kr): Korean legislation with article-level precision

### Why SQLite for Vector Storage?

For a personal/small-team deployment:
- Zero infrastructure overhead (no Pinecone/Weaviate/Chroma server)
- Single-file backup and migration
- NumPy cosine similarity is fast enough for <100K documents
- Reduces cloud deployment costs significantly

---

## Testing

The project includes a comprehensive test suite validating core logic without LLM dependencies:

```
tests/
├── conftest.py              # Mock factory for heavy dependencies (30+ modules)
├── test_utils.py            # clean_text, parse_json, trust_score (17 tests)
├── test_models.py           # Pydantic model validation (7 tests)
├── test_fix_reference.py    # URL reference patching (8 tests)
├── test_formatters.py       # Company/session formatters (12 tests)
├── test_benchmark_evals.py  # Benchmark evaluation functions (23 tests)
└── test_research_cache.py   # SQLite cache CRUD + sessions (10 tests)
```

**83+ tests** passing in **< 1 second**, covering:
- Pure utility functions (text cleaning, JSON parsing, trust scoring)
- Pydantic model validation with edge cases
- Deep research URL reference fixing
- Research cache SQLite integration (using `tmp_path` fixtures)
- Benchmark evaluation scoring functions

The mock strategy uses `types.ModuleType` with proper `__path__`, `__spec__`, `__package__` attributes to satisfy Python's import system for 30+ heavy dependencies (LangChain, LangGraph, Google AI, PyTorch, etc.) — enabling fast, isolated testing of pure business logic.

---

## Demo

> Screenshots and demo recordings will be added here.

<!--
![General Chat Demo](./assets/demo-general.gif)
![Deep Research Demo](./assets/demo-research.gif)
-->

*Coming soon: Live demo GIFs showcasing General Chat, Deep Research pipeline, and Code mode.*

---

## Contact

**Available for freelance LLM agent system development.**

I design and build production-grade AI agent systems — from architecture to deployment.

If you need:
- Custom LLM agent pipelines for your business domain
- RAG systems with domain-specific document ingestion
- Multi-source research automation tools
- Hybrid cloud/on-premise AI deployments

Feel free to reach out.

- Email: parupin72@gmail.com
<!-- - LinkedIn: [Your Profile](https://linkedin.com/in/yourprofile) -->
- GitHub: [@leovis87](https://github.com/leovis87)

---

<div align="center">

*Built with LangGraph, FastAPI, Gemini, and a lot of coffee.*

</div>
