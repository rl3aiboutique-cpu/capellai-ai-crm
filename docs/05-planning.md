---

# Compliance Brain - Development Planning

> Design Thinking methodology with 3-developer parallel tracks. Journey-centric sprints working backward from customer experience.
> Each sprint delivers a usable increment. Decision Gates validate direction before proceeding.
> Estimated using relative sizing (S/M/L/XL) — no absolute time estimates.

---

## Methodology: Design Thinking + Agile Sprints

```
         EMPATHIZE          DEFINE           IDEATE           PROTOTYPE          TEST
         ─────────          ──────           ──────           ─────────          ────
      Wireframe review   Journey mapping   Sprint planning   Sprint delivery   Decision Gate
      User interviews    Story selection   Track assignment   Working code      Stakeholder demo
      Pain point audit   Decision Gate     Architecture      API + UI          Feedback loop
                         criteria          decisions         Integration       Go / Pivot / Stop
```

**Core principle**: Every sprint is organized around a **customer experience milestone** — what the user can see and do at the end of the sprint. Backend and frontend for each journey are delivered together in the same sprint.

**Anti-waterfall rules:**
- No "backend-only" or "frontend-only" sprints — every sprint delivers visible value
- No story starts without its Decision Gate criteria defined
- Scope adjustments happen at Decision Gates, not mid-sprint
- If a Decision Gate reveals wrong direction, the next sprint pivots (not the current one)

---

## Phase Overview

```
Phase 1: FOUNDATION (MVP)                    Phase 2: EXPANSION
─────────────────────────                    ─────────────────────
CRM + AML + DocFactory + Workflow            Authorisations + Entities + CMP
+ Dashboards + Compliance (baseline)         + Full Compliance + AI Agents
+ IAM + Audit Trail + Regulatory Radar       + Regulatory Intelligence
+ Light Auth/Entity tracking + Agents

47 P0 stories | 9 sprints | 3 devs          37 P1+P2 stories | 3 devs
```

---

## Decision Gate Framework

Every sprint ends with a **Decision Gate** — a structured checkpoint where stakeholders validate direction.

| Element | Description |
|---------|------------|
| **Demo** | Live walkthrough of the customer experience milestone (not a slide deck) |
| **Feedback** | Stakeholders (Isahaq/Sagheer) test the increment hands-on |
| **Metrics** | Stories completed, tests passing, coverage %, known bugs |
| **Decision** | **GO** (proceed as planned) · **PIVOT** (adjust next sprint scope) · **STOP** (critical issue, halt) |
| **Artifacts** | Sprint review notes, feedback items, scope adjustments logged |

**Gate escalation**: If 2 consecutive gates result in PIVOT, trigger a Design Sprint (half-day workshop) to realign before continuing.

---

## Developer Track Assignment

### Phase 1 Tracks (3 developers working in parallel)

```
TRACK A (Senior — Infrastructure + Cross-cutting)
──────────────────────────────────────────────────
Owns: Scaffolding, shared kernel, IAM, audit trail, workflow engine,
      document factory, compliance registers, dashboards, scheduled jobs,
      notification service, deployment, data migration

TRACK B (Senior — Domain Core + Integration)
────────────────────────────────────────────
Owns: Client domain, AML full DDD (aggregates, state machines, sagas),
      risk scoring, screening, compliance domain, light tracking,
      agent framework, integration testing, performance

TRACK C (Junior — Frontend + Data, PRs reviewed by both seniors)
───────────────────────────────────────────────────────────────────
Owns: All frontend screens (~15), reusable components, data tables,
      forms, dashboards, Kanban views, responsive design, component tests
```

### Sprint-by-Sprint Delivery

Each sprint is named by its **customer experience milestone**: what the user can do at the end.

---

### Sprint 1: "I can log in and see the app"

**Stories**: US-1.1 (Auth+MFA), US-1.3 (Multi-tenant isolation)

```
TRACK A                              TRACK B                              TRACK C
──────                               ──────                               ──────
├─ Project scaffold (monorepo)       ├─ Client data model (SQLAlchemy)    ├─ Next.js scaffold + Tailwind
├─ Shared kernel (base classes)      ├─ Person/UBO data model             ├─ shadcn/ui setup
├─ Docker Compose (PG + MailHog)     ├─ DB migrations (Alembic)           ├─ Auth pages (login + MFA flow)
├─ CI pipeline (GH Actions)         ├─ Test infrastructure               ├─ App layout: sidebar + topbar
├─ Supabase setup (auth + storage)   │  (conftest, factories)             ├─ Reusable: DataTable, Form,
└─ Dev auth bypass middleware        └─ Seed data scripts                 │  Badge, StatusBadge
                                                                          └─ API client setup
                                                                             (typed fetch + TanStack Query)
```

