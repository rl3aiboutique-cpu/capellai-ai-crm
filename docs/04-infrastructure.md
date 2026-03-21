# Compliance Brain - Infrastructure Architecture

> Simple, Docker-first infrastructure. Cloud-agnostic with Supabase as primary managed service.
> Principle: Start simple, scale when needed. < 10 concurrent users initially.

---

## Architecture Overview

```
┌────────────────────────────────────────────────────────────────────┐
│                    PRODUCTION ENVIRONMENT                          │
│                                                                    │
│  ┌─────────────────────────┐    ┌─────────────────────────────┐   │
│  │  Frontend (Next.js)      │    │  Supabase Cloud              │   │
│  │  Vercel / Cloud Run /    │    │  ┌───────────────────────┐  │   │
│  │  Static hosting          │    │  │  PostgreSQL 15+       │  │   │
│  └──────────┬──────────────┘    │  │  (RLS enabled)        │  │   │
│             │ HTTPS              │  └───────────────────────┘  │   │
│  ┌──────────▼──────────────┐    │  ┌───────────────────────┐  │   │
│  │  Backend (FastAPI)       │    │  │  Supabase Auth        │  │   │
│  │  Docker container        │    │  │  (JWT + MFA)          │  │   │
│  │  Cloud Run / Fly.io /    │    │  └───────────────────────┘  │   │
│  │  Railway                 │    │  ┌───────────────────────┐  │   │
│  └──────────┬──────────────┘    │  │  Supabase Storage     │  │   │
│             │                    │  │  (documents, evidence) │  │   │
│             └────────────────────│  └───────────────────────┘  │   │
│                                  └─────────────────────────────┘   │
│                                                                    │
│  External Services:                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐    │
│  │  SMTP/Resend  │  │ Anthropic API │  │ Screening APIs (P2)  │    │
│  │  (email)      │  │ (AI agents)   │  │ World-Check etc.     │    │
│  └──────────────┘  └──────────────┘  └──────────────────────┘    │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │  Monitoring: Sentry (errors) + Structured JSON logs          │ │
│  └──────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────┘
```

---

## Local Development (Docker Compose)

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: compliance_brain
      POSTGRES_USER: cbrain
      POSTGRES_PASSWORD: dev_password
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data

  mailhog:
    image: mailhog/mailhog
    ports:
      - "1025:1025"   # SMTP
      - "8025:8025"   # Web UI

  api:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      DATABASE_URL: postgresql+asyncpg://cbrain:dev_password@db:5432/compliance_brain
      # SUPABASE_URL: not needed locally (auth bypassed in dev)
      # SUPABASE_ANON_KEY: not needed locally (auth bypassed in dev)
      SMTP_HOST: mailhog
      SMTP_PORT: 1025
      APP_ENV: development
    ports: ["8000:8000"]
    depends_on: [db]
    volumes:
      - ./backend/src:/app/src  # Hot reload

volumes:
  pgdata:
```

All developers run `docker compose up` and have a working environment in minutes.

### Auth in Development

In local dev, Supabase Auth is NOT required. A dev middleware bypasses authentication and injects a fixed test user:

```python
# Only active when APP_ENV=development
class DevAuthBypass:
    """Skips Supabase Auth in local dev. Injects a test user."""
    async def __call__(self, request: Request, call_next):
        request.state.user_id = DEV_USER_ID
        request.state.tenant_id = DEV_TENANT_ID
        request.state.role = "MASTER_ADMIN"
        return await call_next(request)
```

In CI and staging/production, Supabase Auth is used with real JWT validation.

---

## Production Stack

| Component | Service | Notes |
|-----------|---------|-------|
| Database | **Supabase Cloud** (PostgreSQL) | Managed, RLS built-in, backups included |
| Auth | **Supabase Auth** | JWT + MFA, user management dashboard |
| File Storage | **Supabase Storage** | RLS-protected, abstractable to GCS/S3 later |
| Backend | **Docker container** on Cloud Run / Fly.io / Railway | Auto-scale, pay-per-use |
| Frontend | **Vercel** or static hosting | Next.js optimized deployment |
| Email | **Resend** or SMTP | Transactional emails, simple API |
| AI | **Anthropic API** | Claude for agent layer |
| Monitoring | **Sentry** | Error tracking + performance |
| CI/CD | **GitHub Actions** | Lint → Test → Build → Deploy |
| Secrets | **Environment variables** | 12-factor app, platform-managed secrets |

### Estimated Costs (Phase 1)

| Service | Tier | Est. Monthly |
|---------|------|-------------|
| Supabase | Pro ($25/mo) | $25 |
| Backend hosting (Cloud Run/Fly) | Pay-per-use | $10-30 |
| Frontend hosting (Vercel) | Free/Pro | $0-20 |
| Resend (email) | Free tier (3K/mo) | $0 |
| Anthropic API | Pay-per-use | $20-50 |
| Sentry | Free tier | $0 |
| GitHub Actions | Free tier (2000 min/mo) | $0 |
| **Total** | | **$55-125/month** |

---

## Database Architecture

### Multi-Tenancy (Row-Level Security)

Every table has `tenant_id`. RLS enforced at PostgreSQL level:

```sql
ALTER TABLE crm_clients ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON crm_clients
    USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

