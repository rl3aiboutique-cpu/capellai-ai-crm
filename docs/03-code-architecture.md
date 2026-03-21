# Compliance Brain - Code Architecture

> Tech stack, project structure, patterns, and coding standards.
> Principles: DDD, SOLID, Dependency Injection, Open/Closed. Pragmatic — simple until complexity is justified.

---

## Tech Stack

### Backend

| Component | Technology | Justification |
|-----------|-----------|---------------|
| Language | **Python 3.12+ (tested up to 3.14)** | Modern typing, performance, ecosystem |
| Framework | **FastAPI 0.115+** | Async-first, Pydantic-native, OpenAPI auto-docs |
| Validation | **Pydantic v2** | Rust-core performance, strict validation |
| ORM | **SQLAlchemy 2.0+** | Mature, async, Unit of Work pattern |
| Migrations | **Alembic** | SQLAlchemy-native, reliable |
| Auth | **Supabase Auth** | Built-in MFA, JWT, user management, simple setup |
| Database | **Supabase (PostgreSQL 15+)** | Managed PostgreSQL, RLS built-in, auth integrated |
| Storage | **Supabase Storage** (abstractable) | File storage with RLS, swap to GCS/S3 later if needed |
| Background Tasks | **FastAPI BackgroundTasks** | Simple, no extra infrastructure for < 10 users |
| Email | **SMTP / Resend** | Simple email sending, no MS Graph complexity |
| AI | **Anthropic SDK (Claude)** | Agent layer, document analysis |
| Doc Generation | **python-docx / python-pptx / fpdf2** | DOCX, PPTX, PDF — pure Python, zero system deps |
| Rate Limiting | **slowapi** | Token bucket rate limiting for FastAPI |
| Testing | **pytest + pytest-asyncio** | Async test support, fixtures |
| Linting | **Ruff** | Fast all-in-one linter + formatter |
| Type Checking | **mypy (strict)** | Catch type errors before runtime |
| Package Manager | **uv** | Fast, deterministic, lockfile-based |

### Frontend

| Component | Technology | Justification |
|-----------|-----------|---------------|
| Framework | **Next.js 15** | App Router, file-based routing, Vercel deploy |
| UI Library | **React 19** | Latest concurrent features |
| Language | **TypeScript 5.5+** | Strict mode, type safety |
| Styling | **Tailwind CSS 4** | Utility-first, consistent with wireframes |
| Components | **shadcn/ui** | Accessible, customizable, not a dependency |
| State | **TanStack Query v5** | Server state management, caching, mutations |
| Forms | **React Hook Form + Zod** | Performant forms with schema validation |
| Tables | **TanStack Table** | Headless, sortable, filterable data tables |
| Charts | **Recharts** | KPI dashboards, heatmaps |
| Package Manager | **pnpm** | Fast, disk-efficient, strict |
| Linting | **ESLint (flat config) + Prettier** | Consistent code style |
| Testing | **Vitest + Testing Library** | Fast, Jest-compatible |

### Infrastructure

| Component | Technology | Justification |
|-----------|-----------|---------------|
| Database | **Supabase** (cloud or self-hosted) | Managed PostgreSQL + Auth + Storage in one |
| Local Dev | **Docker Compose** | PostgreSQL + MailHog locally |
| Cloud | **TBD** (GCP preferred, abstract for now) | Deploy when ready, Docker-first |
| CI/CD | **GitHub Actions** | Pipeline automation |
| Monitoring | **Structured logging + Sentry** | Error tracking, performance |
| Secrets | **Environment variables + .env** | Simple, 12-factor app |

---

## Project Structure

