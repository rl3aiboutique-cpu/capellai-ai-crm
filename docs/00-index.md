# Compliance Brain Platform - Documentation Index

> All-in-one operating system for compliance, AML, regulatory intelligence, and authorisations.
> Built for Capellai (founders: Mohammed Isahaq Aslam & Sagheer).

## Documents

| # | Document | Description |
|---|----------|-------------|
| 01 | [Epics & User Stories](./01-epics-and-user-stories.md) | Complete feature map: 14 epics, 84 user stories with acceptance criteria |
| 02 | [DDD Architecture](./02-ddd-architecture.md) | Hybrid DDD: full DDD (AML), service layer (client/compliance), simple CRUD (rest) |
| 03 | [Code Architecture](./03-code-architecture.md) | Tech stack, project structure, patterns (DDD/SOLID/DI), coding standards |
| 04 | [Infrastructure](./04-infrastructure.md) | Docker + Supabase deployment, CI/CD, security, monitoring, disaster recovery |
| 05 | [Planning](./05-planning.md) | Design Thinking methodology, journey-centric sprints, 3-developer parallel tracks |
| 06 | [Data Model](./06-data-model.md) | Core database schema, multi-tenancy, relationships |
| 07 | [Architecture Diagrams](./07-architecture-diagrams.md) | C4 diagrams, deployment, feature map, async architecture |
| 08 | [ADRs & Tech Debt](./08-adrs-and-tech-debt.md) | 12 Architecture Decision Records + 20 known tech debt items |

## Quick Reference

- **Tech Stack**: Python 3.12+ / FastAPI / Pydantic v2 / SQLAlchemy 2.0 | Next.js 15 / React 19 / TypeScript / Tailwind / shadcn/ui
- **Database**: Supabase (PostgreSQL)
- **Cloud**: Docker local + Cloud TBD (Supabase Cloud / GCP)
- **Auth**: Supabase Auth with MFA
- **AI Agents**: Claude API (Anthropic SDK)
- **Phases**: 4 phases, Phase 1 = CRM + AML + Document Factory + Compliance baseline + Regulatory Radar basic + Light Auth/Entity tracking

## Source Documentation

All requirements derived from:
- `referencias/Compliance Brain Platform.pdf` (49-page platform spec)
- `referencias/Compliance Brain - IA Answered questionnaire.docx` (170+ answered domain questions)
- `referencias/wireframes (3)/` (26 HTML wireframe screens)
- `referencias/Template documents/` (compliance document templates)
