# DevUp — Project Context (CLAUDE.md)

> Context anchor for anyone (incl. CC) picking up this repo. Read this first.

## What this is
**DevUp**: a developer-upskilling recommender. It recommends learning resources
(courses / repos / tools / articles) to a developer based on their behaviour,
tracks their *evolving* interests, and generates a personalized **persuasive nudge**
per recommendation. It also exposes an **interactive chat agent** so the user can
steer/customize recommendations conversationally.

Started for the SmartReco hackathon, now **decoupled** — hackathon and Mesh API are
**out of scope**. This is a standalone AI-engineer portfolio project.

## Working conventions (important)
- **Arnav drives the build himself** (learning-first). The chat is the main
  design/intuition driver; code is written collaboratively/by Arnav, **not**
  generated autonomously. Claude Code picks up **later, mainly for UI**.
- **Git workflow:** Arnav builds structure locally in VS Code and pushes himself
  (`git add` → `commit` → `push origin main`). Claude touches the remote (via
  connector) **only when Arnav explicitly says "verify"** — never otherwise.
- **No shortcuts. Everything must be verifiable/defensible** for resume + interviews.
  Do not inflate claims ("Netflix-scale", "LLM gateway with fallback") unless it was
  actually built.
- Explanations: Hinglish-friendly, direct, analogy-first.

## Architecture (locked)
```
behaviour events
  -> Profiler (recency-weighted taste vector)
  -> Hybrid Retrieval over Qdrant (dense bge-small + BM25 -> RRF fusion)
  -> Cross-encoder rerank (bge-reranker-v2-m3, local)
  -> LangGraph agents: Perceiver -> Retriever -> Ranker -> Persuasion
  -> Response (top-N recs + persuasive nudge)

Interactive surface:
  /chat agent (LangGraph, stateful) -> reuses Core as tools -> steerable recs
```
All LLM (generation/reasoning) calls go through a **provider-agnostic OpenAI-compatible
client**, defaulting to **Google AI Studio (Gemini)**. Embeddings + reranker run **local**.

Cross-cutting: **Redis** (cache + chat session state) and **LangSmith** (observability)
wrap the pipeline; neither is a data source for recommendations.

## Key design decisions
- **Content/vector-first.** embedding-vs-embedding always: content space = Qdrant
  embeddings; ANN + HNSW powers similarity.
- **No collaborative filtering in MVP.** Vector-first already sidesteps cold start.
  CF would only ever be a *separate fused signal* (combined at ranking via RRF, not
  merged into the vector search) — parked for later.
- **Polyglot persistence — two stores, clear responsibilities.**
  - **Postgres** (relational): `items`, `events`, derived `profiles` — all structured
    behaviour + metrics. Metrics *build the query vector* and *filter/boost* results;
    they are **not** themselves searched.
  - **Qdrant** (vector): content embeddings (payload = metadata for filtering) +
    optional user taste vector. ANN + HNSW.
  - **Why Postgres (not SQLite):** microservices/Docker is the destination, so go
    straight to the prod DB for **dev/prod parity** — no throwaway SQLite→Postgres
    migration later. SQLite is single-writer/one-file, poor fit for multi-container
    concurrent access.
  - **Why Qdrant (not pgvector):** Qdrant's ANN tooling is more mature; keep the
    vector store purpose-built and separate from the relational store.
- **Profiler** = the core "intelligence" (pure Python/numpy, no training):
  user taste vector = **recency-weighted average of engaged items' embeddings**
  (weights from metrics: completion, dwell, recency). Also emits filters (drop seen,
  boost strong categories).
- **Retrieval funnel:** high first-stage top-k (recall) -> cross-encoder rerank
  (precision) -> top-N.
- **Redis (in MVP) — for the expensive/stateful paths only:**
  - **LLM / semantic response cache** — Gemini calls are the slow + costly path.
  - **Short-term chat session memory** — conversation buffer + agent scratchpad
    per session for the `/chat` agent.
  - **NOT** for caching Qdrant results — ANN is already fast, and caching results
    fights the recommender's freshness (recs must change as behaviour changes).
