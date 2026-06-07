# 🧠 RepoSage — Codebase Intelligence RAG System

> **Status: 🚧 Actively in development — targeting Q3 2026**  
> This project is being built in public. Architecture is finalized, implementation is in progress.

---

## What is RepoSage?

RepoSage is a **self-hosted, privacy-first codebase intelligence system** that lets you have natural language conversations with any GitHub repository — not just its code, but its entire knowledge history: issues, pull requests, commit messages, and documentation, all reasoned over together.

Ask questions like:
- *"Why was Redis chosen over Postgres for session management?"*
- *"What does the `process_payment` function do and where is it called?"*
- *"What bugs have been reported around the authentication module?"*
- *"Walk me through the architecture decisions made in Q3 2024."*

Unlike GitHub Copilot Chat, RepoSage is fully self-hosted — meaning your proprietary code never leaves your infrastructure. It is built for teams in regulated industries (finance, healthcare, defence) where sending code to a third-party API is not permitted.

---

## The Problem It Solves

Every engineering team accumulates institutional knowledge that lives **nowhere searchable**:

- A new engineer spends weeks asking "why is this written this way?"
- Senior engineers get interrupted constantly answering questions already buried in closed issues
- Critical architectural decisions exist only in PR comments from 18 months ago
- Written documentation goes stale the moment it's published

RepoSage makes that knowledge queryable in natural language — with citations back to the exact source file, issue, or PR where the answer lives.

---

## Architecture

```
GitHub Repo
    │
    ├── Source Code ──────────────────────┐
    ├── Issues & PR Descriptions ─────────┤
    ├── Commit Messages ──────────────────┤──► Ingestion Pipeline
    └── README / Docs ───────────────────┘          │
                                                     │
                              ┌──────────────────────▼──────────────────────┐
                              │           Chunking Layer                     │
                              │  Code: AST-aware (tree-sitter, by function)  │
                              │  Text: Recursive character split (512 tok)   │
                              └──────────────────────┬──────────────────────┘
                                                     │
                              ┌──────────────────────▼──────────────────────┐
                              │         Embedding + Vector Store             │
                              │  Model: text-embedding-3-small (OpenAI)      │
                              │  Store: ChromaDB (dev) / Pinecone (prod)     │
                              │  Collections: code | issues | prs            │
                              └──────────────────────┬──────────────────────┘
                                                     │
                              ┌──────────────────────▼──────────────────────┐
                              │         Query Pipeline (LangChain)           │
                              │  1. Embed user query                         │
                              │  2. MMR retrieval across all collections     │
                              │  3. BM25 hybrid re-score                     │
                              │  4. Cross-encoder re-ranking (top-20 → top-5)│
                              └──────────────────────┬──────────────────────┘
                                                     │
                              ┌──────────────────────▼──────────────────────┐
                              │       Agentic Self-Correction (LangGraph)    │
                              │  ┌─────────────────────────────────────┐    │
                              │  │  Retrieve → Grade relevance (LLM)   │    │
                              │  │       ↓ low score                   │    │
                              │  │  Rewrite query (HyDE / step-back)   │    │
                              │  │       ↓ still low                   │    │
                              │  │  Web search fallback (Tavily)        │    │
                              │  │       ↓                              │    │
                              │  │  Generate cited answer               │    │
                              │  └─────────────────────────────────────┘    │
                              └──────────────────────┬──────────────────────┘
                                                     │
                              ┌──────────────────────▼──────────────────────┐
                              │     FastAPI Backend  +  React Frontend       │
                              │  Streaming responses · Source citations       │
                              │  GitHub deep-links · Conversation memory      │
                              └─────────────────────────────────────────────┘
```

---

## Tech Stack

| Layer | Technology | Reason |
|---|---|---|
| Ingestion | PyGithub | GitHub REST API wrapper — fetches files, issues, PRs, commits |
| Code chunking | tree-sitter | AST-aware splitting by function/class boundary |
| Text chunking | LangChain RecursiveCharacterTextSplitter | Semantic-preserving splits for prose |
| Embedding model | `text-embedding-3-small` (OpenAI) | Best quality-to-cost ratio, strong on code + NL |
| Vector DB (dev) | ChromaDB | Zero-setup local store |
| Vector DB (prod) | Pinecone | Managed, scalable, LangChain-compatible |
| Retrieval | MMR + BM25 hybrid | MMR prevents redundant chunks; BM25 catches exact keyword matches |
| Re-ranking | `ms-marco-MiniLM` cross-encoder | Improves precision: re-scores top-20, keeps top-5 |
| Orchestration | LangChain | Chains, retrievers, memory, prompt management |
| Agent / CRAG | LangGraph | State machine for retrieve → grade → rewrite → fallback loop |
| LLM | GPT-4o-mini / GPT-4o | Cost-efficient primary; GPT-4o for complex multi-source queries |
| Backend | FastAPI | Async, auto-documented API with streaming endpoint |
| Frontend | React + Tailwind CSS | Chat UI with file-level source citations |
| Evaluation | RAGAS | Faithfulness, answer relevancy, context recall metrics |
| Deployment | Railway (backend) + Vercel (frontend) | Free tier, zero-config CI/CD |

