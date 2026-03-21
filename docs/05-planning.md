---

# Compliance Brain - Development Planning

> Phased delivery with 4-developer parallel tracks. Each phase delivers standalone value.
> Estimated using relative sizing (S/M/L/XL) — no absolute time estimates.

---

## Phase Overview

```
Phase 1: FOUNDATION (MVP)                    Phase 2: EXPANSION
─────────────────────────                    ─────────────────────
CRM + AML + DocFactory + Workflow            Authorisations + Entities + CMP
+ Dashboards + Compliance (baseline)         + Full Compliance + AI Agents
+ IAM + Audit Trail + Regulatory Radar       + Regulatory Intelligence
+ Light Auth/Entity tracking + Agents

47 P0 stories                                37 P1+P2 stories
```

---

## Developer Track Assignment

### Phase 1 Tracks (4 developers working in parallel)

```
TRACK A (Senior 1 — Foundation + Infra)     TRACK B (Senior 2 — Domain Core)
──────────────────────────────────────        ────────────────────────────────────
Sprint 1:                                    Sprint 1:
 ├─ Project scaffold (monorepo)               ├─ Client aggregate + domain
 ├─ Shared kernel (base classes)              ├─ Person/UBO aggregate + shared entity
 ├─ Docker Compose (PG + MailHog)             ├─ Client lifecycle state machine
 ├─ CI pipeline (GH Actions)                 └─ Client CRUD API endpoints
 └─ Supabase setup (auth + storage)
                                             Sprint 2:
Sprint 2:                                     ├─ OnboardingCase aggregate
 ├─ IAM context (User + Role + Tenant)        ├─ Stage state machine + transitions
 ├─ Supabase Auth integration                ├─ Onboarding API endpoints
 ├─ RBAC middleware                          └─ Client ↔ Onboarding integration
 ├─ Audit trail middleware
 └─ Multi-tenant RLS setup                  Sprint 3:
                                              ├─ Risk scoring engine (pluggable)
Sprint 3:                                     ├─ ScreeningCase aggregate
 ├─ Workflow engine (Task aggregate)          ├─ Screening hit management
 ├─ Approval workflows                       ├─ False positive workflow
 ├─ Reminder engine                          └─ True match escalation
 ├─ pg_cron for scheduled jobs
 └─ SLA tracking                             Sprint 4:
                                              ├─ Due diligence workflow + checklist
Sprint 4:                                     ├─ SoW/SoF documentation
 ├─ Document Factory context                  ├─ Onboarding approval routing
 ├─ Template management                      ├─ Risk score override (dual sign-off)
 ├─ DOCX/PDF generation (fpdf2)              └─ Onboarding → Active transition saga
 ├─ File upload/download via Storage
 └─ Document versioning + retention          Sprint 5:
                                              ├─ Daily re-screening calendar
Sprint 5:                                     ├─ Doc expiry monitoring (60/30/7)
 ├─ Compliance registers (all types)          ├─ Professional client assessment
 ├─ Incident/breach management               ├─ Corporate ownership structure
 ├─ Policy library + approval workflow       └─ Agent framework + risk scoring agent
 └─ Training records management
                                             Sprint 6:
Sprint 6:                                     ├─ Light: auth case tracking (US-14.1)
 ├─ Dashboard materialized views              ├─ Light: entity register tracking (US-14.2)
 ├─ Regulatory Radar (basic weekly scan)      ├─ Post-licence handoff saga (US-14.3)
 ├─ Data migration scripts                   └─ Integration testing
 ├─ Error handling + logging
 └─ Notification service (Resend)            Sprint 7:
                                              ├─ Bug fixes + polish
Sprint 7:                                     ├─ Performance tuning
 ├─ Import: client data from Excel            └─ Security review
 ├─ Import: templates/policies
 ├─ Parallel run support
 └─ Go-live preparation

TRACK C (Junior — Frontend, seniors review PRs)
────────────────────────────────────────────────
Sprint 1:
 ├─ Next.js scaffold + Tailwind + shadcn/ui
 ├─ Auth pages (login + MFA flow)
 ├─ App layout: sidebar + topbar (wireframe CSS)
 ├─ Reusable: DataTable, Form, Badge, StatusBadge
 └─ API client setup (typed fetch + TanStack Query)

Sprint 2:
 ├─ Client list page (wireframe 02)
 ├─ Client detail page (wireframe 15)
 ├─ UBO structure display
 └─ Client document upload/list

Sprint 3:
 ├─ Onboarding case page (wireframe 03)
 ├─ Stage tracker component
 ├─ Screening hub page (wireframe 04)
 └─ Hit disposition UI (false positive form)

Sprint 4:
 ├─ Founder dashboard (wireframe 01)
 ├─ Team dashboard (wireframe 09)
 ├─ KPI cards + charts (Recharts)
 └─ Document factory UI (wireframe 25)

Sprint 5:
 ├─ Policy list + detail editor (wireframes 10, 11)
 ├─ Register view with tabs (wireframe 12)
 ├─ Incident case page (wireframe 13)
 └─ Training records UI

Sprint 6:
 ├─ Auth pipeline Kanban (wireframe 05)
 ├─ Entity register list (wireframe 20)
 ├─ Audit trail viewer (wireframe 26)
 └─ Global search (topbar search)

Sprint 7:
 ├─ Regulatory change log (basic, wireframe 16)
 ├─ Data migration UI (import wizard)
 ├─ Responsive polish + accessibility
 └─ Component testing (Vitest)

TRACK D (Dev 4 — Full-stack, AML co-owner + ops)
──────────────────────────────────────────────────
Sprint 1:
 ├─ Support Track A: shared kernel + DB migrations
 └─ Set up test infrastructure (conftest, factories)

Sprint 2:
 ├─ Onboarding API endpoints (co-own with Dev 2)
 └─ Client API integration tests

Sprint 3:
 ├─ Screening API endpoints + upload handler
 ├─ Risk scoring API endpoints
 └─ Notification service (Resend integration)

Sprint 4:
 ├─ Due diligence API endpoints
 ├─ Approval workflow API endpoints
 └─ Email templates (reminders, alerts, escalations)

Sprint 5:
 ├─ Compliance register API endpoints
 ├─ Incident/breach API endpoints
 └─ Policy API endpoints

Sprint 6:
 ├─ Scheduled jobs setup (pg_cron + HTTP endpoints)
 ├─ Data migration scripts (Excel import)
 └─ Deployment setup (Docker, CI/CD deploy pipeline)

Sprint 7:
 ├─ Integration testing (full onboarding flow)
 ├─ E2E smoke test
 ├─ Security review + pen test basics
 └─ Go-live preparation
```