```
compliance-brain/
├── backend/
│   ├── src/
│   │   ├── shared/                          # Shared Kernel
│   │   │   ├── domain/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── base.py                  # AggregateRoot, ValueObject, DomainEvent (used by AML)
│   │   │   │   ├── state_machine.py          # StateMachine class (used by all contexts with states)
│   │   │   │   ├── value_objects.py          # TenantId, UserId, shared VOs
│   │   │   │   └── exceptions.py            # DomainException hierarchy
│   │   │   ├── infrastructure/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── database.py              # AsyncSession factory, async engine
│   │   │   │   ├── event_bus.py             # In-process event bus
│   │   │   │   ├── storage.py               # Supabase/abstract storage client (async httpx)
│   │   │   │   └── email.py                 # Resend/SMTP client (async)
│   │   │   └── api/
│   │   │       ├── __init__.py
│   │   │       ├── middleware.py            # Auth, tenant, audit, error handling
│   │   │       ├── dependencies.py          # get_db_session, get_current_user, etc.
│   │   │       └── errors.py               # Exception → HTTP status mapping
│   │   │
│   │   │   # ── FULL DDD (complex domain logic) ──────────────────
│   │   │
│   │   ├── aml/                             # AML & Onboarding (Full DDD)
│   │   │   ├── domain/
│   │   │   │   ├── entities.py              # OnboardingCase, ScreeningCase aggregates
│   │   │   │   ├── value_objects.py          # OnboardingStage, RiskScore, ScreeningHit
│   │   │   │   ├── events.py                # OnboardingApproved, TrueMatchConfirmed, etc.
│   │   │   │   ├── repositories.py          # OnboardingRepository, ScreeningRepository ABCs
│   │   │   │   ├── services.py              # RiskScoringService (pluggable factors)
│   │   │   │   └── exceptions.py
│   │   │   ├── application/
│   │   │   │   ├── commands.py              # CreateOnboarding, AdvanceStage, SubmitScore
│   │   │   │   ├── queries.py               # GetCase, ListCases, GetScreeningResults
│   │   │   │   ├── services.py              # OnboardingApplicationService
│   │   │   │   └── sagas.py                 # TrueMatchEscalationSaga, PepEscalationSaga
│   │   │   ├── infrastructure/
│   │   │   │   ├── models.py                # SQLAlchemy models
│   │   │   │   ├── repositories.py          # SqlOnboardingRepository
│   │   │   │   ├── mappers.py               # to_domain() / to_model() (only here!)
│   │   │   │   └── screening_provider.py   # ManualUploadProvider (Phase 1)
│   │   │   └── api/
│   │   │       ├── routes.py
│   │   │       ├── schemas.py
│   │   │       └── dependencies.py
│   │   │
│   │   │   # ── SERVICE LAYER (moderate logic) ───────────────────
│   │   │
│   │   ├── client/                          # Client Management (Service Layer)
│   │   │   ├── models.py                    # SQLAlchemy: Client, Person, ClientDocument
│   │   │   ├── schemas.py                   # Pydantic request/response
│   │   │   ├── service.py                   # State machine, UBO linking, PEP escalation
│   │   │   ├── routes.py                    # FastAPI endpoints
│   │   │   └── exceptions.py
│   │   │
│   │   ├── compliance/                      # Compliance Framework (Service Layer)
│   │   │   ├── models.py                    # Policy, Register, RegisterEntry, Incident
│   │   │   ├── schemas.py
│   │   │   ├── service.py                   # Policy approval, incident escalation, register mgmt
│   │   │   ├── routes.py
│   │   │   └── exceptions.py
│   │   │
│   │   ├── workflow/                        # Workflow Engine (Service Layer)
│   │   │   ├── models.py                    # Task, Approval
│   │   │   ├── schemas.py
│   │   │   ├── service.py                   # Task assignment, approval logic, SLA, reminders
│   │   │   ├── routes.py
│   │   │   └── exceptions.py
│   │   │
│   │   │   # ── SIMPLE CRUD (no business logic) ─────────────────
│   │   │
│   │   ├── identity/                        # Identity & Access (Simple CRUD)
│   │   │   ├── models.py                    # User, Tenant, UserTenantAssignment
│   │   │   ├── schemas.py
│   │   │   ├── routes.py
│   │   │   └── auth.py                      # Supabase Auth integration
│   │   │
│   │   ├── document_factory/                # Document Factory (Simple CRUD + generators)
│   │   │   ├── models.py                    # DocumentTemplate, GeneratedDocument
│   │   │   ├── schemas.py
│   │   │   ├── routes.py
│   │   │   └── generators/                 # docx_generator.py, pdf_generator.py
│   │   │
│   │   ├── regulatory/                      # Regulatory Intelligence (Simple CRUD Phase 1)
│   │   │   ├── models.py                    # RegulatoryChange
│   │   │   ├── schemas.py
│   │   │   └── routes.py
│   │   │
│   │   ├── light_tracking/                  # Light CRUD: auth cases + entities
│   │   │   ├── models.py                    # LightAuthCase, LightEntity
│   │   │   ├── schemas.py
│   │   │   └── routes.py
│   │   │
│   │   │   # ── SUPPORTING SERVICES ─────────────────────────────
│   │   │
│   │   ├── notification/                    # Email service (no models)
│   │   │   ├── service.py                   # send_email, send_reminder
│   │   │   └── templates/                   # HTML email templates
│   │   │
│   │   ├── agent/                           # AI Agent Layer
│   │   │   ├── base.py                      # AgentAction, AgentResult models
│   │   │   ├── risk_scoring_agent.py        # Drafts risk scores
│   │   │   └── claude_client.py             # Anthropic AsyncClient wrapper
│   │   │
│   │   ├── audit/                           # Audit Trail (middleware-driven)
│   │   │   ├── models.py                    # AuditEntry (append-only)
│   │   │   ├── routes.py                    # List + export endpoints
│   │   │   └── schemas.py
│   │   │
│   │   ├── dashboard/                       # Dashboard aggregations
│   │   │   ├── routes.py                    # Founder, Team dashboard endpoints
│   │   │   └── queries.py                   # Materialized view queries
│   │   │
│   │   └── main.py                          # FastAPI app factory
│   │
│   ├── tests/
│   │   ├── unit/                            # Domain tests (AML) + service tests (client, compliance)
│   │   ├── integration/                     # API endpoint tests with test DB
│   │   ├── conftest.py                      # Fixtures, test DB, factories
│   │   └── factories.py                     # Synthetic test data
│   │
│   ├── migrations/                          # Alembic
│   ├── pyproject.toml
│   ├── alembic.ini
│   └── Makefile
│
├── frontend/
│   ├── src/
│   │   ├── app/                             # Next.js App Router
│   │   │   ├── (auth)/                     # Auth group: login, mfa
│   │   │   ├── (dashboard)/                # Main app group
│   │   │   │   ├── page.tsx                # Founder dashboard
│   │   │   │   ├── clients/
│   │   │   │   │   ├── page.tsx            # Client list
│   │   │   │   │   └── [id]/
│   │   │   │   │       └── page.tsx        # Client detail
│   │   │   │   ├── onboarding/
│   │   │   │   ├── screening/
│   │   │   │   ├── compliance/
│   │   │   │   │   ├── policies/
│   │   │   │   │   ├── registers/
│   │   │   │   │   ├── incidents/
│   │   │   │   │   └── board-packs/
│   │   │   │   ├── regulatory/
│   │   │   │   ├── authorisations/
│   │   │   │   ├── entities/
│   │   │   │   ├── cmp/
│   │   │   │   ├── documents/
│   │   │   │   ├── audit-trail/
│   │   │   │   └── settings/
│   │   │   ├── layout.tsx
│   │   │   └── globals.css
│   │   │
│   │   ├── components/
│   │   │   ├── ui/                         # shadcn/ui components (Button, Dialog, etc.)
│   │   │   ├── layout/                     # Sidebar, TopBar, Breadcrumb
│   │   │   ├── data-table/                 # Reusable table with sort/filter/pagination
│   │   │   ├── forms/                      # Form components
│   │   │   └── dashboard/                  # KPI cards, charts, heatmaps
│   │   │
│   │   ├── features/                       # Feature-specific components
│   │   │   ├── clients/
│   │   │   ├── onboarding/
│   │   │   ├── screening/
│   │   │   ├── compliance/
│   │   │   ├── entities/
│   │   │   └── ...
│   │   │
│   │   ├── lib/
│   │   │   ├── api-client.ts              # Typed API client (fetch wrapper)
│   │   │   ├── auth.ts                    # Auth context and hooks
│   │   │   ├── tenant.ts                  # Tenant context
│   │   │   └── utils.ts
│   │   │
│   │   └── types/
│   │       ├── api.ts                     # Generated from OpenAPI schema
│   │       └── index.ts
│   │
│   ├── public/
│   ├── package.json
│   ├── tsconfig.json
│   ├── tailwind.config.ts
│   ├── next.config.ts
│   └── vitest.config.ts
│
├── infrastructure/
│   └── docker/
│       ├── Dockerfile
│       └── docker-compose.yml             # Local dev: PostgreSQL + MailHog
│
├── .github/
│   └── workflows/
│       ├── ci.yml                         # Lint + test + build on PR
│       ├── deploy-staging.yml             # Deploy to staging on merge to develop
│       └── deploy-prod.yml               # Deploy to prod on merge to main
│
├── .gitignore
├── .gitattributes
├── .editorconfig
├── .pre-commit-config.yaml
├── .env.example
├── CLAUDE.md
└── docs/                                   # This documentation
```