Application sets tenant context per request:
```python
# Middleware sets tenant context on every request
await session.execute(text(f"SET app.current_tenant = '{tenant_id}'"))
```

### Backup & Recovery

| Strategy | Implementation |
|----------|---------------|
| Automated backups | Supabase daily backups (Pro: 7-day PITR) |
| Long-term retention | Weekly pg_dump to cloud storage (cold tier), 6-year retention |
| Recovery RTO | < 1 hour (Supabase restore) |
| Recovery RPO | < 24 hours (daily backup) |
| Backup testing | Monthly restore test to staging |

### Audit Trail (Append-Only)

```sql
CREATE TABLE audit_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    actor_type VARCHAR(20) NOT NULL,  -- USER, AGENT, SYSTEM
    actor_id VARCHAR(255) NOT NULL,
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    action VARCHAR(255) NOT NULL,
    resource_type VARCHAR(100) NOT NULL,
    resource_id UUID,
    before_snapshot JSONB,
    after_snapshot JSONB,
    source VARCHAR(20) NOT NULL,
    correlation_id UUID NOT NULL,
    metadata JSONB
) PARTITION BY RANGE (timestamp);

-- No UPDATE/DELETE allowed
REVOKE UPDATE, DELETE ON audit_entries FROM app_user;
```

Partitioned monthly. Use `pg_partman` for automatic partition creation and cleanup after 6 years.

---

## Security

| Layer | Implementation |
|-------|---------------|
| Transport | HTTPS everywhere (platform-managed TLS) |
| Authentication | Supabase Auth with MFA (TOTP) |
| Authorization | RBAC middleware + PostgreSQL RLS |
| Data isolation | RLS at DB level + tenant_id in application |
| Secrets | Environment variables, platform-managed |
| Audit | Immutable append-only audit trail |
| Rate limiting | slowapi on FastAPI (5/min auth, 60/min API) |
| CORS | Configured origins only. Dev: `localhost:3000`. Prod: frontend domain |
| Input validation | Pydantic v2 strict mode on all endpoints |
| Dependency scanning | Dependabot + pip-audit in CI |
| Secret scanning | gitleaks in pre-commit + CI |

### GDPR / Data Privacy

| Requirement | Implementation |
|-------------|---------------|
| Data minimisation | Only required fields per context |
| DSAR (right to access) | Export endpoint per person/client |
| Right to deletion | Soft delete + anonymization (subject to 6-year retention) |
| Data residency | Supabase region selection (configure per compliance need) |
| Retention | 6 years default, configurable per category |

---

## File Upload Policy

| Rule | Value |
|------|-------|
| Max file size | **50 MB** |
| Allowed types | PDF, DOCX, XLSX, PPTX, PNG, JPG, JPEG |
| Validation | File extension + Content-Type header check |
| Storage | Supabase Storage with tenant-isolated buckets |
| Naming | `{tenant_id}/{context}/{entity_id}/{uuid}.{ext}` |
| Virus scanning | Not in Phase 1. Add ClamAV integration in Phase 2 if needed |

```python
ALLOWED_EXTENSIONS = {".pdf", ".docx", ".xlsx", ".pptx", ".png", ".jpg", ".jpeg"}
MAX_FILE_SIZE = 50 * 1024 * 1024  # 50 MB

async def validate_upload(file: UploadFile) -> None:
    ext = Path(file.filename).suffix.lower()
    if ext not in ALLOWED_EXTENSIONS:
        raise InvalidFileTypeError(f"File type {ext} not allowed")
    if file.size and file.size > MAX_FILE_SIZE:
        raise FileTooLargeError(f"File exceeds {MAX_FILE_SIZE // 1024 // 1024} MB limit")
```

---

## Timezone Policy

- **Database**: All timestamps stored as `TIMESTAMPTZ` in **UTC**
- **API**: All timestamps sent/received in **ISO 8601 UTC** (e.g., `2026-03-21T10:00:00Z`)
- **Frontend**: Displays in user's configured timezone (toggle in settings, default: UTC+4 Dubai)
- **Business days**: Calculated using tenant's configured calendar (UAE default: Sun-Thu, excluding UAE public holidays)
- **SLA calculations**: Based on business days, not calendar days

---

## CI/CD Pipeline

```
Developer → Push PR → GitHub Actions CI → Lint + Test + Build
                           ↓ (pass)
                      PR Review + Approve
                           ↓
                      Merge to main → Deploy (auto or manual approve)
```

### CI Workflow

```yaml
name: CI
on: [pull_request]

jobs:
  backend:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: test_db
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: cd backend && uv sync
      - run: cd backend && uv run ruff check src/
      - run: cd backend && uv run ruff format --check src/
      - run: cd backend && uv run mypy src/ --strict
      - run: cd backend && uv run pytest tests/unit -x --cov=src --cov-fail-under=80
      - run: cd backend && uv run alembic upgrade head
      - run: cd backend && uv run pytest tests/integration -x

  frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22' }
      - run: cd frontend && pnpm install --frozen-lockfile
      - run: cd frontend && pnpm lint
      - run: cd frontend && pnpm typecheck
      - run: cd frontend && pnpm test -- --coverage
      - run: cd frontend && pnpm build
```

