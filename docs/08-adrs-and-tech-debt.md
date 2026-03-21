# Compliance Brain - ADRs & Known Technical Debt

> Architecture Decision Records (ADRs) and known technical debt to address in future phases.

---

## Architecture Decision Records

### ADR-001: Hybrid DDD Architecture
**Date**: 2026-03-21
**Status**: Accepted
**Context**: Full DDD (domain entities + ORM models + mappers + repositories + commands/queries) for all 9 bounded contexts adds ~40% more code. Most contexts are simple CRUD.
**Decision**: Use three architecture levels — Full DDD only for AML (complex invariants), Service Layer for contexts with moderate logic (Client, Compliance, Workflow), Simple CRUD for the rest.
**Consequences**: Less code, faster delivery, junior-friendly. Risk: if Client or Compliance grows complex, will need to promote to full DDD (effort but not risky).

### ADR-002: SQLAlchemy Models as Source of Truth (non-DDD contexts)
**Date**: 2026-03-21
**Status**: Accepted
**Context**: Maintaining separate domain entities (dataclasses) + ORM models + mappers for every entity is overhead when the domain shape equals the DB shape.
**Decision**: For Service Layer and Simple CRUD contexts, SQLAlchemy models ARE the domain. Business logic lives in service.py. Pydantic handles validation.
**Consequences**: Fewer files, simpler mental model. Domain logic is testable with test DB session. Trade-off: domain logic depends on SQLAlchemy (acceptable for this project).

### ADR-003: No Separate Unit of Work Wrapper
**Date**: 2026-03-21
**Status**: Accepted
**Context**: SQLAlchemy's AsyncSession already implements Unit of Work (flush, commit, rollback). A custom UoW wrapper adds abstraction without value.
**Decision**: Use AsyncSession directly via `Depends(get_db_session)`. Session lifecycle managed per-request by FastAPI dependency.
**Consequences**: Simpler code. If multi-repo transactions needed later, wrap session then.

### ADR-004: No BFF Proxy (Frontend calls Backend directly)
**Date**: 2026-03-21
**Status**: Accepted
**Context**: Next.js API routes as BFF proxy adds latency, code, and a point of failure for < 10 users on the same network.
**Decision**: Frontend calls FastAPI backend directly. CORS configured. JWT in Authorization header.
**Consequences**: Less code, less latency. If we need server-side token exchange or caching later, add BFF then.

### ADR-005: Client-Side Rendering Only (No SSR)
**Date**: 2026-03-21
**Status**: Accepted
**Context**: Internal tool with < 10 users, no SEO requirements. SSR adds complexity (hydration, server-side auth, data fetching patterns).
**Decision**: All pages use `'use client'`. Supabase Auth works client-side. TanStack Query handles data fetching and caching.
**Consequences**: Simpler auth flow, no hydration issues. Trade-off: slightly slower initial page load (acceptable for internal tool).

### ADR-006: Supabase for DB + Auth + Storage
**Date**: 2026-03-21
**Status**: Accepted
**Context**: Need managed PostgreSQL with RLS, authentication with MFA, and file storage. Supabase provides all three.
**Decision**: Supabase Cloud for production. Local dev uses plain PostgreSQL (Docker) with auth bypass middleware.
**Consequences**: Single vendor for three services ($25/mo Pro). Portable: PostgreSQL is standard, auth is abstractable, storage is abstractable. Risk: Supabase downtime affects all three (acceptable at this scale).

### ADR-007: No Redis, No Celery, No Message Queue
**Date**: 2026-03-21
**Status**: Accepted
**Context**: < 10 users, simple background tasks (email, reminders). Celery requires worker process, beat scheduler, Redis broker — operational overhead.
**Decision**: FastAPI BackgroundTasks for fire-and-forget. pg_cron for scheduled SQL. HTTP endpoints + external cron for scheduled Python jobs.
**Consequences**: Zero extra infrastructure. Scale to task queue only if jobs take > 30s or need retry logic.

### ADR-008: No Terraform (Phase 1)
**Date**: 2026-03-21
**Status**: Accepted
**Context**: Infrastructure is Supabase (configured via dashboard) + Docker container (deployed via CI). No infrastructure to manage programmatically.
**Decision**: Docker Compose for local dev. Manual/CI deployment. Add Terraform when deploying to GCP or managing multiple environments.
**Consequences**: Simpler setup. Risk: manual infra config (acceptable for one environment).

### ADR-009: Async/Await End-to-End
**Date**: 2026-03-21
**Status**: Accepted
**Context**: FastAPI is async-native. Mixing sync and async causes event loop blocking and performance issues.
**Decision**: Everything async. asyncpg for DB, httpx for HTTP, asyncio.to_thread() for CPU-bound sync code. Zero sync HTTP clients or DB drivers.
**Consequences**: Consistent async patterns. All developers must understand async/await. Claude Code helps with this.