> **No BFF proxy**: Frontend calls FastAPI backend directly. CORS configured on backend. JWT from Supabase Auth sent in Authorization header.

---

## Architectural Patterns

### 1. Architecture Levels

The project uses three levels of architecture based on domain complexity:

```
FULL DDD (aml/ only):
┌─────────────────┐
│      API         │  ← FastAPI routes, Pydantic schemas
├─────────────────┤
│   Application    │  ← Commands, queries, sagas
├─────────────────┤
│     Domain       │  ← Entities (Pydantic BaseModel), VOs, events, repo interfaces
├─────────────────┤
│  Infrastructure  │  ← SQLAlchemy models, repo implementations, mappers
└─────────────────┘

SERVICE LAYER (client/, compliance/, workflow/):
┌─────────────────┐
│    routes.py     │  ← FastAPI endpoints
├─────────────────┤
│   service.py     │  ← Business logic, state machines, workflows
├─────────────────┤
│   models.py      │  ← SQLAlchemy models (source of truth)
├─────────────────┤
│   schemas.py     │  ← Pydantic validation
└─────────────────┘

SIMPLE CRUD (identity/, document_factory/, regulatory/, light_tracking/):
┌─────────────────┐
│    routes.py     │  ← FastAPI endpoints (thin)
├─────────────────┤
│   models.py      │  ← SQLAlchemy models
├─────────────────┤
│   schemas.py     │  ← Pydantic validation
└─────────────────┘
```