- **Observability (in MVP) — LangSmith:** traces the *system*, not the user.
  Per-node latency, token cost, LangGraph routing, per-agent prompt/response, tool
  calls, failures. **Distinct plane** from **product analytics** (CTR / dwell /
  completion / D1-D7), which lives in Postgres and feeds the Profiler. Do not conflate
  the two: LangSmith answers "is my Ranker slow / how many tokens", the events store
  answers "what did the user do".
- **`/chat` agent (in MVP):** conversational, steerable recommender surface
  (bottom-right corner UI). Reuses the Core (Profiler/retrieve/rerank) as tools inside
  a stateful LangGraph loop; Redis holds session state.
- **Models:** `BAAI/bge-small-en-v1.5` (bi-encoder, local), `bge-reranker-v2-m3`
  (cross-encoder, local). Sparse via `rank-bm25`.
- **Guardrails (post-MVP):** output-side judge — start with a small Gemini/Gemma
  judge; Llama Guard (Groq) or a local deberta prompt-injection classifier later —
  plus XML/delimiter + pydantic schema + regex for structure & injection defense.

## Data model (sketch)
- **Postgres:** `items(id, title, description, tags, type, level)`,
  `events(user_id, item_id, event_type, dwell, ts)`, derived `profiles`.
  Managed via **SQLAlchemy models + Alembic migrations** (no hand-written DDL).
- **Qdrant:** item vectors (payload = metadata for filtering); optional user taste vectors.
- **Redis:** `llmcache:<hash>` (response cache), `chat:<session_id>` (session memory).

## API (FastAPI)
- `POST /events` — log behaviour
- `GET /recommend/{user_id}` — ranked recs + persuasive nudge
- `GET /profile/{user_id}` — inspect profile (debug)
- `POST /chat` — interactive, steerable recommender (MVP)

## Scope
**MVP:** catalog + Postgres events · Qdrant content embeddings · Profiler ·
hybrid retrieval (dense+BM25->RRF) · cross-encoder rerank · LangGraph agents
(Perceiver/Retriever/Ranker/Persuasion) · **/chat agent** · **Redis** (LLM cache +
chat session state) · **LangSmith** observability · FastAPI · Gemini-backed LLM layer ·
docker-compose (postgres + qdrant + redis) for local dev.

**Post-MVP (parked):** guardrail judge, retention metrics
(CTR / dwell / completion / D1-D7), CF, offline eval (recall@k / NDCG / MRR),
Streamlit/web UI, Dockerfile + deploy (microservices split on AWS).

## Stack
Python 3.11 · FastAPI · **Postgres (SQLAlchemy + Alembic)** · Qdrant ·
sentence-transformers (bge-small, bge-reranker) · rank-bm25 · LangGraph · **LangSmith** ·
**Redis** · OpenAI-compatible client -> Google AI Studio (Gemini) ·
docker-compose (local stack).

## Project structure (locked)
```
devup/
  config.py            # pydantic-settings — single source of truth (keys, models, top-k, weights, URLs)
  api/     main.py, schemas.py
  core/    profiler.py, retrieval.py (dense+BM25->RRF), rerank.py
  agents/  graph.py     # Perceiver->Retriever->Ranker->Persuasion
           chat.py      # /chat stateful agent  (added when built)
  llm/     client.py    # provider-agnostic, Gemini default
  db/      models.py, session.py     # Postgres: SQLAlchemy models + engine/session
  vectors.py           # Qdrant: client + collection + search
  cache.py             # Redis: LLM/semantic cache + chat session memory
  data/    catalog + seed events, seed.py
  tests/   golden/      # small hand-labeled eval set  (added later)
alembic/ + alembic.ini  # schema migrations
docker-compose.yml      # postgres + qdrant + redis (local)
Dockerfile              # (added at containerize step)
.env.example            # provider key, model names, top-k, RRF weights, DB/Redis URL, LangSmith key
requirements.txt · CLAUDE.md · README.md · .gitignore
```
> LangSmith is not a folder — it's env vars (`LANGCHAIN_TRACING_V2`, `LANGCHAIN_API_KEY`)
> + LangGraph auto-tracing, wired via `config.py`.

## Repo
`OkBruh1234/smart-reco-challenge` (public). Hackathon CI workflow **removed**
(decoupled). Repo to be **renamed** later (likely `devup`) — deferred.