**Decision Gate 1**: Developers can log in (or auth bypass in dev), see the app shell with sidebar navigation. Database is running with seed data. CI pipeline is green.

---

### Sprint 2: "I can see and manage my clients"

**Stories**: US-1.2 (RBAC), US-1.4 (Audit trail), US-2.1 (Client lifecycle), US-2.2 (Client CRUD), US-2.3 (UBO), US-2.4 (Client docs), US-2.5 (Corporate ownership), US-2.6 (Client search)

```
TRACK A                              TRACK B                              TRACK C
──────                               ──────                               ──────
├─ IAM context (User + Role +        ├─ Client lifecycle state machine    ├─ Client list page (wireframe 02)
│  Tenant)                           ├─ Client CRUD API endpoints         ├─ Client detail page
├─ Supabase Auth integration         ├─ UBO management API                │  (wireframe 15)
├─ RBAC middleware                   ├─ Corporate ownership structure     ├─ UBO structure display
├─ Audit trail middleware            ├─ Client document upload API        ├─ Client document upload/list
├─ Multi-tenant RLS setup           ├─ Client search + filtering API     └─ Client search + filter UI
└─ Error handling middleware         └─ Client API integration tests
```

**Decision Gate 2**: Founders can log in, see a list of clients, click into a client detail page, view UBO ownership structure, upload documents, and search/filter clients. All actions are audit-logged. Tenant isolation verified.

---

### Sprint 3: "I can start onboarding a client"

**Stories**: US-3.1 (Onboarding lifecycle), US-3.2 (Client intake), US-10.1 (Task mgmt), US-10.2 (Approvals), US-12.1 (Founder dashboard), US-12.2 (Team dashboard)

```
TRACK A                              TRACK B                              TRACK C
──────                               ──────                               ──────
├─ Workflow engine                   ├─ OnboardingCase aggregate          ├─ Onboarding case page
│  (Task aggregate)                  │  (Full DDD)                        │  (wireframe 03)
├─ Approval workflows               ├─ 8-stage state machine +           ├─ Stage tracker component
├─ SLA tracking                      │  transitions                       ├─ Founder dashboard
└─ Dashboard materialized views      ├─ Onboarding API endpoints          │  (wireframe 01)
   (mv_founder_dashboard)            ├─ Client ↔ Onboarding              ├─ Team dashboard
                                     │  integration                       │  (wireframe 09)
                                     └─ Auto-generate document            └─ KPI cards + charts
                                        checklist per client type            (Recharts)
```

**Decision Gate 3**: Users can create an onboarding case from a client, see it progress through stages with a visual tracker. Founder dashboard shows open cases, high-risk clients, overdue tasks. Team dashboard shows "My Tasks". Stage transitions enforce entry/exit criteria.

---

### Sprint 4: "I can screen clients and score risk"

**Stories**: US-3.3 (Risk scoring), US-3.4 (Screening hub), US-3.5 (Hit management), US-3.14 (Risk config), US-10.3 (Reminders)

```
TRACK A                              TRACK B                              TRACK C
──────                               ──────                               ──────
├─ Notification service (Resend)     ├─ Risk scoring engine               ├─ Screening hub page
├─ Email templates                   │  (pluggable factors)               │  (wireframe 04)
│  (reminders, alerts, escalations)  ├─ Risk scoring configuration        ├─ Hit disposition UI
├─ Reminder engine                   │  (per-tenant thresholds)           │  (false positive form)
└─ pg_cron for scheduled jobs        ├─ ScreeningCase aggregate           └─ Risk score display +
                                     ├─ Screening hit management              override UI
                                     ├─ False positive workflow
                                     │  (rationale + sign-off)
                                     └─ True match escalation
```

**Decision Gate 4**: Users can upload screening results (CSV), see hits per client, mark false positives with documented rationale, escalate true matches. Risk scoring runs with configurable factors and produces Low/Medium/High/Unacceptable. Email reminders are sent for overdue tasks.

---

### Sprint 5: "I can complete onboarding end-to-end"

**Stories**: US-3.6 (Due diligence), US-3.7 (SoW/SoF), US-3.8 (Approval routing), US-3.10 (Risk override), US-3.13 (Onboarding→Active), US-9.1 (Templates), US-9.2 (Doc auto-pop), US-9.3 (Doc versioning)