**Dependency Rule**: In Full DDD, dependencies point inward (domain has zero deps). In Service Layer and CRUD, models.py is the source of truth — no separate domain entities.

**Promotion path**: CRUD → Service Layer → Full DDD. Start simple, add layers only when complexity demands it.

> **One model system**: Pydantic BaseModel is used for everything — domain entities (in Full DDD AML context), value objects, commands, API schemas, and SQLAlchemy model validation. No dataclasses. This keeps the stack simple and consistent.

### 2. Dependency Injection

```python
# Simple DI with FastAPI Depends() — no external library needed

# src/client/api/dependencies.py
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

async def get_client_repository(
    session: AsyncSession = Depends(get_db_session),
) -> ClientRepository:
    return SqlClientRepository(session)

async def get_client_service(
    repo: ClientRepository = Depends(get_client_repository),
    event_bus: EventBus = Depends(get_event_bus),
) -> ClientApplicationService:
    return ClientApplicationService(repo=repo, event_bus=event_bus)

# Usage in routes
@router.post("/clients")
async def create_client(
    cmd: CreateClientRequest,
    service: ClientApplicationService = Depends(get_client_service),
    current_user: User = Depends(get_current_user),
):
    return await service.create(cmd, created_by=current_user.id)
```

> **Unit of Work**: SQLAlchemy's `AsyncSession` already implements the Unit of Work pattern (flush, commit, rollback). No custom wrapper needed. The `get_db_session` dependency manages the session lifecycle per request.

