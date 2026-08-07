# AGENTS.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Structure

HFusionHub is a **three-tier monorepo** enterprise AI Agent platform with a Java + Python hybrid architecture.

```
HFusionHub/
├── java-backend/           # Spring Boot 3.2.5 backend (port 8080, /api context-path)
├── python-ai/              # FastAPI AI service (port 9000)
├── hfusionhub-frontend/    # Vue 3 + Vite + TypeScript SPA (dev port 3000, prod port 80)
├── docker/                 # Dev Docker Compose: MySQL 8.0 + Redis 7 + MinIO + Plugin Runner
├── deploy/                 # Production Docker Compose, Dockerfiles, Helm chart, monitoring
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
npx vitest run                                # Run unit tests
npx playwright test                           # Run E2E tests
```

### Infrastructure
```bash
cd docker && docker compose up -d             # Dev: MySQL + Redis + MinIO + Plugin Runner
docker compose -f deploy/docker-compose.prod.yml up -d  # Production: all 7 services
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
# run in parallel. See docs/CI_GATES.md.
```

## Architecture Overview

### Three-Tier Communication

```
Frontend (Vue 3 :3000 dev / :80 prod) ──HTTP/SSE──> Java Backend (:8080)
                                                          │
                                                    HTTP + X-Internal-Token
                                                          │
                                                   Python AI Service (:9000)
```

- **Frontend** communicates only with Java backend (via Axios). Vite proxies `/api` to `:8080`.
- **Java backend** owns auth (Sa-Token JWT), CRUD (MyBatis Plus + MySQL), file uploads, and SSE streaming. It proxies all AI requests to the Python service.
- **Python AI service** is the intelligence layer — only accessible internally via `X-Internal-Token` header. All business endpoints except `/health` require this token (HMAC constant-time comparison).

### Java Backend (`java-backend/`)

Standard **Controller → Service → Mapper → Entity** layered architecture under `com.hfusionhub`.

- **Controllers** (25): Auth, User, KnowledgeBase, Document (3), Conversation (2, SSE streaming), Vectorization (2, callbacks), RagObservability, Agent (3: task/approval/status), Admin, Memory, Notification, Plugin, PromptTemplate, PromptTestSet, System, Tenant, FeatureFlag, Health, Metric, MCP
- **Services** (~25): Matching service layer with interfaces + implementations
- **Entities** (42): User, KnowledgeBase, Document, DocumentChunk, DocumentIndexJob, Conversation, Message, MemoryEntry, SystemNotice, NoticeRecipient, AgentTask, AgentRun, AgentStep, AgentApproval, AgentStatusEvent, AgentAlertRule, AgentAlertEvent, AgentEvaluationDataset, Plugin, PluginAuditLog, PromptTemplate, PromptTestSet, PromptTestSetRun, FeatureFlag, Tenant, TenantMember, RolePermission, UsageQuota, UsageLedger, DeletionTask, and more — all extend `BaseEntity` (createdAt, updatedAt, deleted logical delete)
- **Auth**: Sa-Token with JWT. 86400s timeout, 1800s active timeout. All paths except `/user/login`, `/user/register`, `/vectorize/**/callback`, and Swagger docs require login.
- **AI Client** (`client/AiClient.java`): HTTP calls to Python AI with `X-Internal-Token`. Supports sync chat, SSE streaming, and cancellation.
- **Schedulers** (13): DocumentIndexRecovery, DocumentOrphanRecovery, DeletionTaskProcessor, AgentRunRecovery, AgentRunTimeout, AgentApprovalExpiry, AgentStatusEventRetention, PromptTestSetRunRecovery, PromptTestSetRunTimeout, UsageLedgerAggregation, PluginAuditRetention, FeatureFlagSync, TenantQuotaEnforcement

### Python AI Service (`python-ai/`)

Modular domain organization — the most architecturally complex subproject.

**Agent System** (`app/core/agent/`):
- `ReactAgent` — ReAct loop: Thought/Action/Observation cycle (max 5 steps)
- `SingleAgentWorkflow` — bounded wrapper with timeout/retry (disabled by default)
- `BoundedMultiAgentWorkflow` — agent + evidence critic (disabled by default)

