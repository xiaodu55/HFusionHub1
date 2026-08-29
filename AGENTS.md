# AGENTS.md

> Guidance for AI coding agents (Claude Code, Codex, etc.) working in this repository.
> **The authoritative repository guide is [CLAUDE.md](CLAUDE.md)** — this file adds agent-specific notes and points to the canonical reference. Keep both in sync for test counts and migration versions.

## TL;DR

- **Architecture**: Java + Python hybrid three-tier (see [CLAUDE.md](CLAUDE.md) → Architecture Overview). Java owns writes (ACID), Python owns reads/intelligence.
- **Flyway**: current `V1–V74`; new scripts must be **V75+**. Never modify existing migrations. New tables MUST include `tenant_id` (unless in `TENANT_IGNORE_TABLES`).
- **Tests**: Java 563 · Python 1407 · Frontend 49.
- **Commands, data flows, design patterns, project docs**: all in [CLAUDE.md](CLAUDE.md).

## Agent-Specific Notes

### Reading the codebase first

1. Read [CLAUDE.md](CLAUDE.md) (canonical), [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md), [docs/database.md](docs/database.md).
2. Follow `docs/` — each file's purpose is indexed in [README.md](README.md#文档索引-documentation-index).

### Implementation rules

- **Schema changes**: create a new Flyway `V75+` script; never touch V1–V74. Always add `tenant_id` to new tables.
- **Internal endpoints** (Java ↔ Python): must be protected by `X-Internal-Token` / `CALLBACK_SECRET`; keep the token key names consistent (see `scripts/static-checks.py`, enforced in CI).
- **Feature flags**: gate new AI capabilities behind env-var flags (see [docs/ENVIRONMENT.md](docs/ENVIRONMENT.md) — P5 stable ON, experimental default OFF, frozen flags stay OFF).
- **Don't hardcode secrets**: read from `docker/.env` / `deploy/.env` / env vars; never commit `.env` or plaintext passwords.

### Verification before completing a change

- Java: `cd java-backend && mvn test`
- Python: `cd python-ai && .venv\Scripts\activate && pytest -q tests`
- Frontend: `cd hfusionhub-frontend && npm run build && npx vitest run`
- Full stack: `.\scripts\smoke-test.ps1` (expect 47 PASS / 0 FAIL)
- Static consistency: `python scripts/static-checks.py`

### CI gates

See [docs/CI_GATES.md](docs/CI_GATES.md) — Compose validation, Python tests, Java tests (+Flyway), Frontend build + audit, and the offline eval gate run in parallel.

---
*HFusionHub — Built with Java's reliability and Python's AI ecosystem.*