### 3. Repository Pattern

> **Note**: Repository pattern is used ONLY in the AML context (Full DDD). Service Layer contexts use `AsyncSession` directly in `service.py`. CRUD contexts use `AsyncSession` directly in `routes.py`.

```python
# src/client/domain/repositories.py (Interface)
from abc import ABC, abstractmethod

class ClientRepository(ABC):
    @abstractmethod
    async def get_by_id(self, tenant_id: TenantId, client_id: ClientId) -> Client | None: ...

    @abstractmethod
    async def save(self, client: Client) -> None: ...

    @abstractmethod
    async def search(self, tenant_id: TenantId, filters: ClientFilters) -> list[Client]: ...

# src/client/infrastructure/repositories.py (Implementation)
class SqlClientRepository(ClientRepository):
    def __init__(self, session: AsyncSession):
        self._session = session

    async def get_by_id(self, tenant_id: TenantId, client_id: ClientId) -> Client | None:
        stmt = select(ClientModel).where(
            ClientModel.tenant_id == tenant_id,
            ClientModel.id == client_id,
        )
        result = await self._session.execute(stmt)
        row = result.scalar_one_or_none()
        return to_domain(row) if row else None

    async def save(self, client: Client) -> None:
        model = to_model(client)
        await self._session.merge(model)
```

> **Keep it simple**: Domain entities are Pydantic BaseModel. ORM models are SQLAlchemy. Mappers are simple functions (not classes) — only create mapper files where the domain shape differs from the DB shape. For simple entities, the mapper is 2-3 lines.

### 4. Command/Query Separation (lightweight CQRS)

> **Note**: Command/Query separation is used ONLY in the AML context. Service Layer contexts use simple service methods. CRUD contexts handle everything in route handlers.

```python
# src/aml/application/commands.py
class CreateOnboardingCommand(BaseModel):
    model_config = ConfigDict(frozen=True)

    tenant_id: TenantId
    client_id: ClientId
    client_type: ClientType
    created_by: UserId

# src/client/service.py (Service Layer — no separate query class needed)
async def search_clients(
    session: AsyncSession,
    tenant_id: TenantId,
    filters: ClientFilters,  # Pydantic model
    page: int = 1,
    page_size: int = 20,
) -> PaginatedResult[ClientSummaryDTO]:
    # Direct SQL for read performance
    ...
```

### 5. Multi-Tenancy (Row-Level Security)

```python
# Every query automatically scoped by tenant_id

# src/shared/infrastructure/middleware.py
class TenantMiddleware:
    """Extracts tenant_id from JWT token and sets it in request state."""

    async def __call__(self, request: Request, call_next):
        token = extract_token(request)
        request.state.tenant_id = token.tenant_id
        response = await call_next(request)
        return response

# All repository methods receive tenant_id as first parameter
# PostgreSQL RLS as defense-in-depth:
# CREATE POLICY tenant_isolation ON clients
#   USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

### 6. Audit Trail (Cross-Cutting via Middleware)

```python
# src/shared/api/middleware.py
class AuditMiddleware:
    """Captures every state-changing request as an audit entry."""

    async def __call__(self, request: Request, call_next):
        response = await call_next(request)
        if request.method in ("POST", "PUT", "PATCH", "DELETE"):
            await self._record_audit(request, response)
        return response