**RAG Engine** (`app/core/rag/`) — ~31 modules:
- `MultiChannelRetriever` orchestrates rewriting → routing → RRF fusion → reranking → postprocessing
- `QueryRouter` routes to Vector (Milvus), Keyword (BM25), and Graph channels
- `IntentClassifier`, `QueryDecomposer`, `ContextCompressor`, `SelfReflector`
- `ScopedGraph` — KB-scoped GraphRAG; `Reranker` — optional second-stage

**LLM & Embedding** (`app/core/llm/`, `app/core/embedding/`):
- Abstract `BaseLLM`; implementations: DeepSeek (default), Ollama, Mock
- Embedding: Ollama (primary) with `EMBEDDING_ALLOW_FALLBACK` guard

**Vector Store** (`app/core/vectorstore/`):
- Dev: Milvus Lite (`milvus_data.db`)
- Prod: Milvus standalone/cluster (`VECTOR_STORE_MODE=cluster`, `MILVUS_HOST`/`MILVUS_PORT`)

**Key env feature flags** (in `.env.example`):
- `RAG_HYBRID_ENABLED` (default true), `RAG_GRAPH_ENABLED`, `RAG_RERANKER_MODE`
- `RAG_MULTIMODAL_ENABLED`, `RAG_AGENT_WORKFLOW_ENABLED`, `RAG_MULTI_AGENT_ENABLED`

### Frontend (`hfusionhub-frontend/`)

Feature-based SPA with Vue 3 + Pinia + Vue Router.

- **API layer** (`src/api/`): Axios with `satoken` header injection, modules for auth, kb, doc, conversation, agent, approval, memory, plugins, promptTemplate, promptTestSet, system, tools, rag
- **Pages** (13): Login, Register, Dashboard, Knowledge (list/detail/chunks), Document, Chat (list/detail), RAG observability, Agent tasks/approvals, Memory, Plugins (builder/manager), Prompt templates/test sets, Settings (profile/admin), Admin (users/tenants)
- **Components**: Radix Vue UI primitives, MarkdownRenderer (marked + dompurify), ToastContainer
- **Router**: Auth guard → `/login`. MainLayout for authenticated routes.

### Database

MySQL 8.0 with MyBatis Plus + Flyway (V1–V35). Key tables:
- Core: `sys_user`, `knowledge_base`, `document`, `document_chunk`, `document_index_job`
- Conversation: `conversation`, `message` (JSON `sources`, `token_count`)
- Agent: `agent_task`, `agent_run`, `agent_step`, `agent_approval`, `agent_status_event`
- Memory/Notice: `memory_entry`, `system_notice`, `notice_recipient`
- Plugin: `plugin`, `plugin_audit_log`, `plugin_image_digest`
- Prompt: `prompt_template`, `prompt_test_set`, `prompt_test_set_run`
- Tenant: `tenant`, `tenant_member`, `role_permission`, `usage_quota`, `usage_ledger`
- Feature flags: `feature_flag`
- Logical delete via `deleted` column on all major tables
- Admin: created via `ADMIN_PASSWORD` env var by `AdminInitializer`
- Flyway: `java-backend/src/main/resources/db/migration/` (V1–V35; new scripts must be V36+)

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
| python-ai | pytest + pytest-asyncio | 1220+ test cases across 27+ files | `python-ai/tests/` |
| java-backend | JUnit 5 + H2 (spring-boot-starter-test) | 397 tests | `java-backend/src/test/` |
| frontend | Vitest + Playwright | 32 unit + E2E | `hfusionhub-frontend/src/__tests__/` |

Java tests use H2 in-memory database (MySQL compatibility mode) via the `test` Spring profile.
Flyway is disabled in tests; schema is loaded from `src/test/resources/schema-h2.sql`.
For full integration tests against real MySQL, use the `itest` Maven profile (`mvn test -P itest`).

## Design Patterns

- **Strategy pattern**: Intent classifier, self-reflector, query router, multi-turn strategy — all with factory creation
- **Singleton**: Global instances for retriever, config, trace store, knowledge graph manager, reflector
- **Fallback chains**: Embedding (Ollama → fail closed; random only with `EMBEDDING_ALLOW_FALLBACK=true`)
- **CQRS-like**: Java owns write path (documents, conversations), Python owns read/retrieval path
- **Idempotent indexing**: `document_index_job.index_version` prevents duplicate processing
- **Defense in depth**: KB ownership verified at Java (Sa-Token) and Python (callback secrets)