```
TRACK A                              TRACK B                              TRACK C
──────                               ──────                               ──────
├─ Document Factory context          ├─ Due diligence workflow +           ├─ Document factory UI
├─ Template management               │  checklist engine                   │  (wireframe 25)
├─ DOCX/PDF generation (fpdf2)      ├─ SoW/SoF documentation             ├─ Approval flow UI
├─ File upload/download              │  capture                           ├─ DD checklist UI
│  via Supabase Storage              ├─ Onboarding approval routing       └─ Document preview +
└─ Document versioning +             │  (risk-tier based)                     generation UI
   retention (6yr)                   ├─ Risk score override
                                     │  (dual sign-off)
                                     └─ Onboarding → Active
                                        transition saga
```

**Decision Gate 5 (CRITICAL)**: Full onboarding cycle works end-to-end: PROSPECT → Client Intake → Screening → Risk Scoring → Due Diligence → File Completion → Approval → ACTIVE. Documents auto-generated from templates. This is the **core value proposition** — the 70% time-saving claim must be believable after this demo. If this gate fails, the next sprint pivots to fix it before proceeding.

---

### Sprint 6: "I can run compliance operations"

**Stories**: US-4.1 (Policy library), US-4.2 (Policy approval), US-4.3 (Registers), US-4.4 (Incidents), US-4.5 (Training)

```
TRACK A                              TRACK B                              TRACK C
──────                               ──────                               ──────
├─ Compliance registers              ├─ Incident/breach management        ├─ Policy list + detail editor
│  (all 18 register types)           │  (5-stage state machine)           │  (wireframes 10, 11)
├─ Policy library +                  ├─ Register auto-update from         ├─ Register view with tabs
│  approval workflow                 │  domain events                     │  (wireframe 12)
│  (Draft→Review→Approved→Published) ├─ Incident escalation rules         ├─ Incident case page
└─ Training records management       │  (CRITICAL → notification)         │  (wireframe 13)
                                     └─ Integration: incident →           └─ Training records UI
                                        register entries
```

**Decision Gate 6**: Users can manage a policy library with version control and approval workflow. All 18 register types are operational. Incidents can be captured, assessed, escalated, and closed. Training records are tracked. Registers auto-update from case events (e.g., screening hit → screening log entry).

---

### Sprint 7: "I can track authorizations, entities, and regulatory changes"

**Stories**: US-14.1 (Auth case light), US-14.2 (Entity register light), US-14.3 (Post-licence handoff), US-5.1 (Regulatory Radar), US-12.4 (Global search)

```
TRACK A                              TRACK B                              TRACK C
──────                               ──────                               ──────
├─ Regulatory Radar                  ├─ Light auth case tracking           ├─ Auth pipeline Kanban
│  (basic weekly scan)               │  (US-14.1)                         │  (wireframe 05)
├─ Error handling + logging          ├─ Light entity register              ├─ Entity register list
│  (structured JSON)                 │  tracking (US-14.2)                │  (wireframe 20)
└─ Scheduled jobs:                   ├─ Post-licence handoff saga          ├─ Audit trail viewer
   check-document-expiry,            │  (US-14.3)                         │  (wireframe 26)
   send-overdue-reminders,           └─ Light tracking API                └─ Global search
   daily-rescreening-check,             integration tests                    (topbar, full-text)
   escalate-overdue-tasks
```

**Decision Gate 7**: Auth cases visible in Kanban board with stage tracking. Entity register shows all entities with traffic-light status and renewal dates. Regulatory changes logged with weekly scan results. Global search finds clients, cases, policies, entities. Audit trail is browsable and exportable.

---

### Sprint 8: "I can migrate data and the system runs autonomously"

**Stories**: US-13.1 (Client data import), US-13.3 (Template import), US-11.1 (Agent framework), US-11.2 (Risk scoring agent), US-3.9 (Re-screening), US-2.8 (Doc expiry), US-3.11 (Professional client), US-10.4 (SLA tracking)

