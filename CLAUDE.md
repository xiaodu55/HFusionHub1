# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Structure

HFusionHub is a **three-tier monorepo** enterprise AI Agent platform with a Java + Python hybrid architecture.

```
HFusionHub/
├── java-backend/           # Spring Boot 3.5.16 backend (port 8080, /api context-path)
├── python-ai/              # FastAPI AI service (port 9000)
├── hfusionhub-frontend/    # Vue 3 + Vite + TypeScript SPA (dev port 3000, prod port 80)
├── docker/                 # Dev Docker Compose: MySQL + Redis + MinIO + Milvus/etcd/Attu + Plugin Runner
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
python -m venv .venv
source .venv/bin/activate   # Linux/macOS
.venv\Scripts\activate      # Windows

pip install -r requirements.txt
pip install -r requirements-dev.txt

python -m app.main                           # Run dev server
pytest -q tests                              # Run all tests (1451 test functions)
pytest -q tests/test_adaptive_retrieval.py   # Run single test file
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
cd docker && docker compose up -d             # Dev: MySQL + Redis + MinIO + Milvus/etcd/Attu + Plugin Runner
docker compose -f deploy/docker-compose.prod.yml up -d  # Production: all 7 services
```

### First Deployment — Admin Setup
```bash
export ADMIN_PASSWORD=YourSecurePassword
cd java-backend && mvn spring-boot:run
# AdminInitializer creates the admin user automatically.
```

### CI
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
- **Java backend** owns auth (Sa-Token JWT), CRUD (MyBatis Plus + MySQL), file uploads, and SSE streaming. Proxies all AI requests to Python.
- **Python AI service** is the intelligence layer — only accessible internally via `X-Internal-Token` header.

### Java Backend (`java-backend/`)

- **Controllers** (55, 含 `bot/` 下 3 个 IM bot 控制器), **Services** (41 接口 + 4 独立 `@Service` 类), **Entities** (71 `@TableName` 实体 + 1 抽象基类) — all under `com.hfusionhub`
- **Auth**: Sa-Token with JWT. 86400s timeout, 1800s active timeout.
- **AI Client** (`client/AiClient.java`): HTTP → Python AI with `X-Internal-Token`. Sync chat, SSE streaming, cancellation.
- **Schedulers** (13): DocumentIndexRecovery, DeletionTaskProcessor, AgentRunRecovery, AgentRunTimeout, FeatureFlagSync, UsageLedgerAggregation, and others.

### Python AI Service (`python-ai/`)

- **Agent System**: `ReactAgent` (ReAct loop, max 5 steps), `SingleAgentWorkflow`, `BoundedMultiAgentWorkflow`
- **RAG Engine** (~31 modules): MultiChannelRetriever, QueryRouter, IntentClassifier, QueryDecomposer, ContextCompressor, SelfReflector, ScopedGraph, Reranker
- **LLM**: Abstract `BaseLLM`; implementations: DeepSeek (default), Ollama, Mock
- **Vector Store**: Milvus Standalone + external etcd（默认/生产, `VECTOR_STORE_MODE=cluster`）; Milvus Lite 仅本地裸跑/测试可选
- **Feature flags**: `RAG_HYBRID_ENABLED` (default true), `RAG_GRAPH_ENABLED`, `RAG_RERANKER_MODE`, `RAG_MULTIMODAL_ENABLED`, `RAG_AGENT_WORKFLOW_ENABLED`, `RAG_MULTI_AGENT_ENABLED`

### Frontend (`hfusionhub-frontend/`)

- Vue 3 + Pinia + Vue Router + Radix Vue + Tailwind CSS 4
- **API layer**: Axios with `satoken` header injection; 31 API modules
- **Pages** (43 routes across 17 business groups): Login, Register, Dashboard, Knowledge, Document, Chat, RAG observability, Agent, Memory, Plugins, Prompt, Settings, Admin, Builder, Bid, Cost, Embed, Notes, Profile 等

### Database