---

## Dockerfile (Backend)

```dockerfile
# Use latest stable Python. Compatible with 3.12-3.14
FROM python:3.13-slim

RUN groupadd -r app && useradd -r -g app app
WORKDIR /app

COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

COPY backend/pyproject.toml backend/uv.lock ./
RUN uv sync --frozen --no-dev

COPY backend/src/ ./src/
COPY backend/migrations/ ./migrations/
COPY backend/alembic.ini .

USER app
EXPOSE 8000

CMD ["uv", "run", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## Monitoring & Observability

| What | How |
|------|-----|
| Errors | Sentry (auto-capture exceptions, stack traces) |
| Logs | Structured JSON logs with correlation_id |
| Health | `GET /health` → 200 OK with DB/storage status |
| Uptime | Platform monitoring (Cloud Run / Fly.io built-in) |
| AI agents | `agent_actions` table with metrics (see DDD doc) |

### Health Check

```python
@router.get("/health")
async def health(db: AsyncSession = Depends(get_db_session)):
    db_ok = await check_db(db)
    return {"status": "healthy" if db_ok else "degraded", "database": db_ok}
```

---

## Scheduled Jobs

No Celery, no Redis, no worker processes. Two simple mechanisms:

### SQL Jobs (pg_cron)

For pure SQL operations, use `pg_cron` extension (available in Supabase):

```sql
-- Refresh dashboard materialized view every 5 minutes
SELECT cron.schedule('refresh-dashboard', '*/5 * * * *',
    'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_founder_dashboard');

-- Clean up expired idempotency keys daily
SELECT cron.schedule('cleanup-idempotency', '0 2 * * *',
    'DELETE FROM idempotency_keys WHERE expires_at < NOW()');
```

### Business Logic Jobs (HTTP endpoint + external cron)

For jobs that need Python logic (sending emails, checking business rules):

```python
# Internal endpoint, protected by API key
@router.post("/internal/jobs/{job_name}")
async def run_job(job_name: str, api_key: str = Header(...)):
    if api_key != settings.INTERNAL_API_KEY:
        raise HTTPException(401)

    jobs = {
        "check-document-expiry": check_document_expiry,
        "send-overdue-reminders": send_overdue_reminders,
        "daily-rescreening-check": daily_rescreening_check,
        "escalate-overdue-tasks": escalate_overdue_tasks,
    }
    await jobs[job_name]()
```

Called by external cron (GitHub Actions scheduled workflow or cloud scheduler):

```yaml
# .github/workflows/scheduled-jobs.yml
name: Scheduled Jobs
on:
  schedule:
    - cron: '0 6 * * *'  # Daily at 6 AM UTC
jobs:
  run-jobs:
    runs-on: ubuntu-latest
    steps:
      - run: |
          curl -X POST https://api.compliancebrain.app/internal/jobs/check-document-expiry \
            -H "api-key: ${{ secrets.INTERNAL_API_KEY }}"
          curl -X POST https://api.compliancebrain.app/internal/jobs/send-overdue-reminders \
            -H "api-key: ${{ secrets.INTERNAL_API_KEY }}"
          curl -X POST https://api.compliancebrain.app/internal/jobs/daily-rescreening-check \
            -H "api-key: ${{ secrets.INTERNAL_API_KEY }}"
          curl -X POST https://api.compliancebrain.app/internal/jobs/escalate-overdue-tasks \
            -H "api-key: ${{ secrets.INTERNAL_API_KEY }}"
```

Simple. No extra infrastructure. Scale to a proper task queue only if jobs take > 30 seconds.

---

## Scaling Strategy

| Phase | Users | Approach |
|-------|-------|----------|
| Phase 1 | 5-10 | Single container, Supabase free/pro |
| Phase 2 | 10-50 | 2 container replicas, Supabase pro |
| Phase 3 | 50+ | Auto-scaling containers, read replicas, task queue if needed |

Scale when you actually need it, not before.

---

## Environment Variables

```bash
# .env.example

# Database
DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/compliance_brain

# Supabase
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_KEY=your-service-key

# Email
SMTP_HOST=localhost
SMTP_PORT=1025
RESEND_API_KEY=re_xxxx

# AI
ANTHROPIC_API_KEY=sk-ant-xxxx

# App
APP_ENV=development
APP_SECRET_KEY=change-me-in-production
CORS_ORIGINS=http://localhost:3000

# Internal Jobs
INTERNAL_API_KEY=change-me-in-production
```

---

## Disaster Recovery

| Scenario | Strategy | RTO |
|----------|----------|-----|
| DB corruption | Supabase point-in-time restore | < 1 hour |
| Container crash | Auto-restart (platform managed) | < 2 min |
| Complete outage | Restore from daily backup | < 4 hours |
| Data breach | Rotate keys, revoke sessions, investigate | < 30 min |

Keep it simple. Add multi-region only when the business requires it.