```

### 7. Domain Event Bus (In-Process, Phase 1)

```python
# src/shared/infrastructure/event_bus.py
class InProcessEventBus(EventBus):
    """Synchronous in-process event bus. Replace with message queue if needed in Phase 2."""

    def __init__(self):
        self._handlers: dict[type[DomainEvent], list[Callable]] = {}

    def subscribe(self, event_type: type[DomainEvent], handler: Callable) -> None:
        self._handlers.setdefault(event_type, []).append(handler)

    async def publish(self, events: list[DomainEvent]) -> None:
        for event in events:
            for handler in self._handlers.get(type(event), []):
                await handler(event)
```

### 8. Error Handling Strategy

```python
# Domain exceptions map to HTTP status codes via middleware

EXCEPTION_STATUS_MAP = {
    UnauthorizedActionError: 403,
    InvalidStateTransitionError: 409,
    InvariantViolationError: 422,
    EntityNotFoundError: 404,
    IdempotencyConflictError: 409,
    ConcurrencyConflictError: 409,  # Optimistic lock failure
}

# Error response format (consistent across all endpoints)
{
    "error": {
        "code": "INVALID_STATE_TRANSITION",
        "message": "Cannot transition client from EXITED to ACTIVE. Use reactivation.",
        "details": { "current_state": "EXITED", "requested_state": "ACTIVE" },
        "correlation_id": "uuid-here"
    }
}

# Errors are logged to structured logs AND an error_log table for dashboard display
```

### 9. Rate Limiting

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@router.post("/auth/login")
@limiter.limit("5/minute")  # Anti brute-force
async def login(request: Request, ...): ...

@router.post("/api/v1/clients")
@limiter.limit("60/minute")  # Normal API rate
async def create_client(request: Request, ...): ...
```

### 10. Open/Closed Principle in Practice

> **Note**: The pluggable factor pattern is part of the AML Full DDD context. Other contexts use simpler approaches.

```python
# Example: Risk Scoring is open for extension via pluggable risk factors

class RiskFactor(ABC):
    @abstractmethod
    def name(self) -> str: ...

    @abstractmethod
    def calculate(self, client: Client, context: ScoringContext) -> FactorScore: ...

class JurisdictionRiskFactor(RiskFactor):
    def name(self) -> str: return "jurisdiction_risk"
    def calculate(self, client, context) -> FactorScore: ...

class PepRiskFactor(RiskFactor):
    def name(self) -> str: return "pep_status"
    def calculate(self, client, context) -> FactorScore: ...

class RiskScoringService:
    """Closed for modification, open for extension via new RiskFactor implementations."""

    def __init__(self, factors: list[RiskFactor]):
        self._factors = factors

    def calculate(self, client: Client, context: ScoringContext) -> RiskScore:
        factor_scores = [f.calculate(client, context) for f in self._factors]
        total = sum(fs.weighted_score for fs in factor_scores)
        tier = self._determine_tier(total, context.thresholds)
        return RiskScore(total_score=total, tier=tier, factors=factor_scores)
```

### 11. Async/Await Rules

**Everything is async. No exceptions.**

| Component | Async approach |
|-----------|---------------|
| All FastAPI endpoints | `async def` |
| All DB operations | `AsyncSession` via `asyncpg` |
| All HTTP calls (APIs) | `httpx.AsyncClient` (never `requests`) |
| Supabase Auth/Storage | `httpx.AsyncClient` direct calls |
| Email sending | `httpx.AsyncClient` to Resend API |
| AI (Claude) | `AsyncAnthropic` client |
| Doc generation (CPU) | `await asyncio.to_thread(generate_pdf, ...)` |
| File I/O | `await asyncio.to_thread(...)` for large files |