```
TRACK A                              TRACK B                              TRACK C
──────                               ──────                               ──────
├─ Import: client data               ├─ Agent framework                   ├─ Data migration UI
│  from Excel                        │  (human-in-the-loop)               │  (import wizard)
├─ Import: templates + policies      ├─ Risk scoring agent                ├─ Regulatory change log
├─ Data migration scripts            │  (Claude API → draft score)        │  (wireframe 16)
│  (validation + dry-run)            ├─ Daily re-screening calendar       ├─ Agent draft review UI
└─ Deployment setup                  ├─ Doc expiry monitoring             │  (risk score confirmation)
   (Docker, CI/CD deploy pipeline)   │  (60/30/7 day alerts)             └─ Responsive polish
                                     ├─ Professional client assessment
                                     ├─ SLA tracking queries
                                     └─ Integration testing
                                        (full onboarding flow)
```

**Decision Gate 8**: ~30 existing clients imported from Excel. Templates and policies loaded. AI agent drafts risk scores for human confirmation. Re-screening runs daily. Document expiry alerts fire at 60/30/7 days. Data migration validated with dry-run. SLA compliance visible on dashboards.

---

### Sprint 9: "Platform is ready to go live"

**Stories**: US-13.4 (Parallel run support)

```
TRACK A                              TRACK B                              TRACK C
──────                               ──────                               ──────
├─ Parallel run support              ├─ Bug fixes from testing            ├─ Accessibility audit
├─ Go-live preparation               ├─ Performance tuning                ├─ Component testing (Vitest)
│  (rollback plan, SOPs)             │  (query optimization, N+1)        ├─ UI polish from
├─ Security review                   ├─ Security review                   │  stakeholder feedback
│  (OWASP checklist)                 │  (pen test basics)                └─ Cross-browser testing
└─ Monitoring setup                  ├─ E2E smoke test
   (Sentry + health checks)         └─ Parallel run validation
```

**Decision Gate 9 (FINAL)**: All P0 stories pass acceptance criteria. Data migration complete and validated. Parallel run successful (2-4 weeks alongside legacy). Security audit clean. Performance acceptable (< 2s p95 for 10 concurrent users). Stakeholder sign-off. **GO-LIVE**.

---

### Phase 1 Dependency Graph

```
Sprint 1 (Foundation):
  Track A (scaffolding, Docker, CI) ──must complete before──► Track B starts DB work
  Track C (frontend setup) ────────── independent ───────────► starts immediately

Sprint 2 (Clients):
  Track A (IAM, RBAC, audit) ────────must complete before──► Track B uses auth/audit
  Track B (client APIs) ─────────────── needs ──────────────► Track A Sprint 1 scaffold
  Track C (client pages) ───────────── needs ──────────────► Track B Sprint 2 APIs

Sprint 3-4 (Onboarding + Screening):
  Track A (workflow engine, notifications) ──before──► Track B uses tasks/approvals/reminders
  Track B (AML domain) ─────────────── needs ────────► Track A Sprint 2 IAM
  Track C (frontend pages) ──────────── needs ────────► Track B APIs from same sprint

Sprint 5 (End-to-End — CRITICAL GATE):
  Track A (doc factory) + Track B (DD + approvals) ──► converge for full onboarding flow
  Track C integrates both Track A and Track B outputs into unified UI

Sprint 6-7 (Compliance + Tracking):
  Track B (light tracking) ──────── needs ──────────► Track A Sprint 3 workflow engine
  Track A + B can otherwise work independently

Sprint 8-9 (Migration + Go-Live):
  All tracks converge ─── integration testing requires all APIs stable
  No new dependencies — focused on testing, migration, and polish
```

### Key Sprint 1 Deliverables (All Tracks Must Complete)

| Track | Deliverable | Dependency |
|-------|------------|------------|
| A | Project scaffold + Docker + CI + Supabase | None (first) |
| A | Shared kernel base classes + dev auth bypass | None |
| B | Client + Person data models + migrations + seed data | Track A scaffolding |
| C | Next.js setup + auth pages + layout + reusable components | None (parallel) |

---

## Phase 2 Tracks (After Phase 1 MVP)