---

## Key Design Decisions

**Why AST-aware chunking over fixed-size chunking?**  
Splitting code at arbitrary token boundaries destroys semantic coherence — a function split mid-body is useless to retrieve. tree-sitter parses the AST and chunks at natural boundaries (function definitions, class bodies), preserving the full semantic unit. This is the single biggest quality improvement over naive GitHub RAG implementations.

**Why three separate ChromaDB collections?**  
Code, issues, and PR descriptions have fundamentally different retrieval characteristics. A question about architecture decisions should weight PR/issue chunks higher; a question about implementation should weight code chunks higher. Separate collections with per-query weighting gives us that control. A single unified collection loses this signal.

**Why CRAG (Corrective RAG) over standard RAG?**  
Standard RAG always returns an answer regardless of whether the retrieved chunks are actually relevant. In a codebase context, vague PR descriptions or missing documentation means retrieval quality varies enormously. The CRAG loop grades each retrieval attempt and either retries with a rewritten query or honestly acknowledges sparse documentation — rather than hallucinating context that was never recorded.

**Why self-hosted over GitHub Copilot Chat?**  
GitHub Copilot Chat sends your code to Microsoft's servers. For teams in regulated industries — banking, healthcare, defence, or any org with strict data residency requirements — this is a non-starter. RepoSage runs entirely on your own infrastructure.

---

## Project Structure

```
RAG/
├── backend/
│   ├── ingest.py          # GitHub API fetch → chunk → embed → ChromaDB
│   ├── retriever.py       # Query embedding + MMR/BM25 hybrid retrieval
│   ├── reranker.py        # Cross-encoder re-ranking (top-20 → top-5)
│   ├── chain.py           # LangChain RAG chain + conversation memory
│   ├── agent.py           # LangGraph CRAG state machine
│   └── api.py             # FastAPI endpoints (POST /query, POST /ingest)
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── ChatWindow.jsx     # Streaming chat interface
│   │   │   ├── SourceCitation.jsx # Clickable GitHub deep-links
│   │   │   └── RepoInput.jsx      # GitHub repo URL input
│   │   └── App.jsx
├── eval/
│   └── ragas_eval.py      # RAGAS evaluation suite (20-question benchmark)
├── data/
│   └── test_queries.json  # Benchmark question set
├── docker-compose.yml
└── README.md
```

---

## Current Status

| Module | Status | Notes |
|---|---|---|
| Project architecture | ✅ Finalized | Tech stack locked, structure defined |
| `ingest.py` | 🔄 In progress | Current focus |
| `retriever.py` | ⏳ Planned | Next up |
| `chain.py` | ⏳ Planned | Next up |
| `agent.py` (CRAG) | ⏳ Planned | After retrieval pipeline |
| React frontend | ⏳ Planned | After backend complete |
| RAGAS evaluation | ⏳ Planned | After frontend |
| Deployment | ⏳ Planned | Final milestone |

---

## Roadmap

- [x] Define architecture and finalize tech stack
- [x] Set up project structure and repository
- [ ] GitHub ingestion pipeline (`ingest.py`)
- [ ] Query pipeline, re-ranking, FastAPI backend (`retriever.py`, `reranker.py`, `api.py`)
- [ ] LangGraph CRAG self-correction agent (`agent.py`)
- [ ] React frontend + RAGAS evaluation
- [ ] Deployment + benchmarking results

---

## Why This Is Different from Existing Tools

| Feature | RepoSage | GitHub Copilot Chat | Greptile |
|---|---|---|---|
| Self-hosted | ✅ Yes | ❌ No | ❌ No |
| Code leaves your servers | ❌ Never | ✅ Always | ✅ Always |
| Cross-references issues + PRs + code | ✅ Yes | Partial | Partial |
| Self-correcting retrieval (CRAG) | ✅ Yes | ❌ No | ❌ No |
| Works without GitHub.com | ✅ Yes | ❌ No | ❌ No |
| Free (self-hosted) | ✅ Yes | ❌ $19-39/user/mo | ❌ Paid |

---

## Getting Started

> ⚠️ Full setup instructions will be added once core modules are complete (Week 3).

```bash
# Clone the repo
git clone https://github.com/AyushDave32/RAG.git
cd RAG

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies (requirements.txt in progress)
pip install -r requirements.txt

# Set environment variables
cp .env.example .env
# Add your OPENAI_API_KEY and GITHUB_TOKEN to .env
```

---

## Evaluation Results

> 📊 RAGAS benchmark results will be added after Week 5.  
> Target metrics: Faithfulness > 0.80 · Answer Relevancy > 0.75 · Context Recall > 0.72

---

## Built By

**Ayush Dave** — Junior AI/ML Engineer  
[LinkedIn](https://www.linkedin.com/in/ayush-dave-57b8b5228) · [GitHub](https://github.com/AyushDave32)

---

## License

MIT License — see [LICENSE](LICENSE) for details.