**Banned**: `requests`, `urllib`, any sync HTTP client, any sync DB driver, `time.sleep()`.

```python
# WRONG - blocks the event loop
import requests
response = requests.post("https://api.resend.com/emails", ...)

# RIGHT - async all the way
async with httpx.AsyncClient() as client:
    response = await client.post("https://api.resend.com/emails", ...)

# WRONG - sync file generation blocks
pdf_bytes = generate_pdf(data)  # CPU-bound, takes 2 seconds

# RIGHT - offload to thread pool
pdf_bytes = await asyncio.to_thread(generate_pdf, data)
```

### 12. Screening Provider Abstraction

Designed for Phase 1 manual upload → Phase 2 API integration:

```python
class ScreeningProvider(Protocol):
    """Interface for screening providers. Swap implementations without changing domain."""
    async def screen(self, person: PersonScreeningData) -> list[RawScreeningHit]: ...
    async def bulk_screen(self, persons: list[PersonScreeningData]) -> list[ScreeningResult]: ...

class ManualUploadProvider:
    """Phase 1: User uploads CSV/PDF with screening results."""
    async def screen(self, person):
        raise NotImplementedError("Use upload endpoint instead")

class WorldCheckProvider:
    """Phase 2: LSEG World-Check API integration."""
    def __init__(self, client: httpx.AsyncClient, api_key: str):
        self._client = client
        self._api_key = api_key

    async def screen(self, person: PersonScreeningData) -> list[RawScreeningHit]:
        response = await self._client.post(
            "https://api.worldcheck.com/v2/screen",
            json=person.to_api_format(),
            headers={"Authorization": f"Bearer {self._api_key}"}
        )
        return [RawScreeningHit.from_worldcheck(hit) for hit in response.json()["hits"]]
```

> **Risk Scoring**: Factors and weights are configurable per tenant. Default configuration works with manual input. Full algorithmic scoring activates when CRA matrix is imported from the client's Excel template.

---

## Coding Standards

### Python

- **Type hints everywhere** — use `from __future__ import annotations`
- **Pydantic v2 models** for all DTOs and API schemas
- **Pydantic BaseModel** for everything: domain entities, value objects, commands, DTOs, API schemas. One model system, not two.
- **Protocol classes** for interfaces (or ABC where state is needed)
- **`async/await`** throughout — no sync blocking in async context
- **No `Any` types** unless absolutely unavoidable
- **Ruff** for linting (replaces flake8, isort, pyupgrade)
- **Ruff format** for formatting (replaces black)
- **mypy strict** for type checking
- **90%+ test coverage** for domain layer (entities, VOs, services, invariants)

### TypeScript

- **Strict mode** in tsconfig.json
- **No `any`** — use `unknown` + type guards
- **Zod schemas** for runtime validation (shared with forms)
- **All pages use `'use client'`**. No SSR needed for internal dashboard tool.
- **TanStack Query** for all API calls (no raw fetch in components)
- **Absolute imports** (`@/components/...`, `@/lib/...`)

### Git

- **Conventional commits**: `feat:`, `fix:`, `refactor:`, `docs:`, `test:`, `chore:`
- **Branch naming**: `feat/US-3.1-onboarding-lifecycle`, `fix/screening-hit-validation`
- **PR size**: max ~400 lines changed (split larger changes)
- **Pre-commit hooks**: ruff, mypy, gitleaks, trailing whitespace, LF enforcement

### API

- **RESTful endpoints** per bounded context: `/api/v1/{context}/{resource}`
- **Consistent error responses**: `{ "error": { "code": "...", "message": "...", "details": [...] } }`
- **Pagination**: `?page=1&page_size=20` → `{ "items": [...], "total": N, "page": 1, "pages": M }`
- **Filtering**: `?status=ACTIVE&risk_tier=HIGH`
- **OpenAPI auto-generated** from Pydantic schemas