```
TRACK A (Senior)                      TRACK B (Senior)
────────────────                      ────────────────
Authorisations + RFI + CMP           Entity Management + AI Agents

Sprint 10-11:                         Sprint 10-11:
 ├─ AuthorisationCase aggregate        ├─ LegalEntity aggregate
 ├─ Stage-gated workflow               ├─ Entity register CRUD
 ├─ Licence categories config          ├─ Obligation tracking
 ├─ QA review workflow                 ├─ Nested ownership structures
 └─ Auth case API                      └─ Entity API endpoints

Sprint 12-13:                         Sprint 12-13:
 ├─ RFI tracking aggregate            ├─ Screening triage agent
 ├─ Document pack generator           ├─ Policy drafter agent
 ├─ MonitoringTest aggregate           ├─ Document auto-fill agent
 ├─ Test library import (CMP xlsx)    ├─ Obligation reminders + escalation
 ├─ CMP scheduling + execution        ├─ Compliance health calculation
 └─ Finding + remediation tracking    └─ Renewal pack generation

TRACK C (Junior)
────────────────
Frontend (Phase 2 screens)

Sprint 10-11:
 ├─ Auth pipeline view + case detail
 ├─ RFI tracker UI
 ├─ Entity register UI + detail + vault
 └─ Obligations calendar

Sprint 12-13:
 ├─ CMP test library + execution UI
 ├─ Findings + remediation UI
 ├─ Entity dashboard
 └─ AI agent draft review UI (expanded)
```

---

## Phase 3 Tracks (Full Brain)

```
TRACK A: Regulatory Intelligence      TRACK B: Full Compliance + Board Packs
 ├─ Regulatory change detection         ├─ Full policy lifecycle
 ├─ Impact assessment workflow          ├─ Board pack builder agent
 ├─ Client alert management            ├─ Advanced training: e-learning, certs
 ├─ Weekly digest                       ├─ Audit/regulator visit prep
 ├─ Regulatory Q&A Copilot            └─ Cross-vertical reporting
 └─ Downstream triggers

TRACK C: Advanced AI + Integrations
 ├─ Screening API integration (World-Check)
 ├─ Agent accuracy metrics
 ├─ Async event bus (if needed)
 ├─ Digital signature integration
 └─ Advanced search (full-text docs) + settings panels
```

---

## Sprint Ceremonies

Each sprint is **2 weeks**:

| Ceremony | When | Duration | Purpose |
|----------|------|----------|---------|
| Sprint Planning | Day 1 (Mon) | 1 hour | Select stories, define Decision Gate criteria |
| Daily Standup | Daily | 15 min | Track blockers, sync between tracks |
| Mid-Sprint Check | Day 5 (Fri) | 30 min | Demo progress, identify integration gaps early |
| Decision Gate | Day 10 (Fri) | 1.5 hours | Live demo + stakeholder feedback + Go/Pivot/Stop |
| Sprint Retro | Day 10 (Fri) | 30 min | Process improvements |

### Sprint Capacity Rules

- Each dev handles 2-4 stories per sprint (depending on size)
- Max 1 XL story per developer per sprint
- Always include 1 story of tech debt / quality improvement
- 20% buffer for bugs, code review, and integration work
- Claude Code Max expected to accelerate all tracks significantly

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
- [ ] Demo'd in Decision Gate

---

## Risk Register

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|------------|
| Scope creep in Phase 1 | High | High | Strict P0 story selection, defer P1/P2. Decision Gates enforce scope discipline |
| CRA Excel not provided in time | High | Medium | Risk scoring abstracted with manual config. Works without CRA |
| Junior learning curve | Medium | Medium | Claude Code Max accelerates. Both seniors review PRs. Pair programming for complex components |
| 3 developers bottleneck vs 4-dev plan | Medium | Medium | Extended to 9 sprints for sustainable pace. Claude Code Max compensates |
| Supabase free tier limitations | Low | Low | Upgrade to Pro ($25/mo). Fallback: self-hosted PG + custom auth |
| AI agent accuracy insufficient | Medium | Medium | All agents in draft mode, human confirms everything |
| Cross-context integration bugs | High | Medium | In-process event bus + integration tests from Sprint 5. Decision Gate 5 is critical checkpoint |
| Data migration data quality | Medium | High | Validation scripts, dry-run mode, manual review in Sprint 8 |
| Decision Gate reveals wrong direction | Medium | Low | Pivot mechanism built into process. Max 1-sprint correction cost |
| Screening provider API unavailable | Low | Low | Phase 1 uses manual upload, API integration Phase 2 |

---

## Branch Strategy

```
main (production)
  └── develop (staging / integration)
       ├── feat/US-1.1-auth-mfa           (Track A)
       ├── feat/US-2.1-client-lifecycle    (Track B)
       └── feat/US-12.1-founder-dashboard  (Track C)
```

- Feature branches from `develop`
- PR + code review required before merge
- CI must pass before merge allowed
- `develop` → `main` via release PR (manual approval)
- Hotfix branches from `main` (merge back to both `main` and `develop`)

### Context Ownership (Prevents Conflicts)