### ADR-010: Screening Provider Abstraction
**Date**: 2026-03-21
**Status**: Accepted
**Context**: Phase 1 uses manual upload for screening results. Phase 2 needs API integration with World-Check/ComplyAdvantage.
**Decision**: Define ScreeningProvider Protocol now. Implement ManualUploadProvider for Phase 1. Swap to API providers in Phase 2 without changing domain logic.
**Consequences**: Clean abstraction boundary. Phase 2 integration is a new provider class, not a refactor.

### ADR-011: State Machine as Dict (No Library)
**Date**: 2026-03-21
**Status**: Accepted
**Context**: Multiple entities have state machines (Client 6 states, Onboarding 8 states, Policy 5 states, Incident 5 states). Libraries like `transitions` add dependency.
**Decision**: Simple `StateMachine` class with transition dict in shared kernel. Each context defines its own transitions as a plain dict.
**Consequences**: Zero dependencies, easily testable, copy-paste pattern. Trade-off: no guard conditions or callbacks (add if needed later).

### ADR-012: Light Tracking (Not Full Bounded Contexts)
**Date**: 2026-03-21
**Status**: Accepted
**Context**: Authorisations and Entity Management are "light" in Phase 1 (tracking + dates only). Full DDD bounded contexts for CRUD tables is over-engineering.
**Decision**: Simple tables (`light_auth_cases`, `light_entities`) with CRUD routes. No domain logic, no events. Promote to full bounded contexts in Phase 2.
**Consequences**: Fast to implement, easy to understand. Phase 2 migration: create new tables with full schema, migrate data from light tables.

---

## Known Technical Debt

### HIGH Priority (Address in Phase 1.5 or Phase 2)

| ID | Debt | Impact | When to Address |
|----|------|--------|-----------------|
| TD-001 | **CRA Excel not imported** — Risk scoring uses manual config, not the client's actual scoring matrix | Risk scores may not match client expectations | When client provides CRA Excel |
| TD-002 | **No virus scanning on file uploads** — Files accepted without malware check | Security risk for uploaded documents | Phase 2: integrate ClamAV or cloud scanning |
| TD-003 | **Screening is manual upload only** — No API integration with World-Check/ComplyAdvantage | Manual process, slower screening | Phase 2: implement WorldCheckProvider |
| TD-004 | **No WebSocket notifications** — All notifications are email only | Users must check email or refresh dashboard | Phase 2: add WebSocket for real-time alerts |
| TD-005 | **Client context may need DDD promotion** — Currently Service Layer, but if UBO/PEP logic grows complex, may need full domain entities | Potential refactor in Phase 2 | When PEP escalation rules multiply |
| TD-006 | **No full-text search on documents** — Only search by metadata, not document content | Users can't find text inside uploaded PDFs | Phase 2: add document content indexing |
| TD-007 | **Light tracking → full bounded contexts** — `light_auth_cases` and `light_entities` need migration to full `auth_cases` and `ent_legal_entities` | Data migration needed in Phase 2 | Phase 2: when auth/entity workflows are implemented |

### MEDIUM Priority (Phase 2+)

| ID | Debt | Impact | When to Address |
|----|------|--------|-----------------|
| TD-008 | **No async event bus** — All cross-context communication is synchronous in-process | If one handler fails, the whole request fails | Phase 2: when context separation needs resilience |
| TD-009 | **No read replicas** — Single DB instance for reads and writes | Not a problem for < 50 users | Phase 3: when query load increases |
| TD-010 | **No canary deployments** — All-or-nothing deploys | Risky for major releases | Phase 2: add deployment strategy |
| TD-011 | **No i18n** — English only | Can't serve Arabic-speaking users | When/if Arabic support is requested |
| TD-012 | **No digital signatures** — Documents can't be signed in-platform | Users sign outside the platform | Phase 2: integrate e-signature provider |
| TD-013 | **No board pack auto-generation** — KPIs displayed but full board pack PDF not generated | Board packs assembled manually | Phase 2: Board Pack Builder agent |
| TD-014 | **No document preview in browser** — Users must download to view | Extra clicks for document review | Phase 2: add PDF.js viewer |
| TD-015 | **Supabase Python SDK is partially sync** — Using httpx directly for async calls | Slightly more code than using SDK | Non-issue: httpx is clean and portable |

### LOW Priority (Nice to have)

| ID | Debt | Impact | When to Address |
|----|------|--------|-----------------|
| TD-016 | **No API rate limiting per tenant** — Rate limiting is per-IP only | One tenant could consume all resources | When multiple tenants with heavy usage |
| TD-017 | **No incident response plan for the platform** — Only client compliance incidents tracked | No documented process for platform outages | Before Phase 2 go-live |
| TD-018 | **No backup restore testing automation** — Monthly manual restore tests | Could miss a broken backup | When ops team grows |
| TD-019 | **Audit trail export is basic CSV** — No advanced filtering or regulatory report formats | Regulators may want specific formats | When a regulator actually asks |
| TD-020 | **No chaos engineering / failover testing** — No systematic resilience testing | Unknown failure modes | Phase 3: when scaling to 50+ users |
