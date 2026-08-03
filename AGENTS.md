# AGENTS.md

This file provides guidance to Codex (Codex.ai/code) when working with code in this repository.

## Repository Structure

HFusionHub is a **three-tier monorepo** enterprise AI Agent platform with a Java + Python hybrid architecture.

```
HFusionHub/
├── java-backend/           # Spring Boot 3.2.5 backend (port 8080)
├── python-ai/              # FastAPI AI service (port 9000)
├── hfusionhub-frontend/    # Vue 3 + Vite + TypeScript SPA (port 3000)
├── docker/                 # Docker Compose: MySQL 8.0 + Redis 7
├── milvus-docker/          # REMOVED — Python uses Milvus Lite
├── scripts/                # PowerShell verification scripts
└── .github/workflows/      # CI pipeline
```

## Key Commands

### Java Backend
```bash
cd java-backend
mvn spring-boot:run          # Run dev server
mvn test                      # Run all tests (H2 in-memory DB)
mvn test -Dtest=HealthControllerTest  # Run single test
mvn test -P itest             # Run with Testcontainers (needs Docker)
mvn package                   # Build JAR
```

### Python AI Service
```bash
cd python-ai
# Use a project-local virtual environment
python -m venv .venv
source .venv/bin/activate   # Linux/macOS
.venv\Scripts\activate      # Windows

pip install -r requirements.txt              # Install deps (pip-compile locked)
pip install -r requirements-dev.txt          # Install test deps
# To update locked dependencies:
# pip-compile requirements.in requirements-dev.in

python -m app.main                           # Run dev server
pytest -q tests                              # Run all tests
pytest -q tests/test_retriever.py            # Run single test file
pytest -q -k "test_intent_classify"          # Run specific test
```

### Frontend
```bash
cd hfusionhub-frontend
npm ci                                        # Install deps (clean)
npm run dev                                   # Run dev server
npm run build                                 # Type-check + build
npm run preview                               # Preview production build
```

### Infrastructure
```bash
cd docker && docker compose up -d             # Start MySQL + Redis
```

### First Deployment — Admin Setup
```bash
# Set ADMIN_PASSWORD to bootstrap the admin account on first startup
export ADMIN_PASSWORD=YourSecurePassword
cd java-backend && mvn spring-boot:run
# AdminInitializer creates the admin user automatically.
# Without ADMIN_PASSWORD, no admin account is created.
```

### CI (runs on push/PR to main — parallel jobs)
```bash
# Docker Compose validation, Python tests, Java tests (+ Flyway migration),
# Frontend build + audit, and the Phase-2 offline evaluation gate (eval-offline)
# run in parallel. eval-offline runs python-ai/scripts/eval_offline.py
# --fail-on-regression — gate failure or baseline regression blocks the merge.
# Nightly runtime evaluation runs via .github/workflows/eval-nightly.yml.
# See docs/CI_GATES.md.
```

## Architecture Overview

### Three-Tier Communication

```
Frontend (Vue 3 :3000)  ──HTTP/SSE──>  Java Backend (:8080)
                                           │
                                     HTTP + X-Internal-Token
                                           │
                                    Python AI Service (:9000)
```

- **Frontend** communicates only with Java backend (via Axios). Port 3000 proxies `/api` to `:8080`.
- **Java backend** owns auth (Sa-Token JWT), CRUD (MyBatis Plus + MySQL), file uploads, and SSE streaming. It proxies all AI requests to the Python service.
- **Python AI service** is the intelligence layer — only accessible internally via `X-Internal-Token` header. All business endpoints except `/health` require this token (HMAC constant-time comparison).

### Java Backend (`java-backend/`)

Standard **Controller → Service → Mapper → Entity** layered architecture under `com.hfusionhub`.

- **Controllers** (7): HealthController, UserController, KnowledgeBaseController, DocumentController, ConversationController (SSE streaming), VectorizationController (callbacks), RagObservabilityController
- **Services** (5): UserServiceImpl, KnowledgeBaseServiceImpl, DocumentServiceImpl (file upload + vectorization trigger), ConversationServiceImpl (chat + SSE forwarding), VectorizationServiceImpl (index job management)
- **Entities** (7): User, KnowledgeBase, Document, DocumentChunk, DocumentIndexJob, Conversation, Message — all extend `BaseEntity` (createdAt, updatedAt, deleted logical delete)
- **Auth**: Sa-Token with JWT. 86400s timeout, 1800s active timeout. All paths except `/user/login`, `/user/register`, `/vectorize/**/callback`, and Swagger docs require login.
- **AI Client** (`client/AiClient.java`): HTTP calls to Python AI at `localhost:9000` with `X-Internal-Token`. Supports sync chat, SSE streaming, and cancellation.
- **Scheduler**: `DocumentIndexRecoveryScheduler` recovers stale index jobs (30 min stale threshold, 3 max attempts).

### Python AI Service (`python-ai/`)

Modular domain organization — the most architecturally complex subproject.

**Agent System** (`app/core/agent/`):
- `ReactAgent` — ReAct loop: Thought/Action/Observation cycle (max 5 steps), intent classification, RAG retrieval, context compression, self-reflection, source citations
- `SingleAgentWorkflow` — bounded wrapper with timeout/retry (disabled by default, `RAG_AGENT_WORKFLOW_ENABLED`)
- `BoundedMultiAgentWorkflow` — runs agent then validates evidence with deterministic critic (disabled by default, `RAG_MULTI_AGENT_ENABLED`)
- Agent chaining: `get_agent()` factory wraps ReactAgent → SingleAgentWorkflow → BoundedMultiAgentWorkflow

