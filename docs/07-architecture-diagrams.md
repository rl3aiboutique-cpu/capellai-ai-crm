# Compliance Brain - Architecture Diagrams

> Simplified C4-style architecture views. Bullet-point explanations, not essays.

---

## C4 Level 1: System Context

```
                    ┌─────────────────┐
                    │   Compliance     │
                    │   Officers &     │
                    │   Founders       │
                    │   (< 10 users)   │
                    └────────┬────────┘
                             │ uses
                             ▼
                    ┌─────────────────┐
                    │  COMPLIANCE     │
                    │  BRAIN          │
                    │  PLATFORM       │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │  Screening   │ │  Regulatory  │ │   Email      │
    │  Providers   │ │  Sources     │ │  (Resend)    │
    │  (Phase 2)   │ │  (DFSA,FSRA) │ │              │
    └──────────────┘ └──────────────┘ └──────────────┘
```

**Actors:**
- Compliance Officers, MLRO, Analysts — daily users managing clients and onboarding
- Founders (Isahaq, Sagheer) — strategic oversight, approvals, configuration
- AI Agent (Claude) — drafts risk scores, triage screening hits (human confirms)

**External Systems:**
- Screening providers (World-Check, ComplyAdvantage) — Phase 2 API integration, Phase 1 manual upload
- Regulatory sources (DFSA, FSRA, UAE CB, VARA) — weekly scan for changes
- Email service (Resend) — reminders, alerts, client notifications
- Supabase — managed PostgreSQL + Auth + Storage

---

## C4 Level 2: Container Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         USERS (Browser)                         │
└──────────────────────────────┬──────────────────────────────────┘
                               │ HTTPS
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  FRONTEND                                                       │
│  Next.js 15 + React 19 + TypeScript                             │
│  Tailwind CSS + shadcn/ui                                       │
│  Client-side rendering ('use client')                           │
│  TanStack Query for API calls                                   │
│  ~15 screens (wireframes reference)                             │
│  Deployed: Vercel / static hosting                              │
└──────────────────────────────┬──────────────────────────────────┘
                               │ REST API (JSON)
                               │ JWT in Authorization header
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  BACKEND                                                        │
│  Python 3.12+ + FastAPI + Pydantic v2                           │
│  Hybrid DDD architecture                                        │
│  ~60 API endpoints across 9 contexts                            │
│  Async/await end-to-end (asyncpg, httpx)                        │
│  Deployed: Docker container (Cloud Run / Fly.io / Railway)      │
└───────┬──────────────┬──────────────┬──────────────┬────────────┘
        │              │              │              │
        ▼              ▼              ▼              ▼
┌──────────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐
│  Supabase    │ │ Supabase │ │ Supabase │ │  External    │
│  PostgreSQL  │ │ Auth     │ │ Storage  │ │  APIs        │
│  (RLS)       │ │ (JWT+MFA)│ │ (files)  │ │  (Resend,    │
│              │ │          │ │          │ │   Claude,    │
│              │ │          │ │          │ │   Screening) │
└──────────────┘ └──────────┘ └──────────┘ └──────────────┘
```

**Key decisions:**
- Frontend calls backend directly (no BFF proxy) — CORS configured
- All communication is REST/JSON — no GraphQL, no WebSocket (Phase 1)
- JWT from Supabase Auth flows through every request
- Backend sets `app.current_tenant` per request for PostgreSQL RLS

---

## C4 Level 3: Backend Components

```
┌─────────────────────────────────────────────────────────────────┐
│  FASTAPI BACKEND                                                │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  MIDDLEWARE (cross-cutting, every request)                  │ │
│  │  auth → tenant → audit → error_handling → rate_limiting    │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌──────────────────── FULL DDD ────────────────────────────┐  │
│  │                                                           │  │
│  │  AML & ONBOARDING                                        │  │
│  │  domain/ → application/ → infrastructure/ → api/          │  │
│  │  OnboardingCase, ScreeningCase, RiskScoringService         │  │
│  │  8-stage state machine, pluggable risk factors             │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────── SERVICE LAYER ───────────────────────┐  │
│  │                                                           │  │
│  │  CLIENT          COMPLIANCE        WORKFLOW               │  │
│  │  models.py       models.py         models.py              │  │
│  │  service.py      service.py        service.py             │  │
│  │  routes.py       routes.py         routes.py              │  │
│  │  schemas.py      schemas.py        schemas.py             │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────── SIMPLE CRUD ─────────────────────────┐  │
│  │                                                           │  │
│  │  IDENTITY    DOC_FACTORY   REGULATORY   LIGHT_TRACKING    │  │
│  │  models.py   models.py    models.py    models.py          │  │
│  │  routes.py   routes.py    routes.py    routes.py          │  │
│  │  schemas.py  generators/  schemas.py   schemas.py         │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────── SUPPORTING ──────────────────────────┐  │
│  │  notification/  agent/  audit/  dashboard/                │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Architecture levels:**
- **Full DDD** (AML only): Domain entities as Pydantic BaseModel, repository interfaces, command/query separation, domain events, mappers between domain and ORM
- **Service Layer** (Client, Compliance, Workflow): SQLAlchemy models are source of truth, service.py contains business logic (state machines, approval workflows), no separate domain entities
- **Simple CRUD** (Identity, DocFactory, Regulatory, LightTracking): Just models + schemas + routes. Zero business logic beyond validation
- **Supporting** (Notification, Agent, Audit, Dashboard): Infrastructure services, not bounded contexts