| Bounded Context | Primary Owner | Backup |
|----------------|---------------|--------|
| Identity (IAM + Tenant) + Shared Kernel | Track A | Track B |
| Audit Trail + Workflow Engine | Track A | Track B |
| Client Management (CRM) | Track B | Track A |
| AML Onboarding + Screening | Track B | Track A |
| Compliance Framework | Track A (registers/policies) + Track B (incidents/domain) | — |
| Document Factory | Track A | Track B |
| Light Tracking (auth + entities) | Track B | Track A |
| Regulatory Radar (basic) | Track A | Track B |
| Notification Service | Track A | Track B |
| Agent Framework | Track B | Track A |
| Frontend (all) | Track C | Any backend dev for API types |

---

## Phase 1 Milestone Map (9 Sprints)

```
Month 1 (Sprint 1-2): "SEE MY WORLD"
├─ Backend scaffolding + CI/CD operational
├─ Auth working (Supabase login + MFA)
├─ Client CRUD + UBO management operational
├─ Corporate ownership structure visible
├─ Frontend layout + auth flow + client pages
├─ RBAC enforced, audit trail recording
└─ GATE: Founders can login, browse clients, view details, upload docs

Month 2 (Sprint 3-4): "ONBOARD & SCREEN"
├─ Onboarding case lifecycle (8 stages)
├─ Workflow engine + task management operational
├─ Risk scoring engine functional (pluggable factors)
├─ Screening hub (manual upload) + hit management
├─ Dashboards populated with real data
├─ Notification service sending reminders
└─ GATE: Can create onboarding, screen clients, score risk. Dashboards live

Month 3 (Sprint 5-6): "END-TO-END OPERATIONS"
├─ Full onboarding cycle: PROSPECT → ACTIVE
├─ Document Factory generating DOCX/PDF from templates
├─ Due diligence workflow with checklists
├─ All compliance registers operational
├─ Policy library + approval workflow
├─ Incident/breach management
├─ Training records
└─ GATE (CRITICAL): Complete onboarding + compliance operations working

Month 4 (Sprint 7-8): "FULL PLATFORM"
├─ Light auth pipeline + entity register
├─ Post-licence handoff saga
├─ Regulatory Radar (basic weekly scan)
├─ Global search operational
├─ AI agent drafting risk scores
├─ Daily re-screening + doc expiry monitoring
├─ Data migration (clients, templates, policies)
├─ Deployment pipeline ready
└─ GATE: All Phase 1 features complete. Data migrated

Month 4.5 (Sprint 9): "GO-LIVE"
├─ Integration testing + security review
├─ Performance tuning
├─ Parallel run (2-4 weeks with legacy)
├─ Training sessions (admin + users)
├─ UI polish + accessibility
└─ GATE (FINAL): Phase 1 Live
```

---

## P0 Story Coverage Map

All 47 P0 stories mapped to sprints:

| Sprint | Stories Delivered |
|--------|-----------------|
| 1 | US-1.1, US-1.3 |
| 2 | US-1.2, US-1.4, US-2.1, US-2.2, US-2.3, US-2.4, US-2.5, US-2.6 |
| 3 | US-3.1, US-3.2, US-10.1, US-10.2, US-12.1, US-12.2 |
| 4 | US-3.3, US-3.4, US-3.5, US-3.14, US-10.3 |
| 5 | US-3.6, US-3.7, US-3.8, US-3.10, US-3.13, US-9.1, US-9.2, US-9.3 |
| 6 | US-4.1, US-4.2, US-4.3, US-4.4, US-4.5 |
| 7 | US-14.1, US-14.2, US-14.3, US-5.1, US-12.4 |
| 8 | US-13.1, US-13.3, US-11.1, US-11.2, US-3.9, US-2.8, US-3.11, US-10.4 |
| 9 | US-13.4 |

---

## Go-Live Checklist

- [ ] All P0 stories pass acceptance criteria
- [ ] All 9 Decision Gates passed (GO)
- [ ] Security audit: no critical/high vulnerabilities
- [ ] Load test: handles 10 concurrent users with < 2s p95 response time
- [ ] Backup/restore verified: successful DR test
- [ ] Monitoring alerts configured and tested (Sentry + health check)
- [ ] Data migration dry-run successful + production migration complete
- [ ] User training complete (2 sessions: admin + users)
- [ ] Quick-start SOPs written
- [ ] Rollback plan documented and tested
- [ ] Parallel run completed (2-4 weeks)
- [ ] Stakeholder sign-off (Isahaq + Sagheer)