### Phase 1 Dependency Graph

```
Sprint 1 (Foundation):
  Track A (scaffolding, Docker, CI) ──must complete before──► Track B starts DB work
  Track C (frontend setup) ────────── independent ───────────► can start immediately
  Track D supports Track A

Sprint 2:
  Track A (IAM, RBAC, audit) ────────must complete before──► Track B uses auth/audit
  Track B (onboarding) ──────────────── needs ──────────────► Track A Sprint 1 entities

Sprint 3-4:
  Track A (workflow engine, doc factory) ──must complete before──► Track B uses tasks/approvals
  Track B (risk, screening) ───────────── needs ──────────────► Track A Sprint 2 IAM
  Track C (frontend pages) ────────────── needs ──────────────► Track B API endpoints

Sprint 5-6:
  Track B (compliance + light tracking) ── needs ──────────────► Track A Sprint 3 workflow engine
  All tracks can otherwise work independently

Sprint 7:
  All tracks focused on polish, migration, testing — no new dependencies
```

### Key Sprint 1 Deliverables (All Tracks Must Complete)

| Track | Deliverable | Dependency |
|-------|------------|------------|
| A | Project scaffold + Docker + CI + Supabase | None (first) |
| A | Shared kernel base classes | None |
| B | Client aggregate + CRUD API | Track A scaffolding |
| C | Next.js setup + auth + layout + reusable components | None (parallel) |
| D | Support Track A with shared kernel + DB setup | Track A scaffolding |

---

## Phase 2 Tracks (After Phase 1 MVP)