MySQL 8.0 with MyBatis Plus + Flyway (V1–V84). Key tables:
- Core: `sys_user`, `knowledge_base`, `document`, `document_chunk`, `document_index_job`
- Conversation: `conversation`, `message` (JSON `sources`, `token_count`)
- Agent: `agent_task`, `agent_run`, `agent_step`, `agent_approval`, `agent_status_event`
- Plugin: `plugin`, `plugin_audit_log`; Prompt: `prompt_template`, `prompt_test_set`
- Tenant: `tenant`, `tenant_member`, `role_permission`, `usage_quota`, `usage_ledger`
- Cost/Notes: `model_usage_record`, `note` (写笔记闭环), `kb_share`, `app`/`app_api_key`
- Flyway: new schema changes must use **V85+** scripts. Never modify existing V1–V84.
- **新表必须含 `tenant_id` 列**（除非加入 `MybatisPlusConfig.TENANT_IGNORE_TABLES`）；CI `scripts/static-checks.py` 静态校验

## Key Data Flows

### Document Processing
1. Upload → Java saves file, creates document (status=PROCESSING)
2. Java → Python `/api/parse`: parse → chunk (500/50) → embed → insert Milvus → callback
3. Java callback → update `document_index_job` + `document_chunk` → COMPLETED
4. Recovery: `DocumentIndexRecoveryScheduler` rescues stale jobs

### Chat
1. User message → Java saves MySQL → Python `/api/chat/stream`
2. Python: intent → retrieval → compression → ReAct loop → reflection → citations
3. SSE: Python → Java `SseEmitter` → Frontend
4. Java saves assistant response (content, sources, model, token_count)

## Testing

| Subproject | Runner | Test count |
|---|---|---|
| python-ai | pytest + pytest-asyncio | 1451 |
| java-backend | JUnit 5 + H2 | 702 |
| frontend | Vitest + Playwright | 57 unit + 73 E2E |

Java tests use H2 in-memory (MySQL compatibility mode). Flyway disabled in tests; schema from `schema-h2.sql`.

## Design Patterns

- **Strategy pattern**: Intent classifier, self-reflector, query router, multi-turn strategy
- **CQRS-like**: Java owns writes (ACID), Python owns reads/retrieval
- **Idempotent indexing**: `document_index_job.index_version` prevents duplicate processing
- **Defense in depth**: KB ownership verified at Java (Sa-Token) and Python (callback secrets)

## Project Docs

> 完整文档索引见 [README.md](README.md#-文档索引documentation-index)（含每个 `docs/` 文件的作用说明）。

- [README.md](README.md) — 项目总览与快速开始
- [docs/ACCESS_MAP.md](docs/ACCESS_MAP.md) — 已启动服务全量访问地图（网址/账号/API 文档/接口清单）
- [docs/startup-guide.md](docs/startup-guide.md) — 中英双语启动/重启/排障指南（原 启动重启1.md 已合并）
- [docs/ENVIRONMENT.md](docs/ENVIRONMENT.md) — 环境变量清单（唯一权威）
- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) — 架构全景图（含白皮书）
- [docs/PRODUCTION_OPS.md](docs/PRODUCTION_OPS.md) — 生产运维手册（含上线检查清单 + 向量库容灾）
- [docs/SCALING.md](docs/SCALING.md) — 扩容与性能手册（含性能基线）
- [docs/ROADMAP.md](docs/ROADMAP.md) — 路线图（含评估快照）
- [docs/java-backend.md](docs/java-backend.md) — Java 后端开发指南
- [docs/python-ai.md](docs/python-ai.md) — Python AI 开发指南
- [docs/database.md](docs/database.md) — 数据库设计
- [AGENTS.md](AGENTS.md) — AI coding agent guidance (shared with Codex)

## Quick Start (Dev)

```bash
# 1. Docker
cd docker && docker compose up -d

# 2. Java
cd java-backend && mvn spring-boot:run

# 3. Python
cd python-ai && .venv\Scripts\activate && python -m app.main

# 4. Frontend
cd hfusionhub-frontend && npm run dev
```

Admin: `admin` / value of `ADMIN_PASSWORD`