**RAG Engine** (`app/core/rag/`) — ~25 modules:
- `MultiChannelRetriever` orchestrates rewriting → routing → RRF fusion → reranking → postprocessing
- `QueryRouter` routes to Vector (Milvus), Keyword (BM25), and Graph channels with weighted RRF fusion
- `IntentClassifier` supports LLM/Rule/Hybrid strategies with caching
- `QueryDecomposer` splits complex queries into dependency-graph sub-questions
- `ContextCompressor` extractively compresses to ~60% target ratio
- `SelfReflector` evaluates answer quality via LLM/Rule/Hybrid strategies
- `ScopedGraph` — KB-scoped GraphRAG with deterministic entity co-occurrence
- `Reranker` — optional cross-encoder or lexical second-stage reranking
- Pattern: Strategy pattern used pervasively (intent classifier, router, reflector, multi-turn). Singletons for retriever, router, config, trace store.

**LLM & Embedding** (`app/core/llm/`, `app/core/embedding/`):
- Abstract `BaseLLM` with `chat()`, `chat_stream()`, `ainvoke()`
- Implementations: DeepSeek (default), Ollama, Mock
- Embedding uses DeepSeek or Ollama (DeepSeek has no real embedding API — falls back to random vectors)

**Vector Store** (`app/core/vectorstore/`):
- Milvus Lite with persistent file `milvus_data.db`
- 1024-dim FLOAT_VECTOR, COSINE metric, IVF_FLAT index
- Local `chunks_store.json` fallback

**Key env feature flags** (all disabled by default, in `.env.example`):
- `RAG_GRAPH_ENABLED`, `RAG_RERANKER_ENABLED`, `RAG_MULTIMODAL_ENABLED`
- `RAG_AGENT_WORKFLOW_ENABLED` (P9), `RAG_MULTI_AGENT_ENABLED` (P10)

### Frontend (`hfusionhub-frontend/`)

Feature-based SPA with Vue 3 + Pinia + Vue Router.

- **API layer** (`src/api/`): Axios instance injects `satoken` header, handles 401 redirect. Modules per domain (user, knowledgeBase, document, conversation, vectorization, rag).
- **Pages** (8): Login, Register, Dashboard, Knowledge (list/detail/chunks), Document, Chat (list/detail), RAG observability, Profile
- **Components**: Radix Vue-based UI primitives (button, card, dialog, input, select, badge), MarkdownRenderer (marked + dompurify), ToastContainer
- **Router**: Auth guard redirects to `/login`. MainLayout for authenticated routes.

### Database

MySQL with MyBatis Plus. Key tables:
- `sys_user`, `knowledge_base`, `document`, `document_chunk`, `document_index_job`
- `conversation`, `message` (with JSON `sources` field and `token_count`)
- Logical delete via `deleted` column on all major tables
- Admin user is created via `ADMIN_PASSWORD` env var (no default password)
- Flyway migrations at `java-backend/src/main/resources/db/migration/`
- Legacy SQL scripts at `java-backend/src/main/resources/sql/`

## Key Data Flows

### Document Processing
1. User uploads → Java saves file, creates document (status=PROCESSING)
2. User triggers parse → Java sends HTTP to Python AI `/api/parse`
3. Python AI: parse → chunk (500 chars, 50 overlap) → embed → insert Milvus → update scoped graph → callback Java
4. Java callback → update `document_index_job` + `document_chunk` → mark document COMPLETED
5. Recovery: `DocumentIndexRecoveryScheduler` rescues stale jobs on restart

### Chat
1. User sends message → Java saves to MySQL → calls Python AI `/api/chat` or `/api/chat/stream`
2. Python AI: intent classification → RAG retrieval → context compression → ReAct loop → self-reflection → source citations
3. SSE chunks flow: Python AI → Java `SseEmitter` → Frontend
4. Java saves assistant response (content, sources, model, token_count) to MySQL

## Testing

| Subproject | Runner | Test count | Location |
|---|---|---|---|
| python-ai | pytest + pytest-asyncio | 615+ test cases across 27 files | `python-ai/tests/` |
| java-backend | JUnit 5 + H2 (spring-boot-starter-test) | 4 tests | `java-backend/src/test/` |
| frontend | npm audit only (no test framework) | — | — |

Java tests use H2 in-memory database (MySQL compatibility mode) via the `test` Spring profile.
Flyway is disabled in tests; schema is loaded from `src/test/resources/schema-h2.sql`.
For full integration tests against real MySQL, use the `itest` Maven profile (`mvn test -P itest`).

## Design Patterns

- **Strategy pattern**: Intent classifier, self-reflector, query router, multi-turn strategy all use strategy pattern with factory creation
- **Singleton**: Global instances for retriever, config, trace store, knowledge graph manager, reflector
- **Fallback chains**: Embedding (Ollama → DeepSeek → random), Reranker (cross_encoder → lexical → disabled)
- **CQRS-like**: Java owns write path (documents, conversations), Python owns read/retrieval path
- **Idempotent indexing**: `document_index_job.index_version` prevents duplicate processing
- **Defense in depth**: KB ownership verified at Java (Sa-Token) and Python (callback secrets)

## Claude Code 会话记忆

当需要了解该项目在此前的 Claude Code 会话中讨论过的问题、已完成的修复或决策时，读取 `CLAUDE_MEMORY.md`（精简提取版，约 48KB）。该文件由脚本从 `~/.claude/projects/d--college-development-0-HFusionHub/*.jsonl` 会话记录自动提取，仅含有效用户提问与 AI 回答。原始完整版为 `CLAUDE_SESSIONS_MEMORY.md`（约 2MB，含工具调用细节，非必要不读取）。如需重新生成：运行 `python C:\Users\15790\AppData\Local\Temp\opencode\extract_claude_sessions.py`。