```
TRACK A (Dev 1)                       TRACK B (Dev 2)
────────────────                      ────────────────
Authorisations + RFI                  Entity Management + Obligations

Sprint 7-8:                           Sprint 7-8:
 ├─ AuthorisationCase aggregate        ├─ LegalEntity aggregate
 ├─ Stage-gated workflow               ├─ Entity register CRUD
 ├─ Licence categories config          ├─ Obligation tracking
 ├─ QA review workflow                 ├─ Nested ownership structures
 └─ Auth case API                      └─ Entity API endpoints

Sprint 9-10:                          Sprint 9-10:
 ├─ RFI tracking aggregate             ├─ Obligation reminders
 ├─ Document pack generator            ├─ Escalation workflows
 ├─ Precedent library                  ├─ Document vault per entity
 ├─ Post-licensing handoff event       ├─ Compliance health calculation
 └─ Pipeline dashboard API             └─ Renewal pack generation

TRACK C (Dev 3)                       TRACK D (Dev 4)
────────────────                      ────────────────
CMP Monitoring + AI Agents            Frontend (Phase 2 screens)

Sprint 7-8:                           Sprint 7-8:
 ├─ MonitoringTest aggregate            ├─ Auth pipeline view
 ├─ Test library import (CMP xlsx)      ├─ Auth case detail
 ├─ ScheduledTest aggregate             ├─ RFI tracker UI
 ├─ Test scheduling engine              ├─ Entity register UI
 └─ CMP API endpoints                   └─ Entity detail + vault

Sprint 9-10:                          Sprint 9-10:
 ├─ Working paper execution             ├─ CMP test library UI
 ├─ Finding management                  ├─ CMP execution + findings UI
 ├─ Remediation tracking                ├─ Obligations calendar
 ├─ Screening triage agent              ├─ Entity dashboard
 ├─ Policy drafter agent                └─ AI agent draft review UI
 └─ Document auto-fill agent
```

---

## Phase 3 Tracks (Full Brain)

```
TRACK A: Regulatory Intelligence      TRACK B: Full Compliance + Board Packs
 ├─ Regulatory change detection         ├─ Full policy lifecycle
 ├─ Impact assessment workflow          ├─ Board pack builder agent
 ├─ Client alert management            ├─ Advanced training: e-learning integration, certification tracking
 ├─ Weekly digest                       ├─ Audit/regulator visit prep
 ├─ Regulatory Q&A Copilot            └─ Cross-vertical reporting
 └─ Downstream triggers

TRACK C: Advanced AI + Integrations   TRACK D: Frontend (Phase 3)
 ├─ Screening API integration           ├─ Regulatory intelligence UI
 ├─ Screening triage agent              ├─ Change log + impact views
 ├─ Policy drafter agent                ├─ Client alert editor
 ├─ Agent accuracy metrics              ├─ Board pack builder UI
 ├─ Async event bus (if needed)        ├─ Advanced search (full-text)
 └─ Digital signature integration      └─ Settings + admin panels
```

---

## Sprint Planning Template

Each sprint is **2 weeks**. Ceremonies:

| Ceremony | When | Duration | Purpose |
|----------|------|----------|---------|
| Sprint Planning | Day 1 (Mon) | 1 hour | Select stories, clarify acceptance criteria |
| Daily Standup | Daily | 15 min | Track blockers, sync between tracks |
| Mid-Sprint Review | Day 5 (Fri) | 30 min | Demo progress, identify integration issues |
| Sprint Review | Day 10 (Fri) | 1 hour | Demo to stakeholders (Isahaq/Sagheer) |
| Sprint Retro | Day 10 (Fri) | 30 min | Process improvements |

### Sprint Capacity Rules

- Each dev handles 2-4 stories per sprint (depending on size)
- Max 1 XL story per developer per sprint
- Always include 1 story of tech debt / quality improvement
- 20% buffer for bugs, code review, and integration work

---

## Story Sizing Guide

| Size | Complexity | Example |
|------|-----------|---------|
| **S** (Small) | Simple CRUD, config, UI page with existing patterns | Add a new register type |
| **M** (Medium) | New aggregate with business rules, new UI page with complex state | Client lifecycle management |
| **L** (Large) | Complex domain logic, multiple aggregates interacting, new workflow | Onboarding case with stage gates |
| **XL** (Extra Large) | Cross-context integration, new infrastructure, complex UI flows | Risk scoring engine with pluggable factors |

---

## Definition of Done

A story is DONE when:

- [ ] Code implements all acceptance criteria
- [ ] Unit tests: 90%+ domain, 80%+ application, 60%+ infrastructure
- [ ] Integration tests for API endpoints
- [ ] Frontend tests for critical user flows
- [ ] Code reviewed by at least one other developer
- [ ] No ruff/mypy/eslint warnings
- [ ] Pre-commit hooks pass
- [ ] CI pipeline passes (all checks green)
- [ ] API documented (OpenAPI auto-generated)
- [ ] Audit trail entries verified for state-changing operations
- [ ] Multi-tenancy verified (tenant isolation tested)
- [ ] Demo'd in sprint review

---