### API Route Structure
```
/api/v1/auth/...                    # Identity (auth endpoints)
/api/v1/tenants/...                 # Identity (tenant sub-resource)
/api/v1/clients/...                 # Client Management
/api/v1/persons/...                 # Person/UBO Management
/api/v1/onboarding/...              # AML Onboarding
/api/v1/screening/...               # Screening
/api/v1/compliance/policies/...     # Compliance - Policies
/api/v1/compliance/registers/...    # Compliance - Registers
/api/v1/compliance/incidents/...    # Compliance - Incidents
/api/v1/regulatory/changes/...      # Regulatory Intelligence
/api/v1/regulatory/alerts/...       # Client Alerts
/api/v1/tracking/authorisations/... # Light Tracking: Authorisation Cases (Phase 1)
/api/v1/tracking/entities/...       # Light Tracking: Entities (Phase 1)
/api/v1/documents/templates/...     # Document Factory
/api/v1/documents/generated/...     # Generated Documents
/api/v1/workflow/tasks/...          # Workflow Tasks
/api/v1/audit/...                   # Audit Trail
/api/v1/dashboard/...               # Dashboard aggregations
```

### API Versioning

All endpoints prefixed with `/api/v1/`. Versioning strategy:
- **URL-based**: `/api/v1/`, `/api/v2/`
- **Breaking changes** get a new version; additive changes stay in current version
- **OpenAPI spec** auto-generated and published per version
- **Deprecation**: old versions supported for 6 months after new version release

---

## Quality Infrastructure

### Pre-Commit Hooks (ordered fast-to-slow)
1. Line endings (LF enforcement)
2. Trailing whitespace removal
3. gitleaks (secrets scanning)
4. ruff check + ruff format (Python lint + format)
5. ESLint + Prettier (TypeScript lint + format)
6. mypy (Python type checking)

### CI Pipeline (GitHub Actions)
```yaml
# .github/workflows/ci.yml
on: [pull_request]
jobs:
  backend:
    - ruff check
    - ruff format --check
    - mypy --strict
    - pytest tests/unit -x --cov=src --cov-fail-under=80
    - pytest tests/integration (with test DB)
    - alembic upgrade head (migration check)

  frontend:
    - eslint
    - prettier --check
    - tsc --noEmit
    - vitest run --coverage
    - next build
```

### Testing Strategy
- **Unit tests**: Pure domain logic (no DB, no framework). Test aggregates, value objects, domain services, invariants.
- **Integration tests**: API endpoints with test database. Test full request → response cycle.
- **Fixtures**: Synthetic data factories (never production data). Use `conftest.py` with factory functions.
- **Coverage target**: 90%+ domain layer (entities, VOs, services, invariants), 80%+ application layer, 60%+ infrastructure.

---

## Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Monorepo vs polyrepo | **Monorepo** | Simplifies shared types, CI, and deployment for a small team (4 devs) |
| Modular monolith vs microservices | **Modular monolith** | Bounded contexts as Python packages, not separate services. Extract later if needed. |
| ORM vs raw SQL | **SQLAlchemy 2.0** | Domain mapping, Unit of Work, mature ecosystem. Raw SQL for complex reads. |
| Frontend rendering | **Client-side rendering (`'use client'`)** | Internal tool, < 10 users, no SEO. Supabase Auth works client-side. Eliminates SSR/hydration complexity. |
| State management | **TanStack Query** | Server state only — no global client state needed |
| Auth | **Supabase Auth** | Simple setup, built-in MFA, JWT, works with PostgreSQL RLS |
| Event bus | **In-process (Phase 1)** | Simplicity. Message queue if needed later. |
| AI integration | **Claude API** | Superior reasoning for compliance domain, tool use for agents |
| Document generation | **python-docx/pptx + fpdf2** | Pure Python, zero system dependencies |
| Background jobs | **FastAPI BackgroundTasks** | Simple, no extra infra. Upgrade to task queue only if needed |