---

## Deployment Diagram

```
┌───────────────────── DEVELOPER MACHINE ─────────────────────┐
│                                                               │
│  docker compose up                                            │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                     │
│  │ PG 16   │  │ MailHog │  │ FastAPI │                      │
│  │ :5432   │  │ :8025   │  │ :8000   │                      │
│  └─────────┘  └─────────┘  └─────────┘                      │
│                              + Next.js dev server :3000       │
│  Auth bypass in dev (no Supabase needed locally)              │
└───────────────────────────────────────────────────────────────┘

┌───────────────────── PRODUCTION ────────────────────────────┐
│                                                               │
│  ┌─────────────────┐     ┌──────────────────────────────┐   │
│  │ Vercel / Static │     │ Supabase Cloud                │   │
│  │ (frontend)      │     │ ├─ PostgreSQL (RLS)           │   │
│  │                 │     │ ├─ Auth (JWT + MFA)           │   │
│  └────────┬────────┘     │ └─ Storage (documents)        │   │
│           │              └──────────────┬───────────────┘   │
│           │                             │                    │
│           └──────────┐   ┌──────────────┘                    │
│                      ▼   ▼                                   │
│           ┌─────────────────────┐                            │
│           │ Docker Container    │                            │
│           │ (Cloud Run/Fly/Rwy) │                            │
│           │ FastAPI backend     │                            │
│           └─────────────────────┘                            │
│                                                               │
│  External: Resend (email) + Anthropic (AI) + Cron (jobs)     │
│  Cost: $55-125/month                                         │
└───────────────────────────────────────────────────────────────┘
```

---

## Feature Implementation Map

### Phase 1 Features (47 P0 stories)

| Feature | Implementation | Complexity |
|---------|---------------|------------|
| **Auth + MFA** | Supabase Auth SDK, middleware validates JWT, RBAC in middleware | Simple |
| **Multi-tenancy** | `tenant_id` on every table, PostgreSQL RLS, middleware sets `app.current_tenant` | Simple (infra) |
| **Audit trail** | Middleware captures POST/PUT/PATCH/DELETE, writes to partitioned `audit_entries` | Simple (middleware) |
| **Client CRUD** | `client/models.py` + `service.py` for state machine + `routes.py` | Service Layer |
| **UBO management** | `crm_persons` + `crm_ubo_links` tables, service handles PEP→HIGH escalation | Service Layer |
| **Doc expiry alerts** | `pg_cron` daily job checks `id_doc_expiry`, creates tasks via scheduled endpoint | Simple (cron) |
| **Onboarding case** | Full DDD: `aml/domain/entities.py` OnboardingCase with 8-stage state machine | Full DDD |
| **Risk scoring** | `aml/domain/services.py` RiskScoringService with pluggable `RiskFactor` classes | Full DDD |
| **Screening hub** | `aml/` ScreeningCase aggregate, manual CSV upload Phase 1, provider abstraction for Phase 2 | Full DDD |
| **Hit management** | False positive workflow with rationale validation, sanctions require founder sign-off | Full DDD |
| **Due diligence** | Checklist engine in `aml/`, auto-generates required docs per client type + risk tier | Full DDD |
| **Approval routing** | `workflow/service.py` Task + Approval models, risk-tier based routing | Service Layer |
| **Daily re-screening** | Scheduled job endpoint, checks all active clients against screening records | Simple (cron) |
| **Policy library** | `compliance/models.py` Policy with version major.minor, `service.py` for approval workflow | Service Layer |
| **Registers (18 types)** | `compliance/models.py` Register + RegisterEntry, flexible JSONB data per type | Service Layer |
| **Incident management** | `compliance/service.py` 5-stage state machine, CRITICAL→notification invariant | Service Layer |
| **Training records** | RegisterEntry with type=TRAINING, no separate aggregate | Simple (register) |
| **Document Factory** | `document_factory/generators/` with python-docx + fpdf2, template merge fields | Simple + generators |
| **Founder dashboard** | `dashboard/queries.py` reads from `mv_founder_dashboard` materialized view | Simple (query) |
| **Team dashboard** | `dashboard/queries.py` filtered tasks/deadlines per user | Simple (query) |
| **Global search** | PostgreSQL `tsvector` full-text search across clients, cases, policies | Simple (SQL) |
| **Regulatory Radar** | `regulatory/models.py` RegulatoryChange, weekly scan logged, basic change log | Simple CRUD |
| **Auth pipeline (light)** | `light_tracking/models.py` LightAuthCase with stage tracking, Kanban UI | Simple CRUD |
| **Entity register (light)** | `light_tracking/models.py` LightEntity with key dates + traffic light | Simple CRUD |
| **Post-licence handoff** | `aml/application/sagas.py` PostLicenceHandoffSaga creates compliance infra | Full DDD (saga) |
| **Risk scoring agent** | `agent/risk_scoring_agent.py` calls Claude API, drafts score, human confirms | Agent |
| **Data migration** | Import scripts: Excel → client records, DOCX → templates | Scripts |
| **Reminders/escalation** | `workflow/service.py` + scheduled job checks due dates, sends emails | Service + cron |