## Risk Register

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|------------|
| Scope creep in Phase 1 | High | High | Strict P0 story selection, defer P1/P2 |
| CRA Excel not provided in time | High | Medium | Risk scoring abstracted with manual config. Works without CRA |
| Junior frontend bottleneck | Medium | Medium | Seniors can do API-first. Claude Code Max accelerates frontend dev |
| Supabase free tier limitations | Low | Low | Upgrade to Pro ($25/mo). Fallback: self-hosted PG + custom auth |
| AI agent accuracy insufficient | Medium | Medium | All agents in draft mode, human confirms everything |
| Cross-context integration bugs | High | Medium | In-process event bus + integration tests in Sprint 5-7 |
| Data migration data quality | Medium | High | Validation scripts, dry-run mode, manual review |
| 4 developers stepping on each other | Medium | Medium | Clear context ownership, feature branches, daily standups |
| Screening provider API unavailable | Low | Low | Phase 1 uses manual upload, API integration Phase 2 |

---

## Branch Strategy

```
main (production)
  └── develop (staging / integration)
       ├── feat/US-1.1-auth-mfa           (Dev 1 - Track A)
       ├── feat/US-2.1-client-lifecycle    (Dev 2 - Track B)
       ├── feat/US-3.1-onboarding-case     (Dev 3 - Track C)
       └── feat/US-12.1-founder-dashboard  (Dev 4 - Track D)
```

- Feature branches from `develop`
- PR + code review required before merge
- CI must pass before merge allowed
- `develop` → `main` via release PR (manual approval)
- Hotfix branches from `main` (merge back to both `main` and `develop`)

### Context Ownership (Prevents Conflicts)

| Bounded Context | Primary Owner | Backup |
|----------------|---------------|--------|
| Identity (IAM + Tenant) + Shared Kernel | Dev 1 | Dev 4 |
| Audit Trail + Workflow Engine | Dev 1 | Dev 2 |
| Client Management (CRM) | Dev 2 | Dev 4 |
| AML Onboarding + Screening | Dev 2 (domain) + Dev 4 (API) | Dev 1 |
| Compliance Framework | Dev 4 (API) + Dev 1 (domain) | Dev 2 |
| Document Factory | Dev 1 | Dev 2 |
| Light Tracking (auth + entities) | Dev 2 | Dev 4 |
| Regulatory Radar (basic) | Dev 1 | Dev 4 |
| Notification Service | Dev 4 | Dev 1 |
| Frontend (all) | Dev 3 (junior) | Any backend dev for API types |

---

## Phase 1 Milestone Map (7 Sprints)

```
Month 1 (Sprint 1-2): FOUNDATION
├─ Backend scaffolding + CI/CD operational
├─ Auth working (Supabase login + MFA)
├─ Client CRUD + UBO management operational
├─ First onboarding case creatable with stage transitions
├─ Frontend layout + auth flow + client pages
└─ MILESTONE: Can login, create client, start onboarding

Month 2 (Sprint 3-4): CORE WORKFLOWS
├─ RBAC enforced, audit trail recording
├─ Risk scoring engine functional
├─ Screening hub (manual upload) + hit management
├─ Due diligence workflow + approval routing
├─ Document Factory generating DOCX/PDF
├─ Dashboards populated with real data
└─ MILESTONE: End-to-end onboarding cycle works

Month 3 (Sprint 5-6): COMPLIANCE + TRACKING
├─ All compliance registers operational
├─ Policy library + approval workflow
├─ Incident/breach management
├─ Training records
├─ Daily re-screening + doc expiry monitoring
├─ Light auth pipeline + entity register
├─ Regulatory Radar (basic weekly scan)
├─ Agent framework + risk scoring agent (draft mode)
└─ MILESTONE: All Phase 1 features complete

Month 3.5 (Sprint 7): POLISH + GO-LIVE
├─ Data migration (clients, templates, policies)
├─ Integration testing + security review
├─ Performance tuning
├─ Parallel run (2-4 weeks with legacy)
├─ Training sessions (admin + users)
└─ MILESTONE: Phase 1 Live
```

---

## Go-Live Checklist

- [ ] All P0 stories pass acceptance criteria
- [ ] Security audit: no critical/high vulnerabilities
- [ ] Load test: handles 10 concurrent users with < 2s p95 response time
- [ ] Backup/restore verified: successful DR test
- [ ] Monitoring alerts configured and tested
- [ ] Data migration dry-run successful
- [ ] User training complete (2 sessions)
- [ ] Quick-start SOPs written
- [ ] Rollback plan documented and tested
- [ ] Parallel run completed (2-4 weeks)
- [ ] Stakeholder sign-off (Isahaq + Sagheer)