### How Data Flows: Client Onboarding (Core Flow)

```
1. User creates client          → client/routes.py → client/service.py → DB
2. User starts onboarding       → aml/api/routes.py → CreateOnboardingCommand → Handler → DB
3. System auto-generates docs   → aml/ emits OnboardingCreated → doc_factory generates checklist
4. User uploads screening CSV   → aml/api/routes.py → ManualUploadProvider → ScreeningCase saved
5. AI drafts risk score         → agent/risk_scoring_agent.py → Claude API → draft saved
6. User confirms risk score     → aml/ SubmitRiskScore command → OnboardingCase advances
7. User completes DD checklist  → aml/ advance stage → FILE_COMPLETION
8. Approver approves            → aml/ Approve command → OnboardingApproved event
9. Client becomes ACTIVE        → client/service.py transitions status
10. Compliance registers created → PostLicenceHandoffSaga (if applicable)
```

---

## Technology Stack Summary

```
BACKEND                          FRONTEND
─────────                        ─────────
Python 3.12+                     Next.js 15
FastAPI                          React 19
Pydantic v2                      TypeScript 5.5
SQLAlchemy 2.0 (async)           Tailwind CSS 4
Alembic (migrations)             shadcn/ui
asyncpg (PostgreSQL driver)      TanStack Query v5
httpx (async HTTP)               React Hook Form + Zod
fpdf2 + python-docx              TanStack Table
Anthropic SDK (async)            Recharts
slowapi (rate limiting)          Vitest
Ruff + mypy (quality)            pnpm
uv (packages)

INFRASTRUCTURE                   SERVICES
──────────────                   ─────────
Docker Compose (local)           Supabase (DB + Auth + Storage)
GitHub Actions (CI/CD)           Resend (email)
Sentry (errors)                  Anthropic API (AI agents)
pg_cron (scheduled SQL)          Screening APIs (Phase 2)
```

---

## Async Architecture

```
Every request flows async end-to-end:

Browser → HTTPS → FastAPI (async def) → AsyncSession (asyncpg) → PostgreSQL
                                       → httpx.AsyncClient → External APIs
                                       → asyncio.to_thread() → CPU-bound work

No sync blocking anywhere. Connection pool manages DB connections.
asyncpg handles async PostgreSQL protocol natively.
httpx handles async HTTP for all external calls (Supabase, Resend, Claude, Screening).
CPU-bound work (PDF generation) offloaded to thread pool via asyncio.to_thread().
```

---

## What Each Developer Builds

```
SENIOR 1 (Track A): Foundation + Infrastructure
  Shared kernel, Identity, Audit middleware, Workflow engine,
  Document Factory, Dashboard views, Regulatory Radar,
  Scheduled jobs, Deployment, Data migration

SENIOR 2 (Track B): Domain Expert
  Client service layer, AML full DDD (core product),
  Screening workflow, Risk scoring, Compliance service layer,
  Light tracking, Post-licence handoff saga

JUNIOR (Track C): Frontend
  All 15 screens from wireframes, reusable components,
  Data tables, forms, dashboards, Kanban board

DEV 4 (Track D): Full-stack Flex + AML Co-owner
  AML API endpoints (co-own with Senior 2),
  Notification service, Email templates,
  Compliance API endpoints, Integration tests,
  Scheduled jobs, Deployment pipeline
```
