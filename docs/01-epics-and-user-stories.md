# Compliance Brain - Epics & User Stories

> Complete feature mapping derived from platform specification, answered questionnaire (170+ questions), and wireframe analysis.
> Priority: **P0** = Phase 1 must-have | **P1** = Phase 1 nice-to-have | **P2** = Phase 2+ | **P3** = Phase 3+

---

## EPIC 1: Platform Foundation & Identity (IAM)

> Core authentication, authorization, multi-tenancy, and audit infrastructure that ALL other features depend on.

### US-1.1: User Authentication with MFA `P0`
**As a** platform user **I want to** log in securely with email/password and MFA **so that** my account is protected against unauthorized access.

**Acceptance Criteria:**
- [ ] Login with email + password
- [ ] MFA required for all users (TOTP or SMS)
- [ ] Session management with configurable timeout
- [ ] Password reset flow
- [ ] Failed login lockout after N attempts
- [ ] Audit log entry for every auth event

### US-1.2: Role-Based Access Control (RBAC) `P0`
**As a** Master Admin **I want to** assign roles to users with granular permissions per bounded context **so that** each team member only accesses what they need.

**Acceptance Criteria:**
- [ ] Roles: Master Admin, Compliance Officer/MLRO, Compliance Analyst, Operations/Admin, Viewer (read-only)
- [ ] Permissions configurable per bounded context (CRM, AML, Compliance, etc.)
- [ ] Master Admins (Isahaq/Sagheer) have full platform access
- [ ] Analysts can create/edit but cannot approve
- [ ] Viewer role is read-only across all contexts
- [ ] Limited per-user permission overrides (admin-controlled)
- [ ] UnauthorizedActionError raised on permission violation

### US-1.3: Multi-Tenant Firm Isolation `P0`
**As a** compliance consultancy **I want** each client firm to be completely isolated **so that** data from one firm is never visible to another.

**Acceptance Criteria:**
- [ ] Every record carries a TenantId
- [ ] Row-level security enforces tenant isolation at database level
- [ ] Users are assigned to one or more tenants
- [ ] Cross-tenant queries are impossible from application layer
- [ ] Tenant configuration: active regulators, thresholds, SLAs
- [ ] Internal group entities also modeled as tenants

### US-1.4: Immutable Audit Trail `P0`
**As a** compliance officer **I want** every action logged immutably **so that** regulators can inspect a complete, tamper-proof history.

**Acceptance Criteria:**
- [ ] Append-only storage (no update, no delete)
- [ ] Every entry: actor (user/agent), timestamp, action, object_id, before/after summary, source (UI/API), correlation_id
- [ ] Agent actions labeled with agent identity
- [ ] Exportable as CSV/PDF with filters
- [ ] Retention: minimum 6 years (configurable)
- [ ] Searchable by actor, date range, object, action type

### US-1.5: Tenant Configuration `P1`
**As a** Master Admin **I want to** configure per-tenant settings **so that** each client firm operates under its own regulatory rules.

**Acceptance Criteria:**
- [ ] Active regulators (DFSA, FSRA, UAE CB, VARA, DIFC, ADGM)
- [ ] Risk score thresholds (Low/Medium/High/Unacceptable)
- [ ] UBO ownership threshold (10% or 25%, configurable)
- [ ] SLA defaults for onboarding, remediation, etc.
- [ ] Reminder cadence (60/30/14/7/1 days before due)
- [ ] Document retention period

---

## EPIC 2: CRM / Client Management

> The central data model. Every other context references Client by ClientId. Getting this right is the highest-priority design decision.

### US-2.1: Client Lifecycle Management `P0`
**As a** team member **I want to** manage clients through defined lifecycle states **so that** every client's status is clear and transitions are controlled.

**Acceptance Criteria:**
- [ ] States: PROSPECT → ONBOARDING → ACTIVE → UNDER_REVIEW → SUSPENDED → EXITED
- [ ] State transition rules enforced (not all transitions valid)
- [ ] Transition requires authorized role
- [ ] Every transition logged in audit trail
- [ ] Dashboard shows client count per state

### US-2.2: Client Record CRUD `P0`
**As a** team member **I want to** create and manage client records **so that** all client information is centralized.

**Acceptance Criteria:**
- [ ] Client types: Individual/HNWI, Corporate, Fund/Fund Manager, Foundation, Fintech/VASP
- [ ] Required fields: name, type, jurisdiction, contact details, engagement date
- [ ] Client record created on: inbound lead, engagement kickoff, or manual entry
- [ ] Any internal staff can create a client record
- [ ] Search by name, ID, type, jurisdiction, status, risk tier
- [ ] Bulk export capability

### US-2.3: UBO Management `P0`
**As a** compliance analyst **I want to** manage Ultimate Beneficial Owners as shared entities **so that** a person who is UBO of multiple clients is tracked once.

**Acceptance Criteria:**
- [ ] UBO modeled as shared Person/Party entity (not embedded in Client)
- [ ] Same person can be UBO of multiple clients
- [ ] Configurable ownership threshold (default 25%, option for 10%)
- [ ] Mandatory fields: full name, nationality, DOB, ID/passport number, ownership %, role, address (country/city), screening status/date
- [ ] Optional fields: phone, email, occupation
- [ ] PEP status flag → auto-triggers risk tier HIGH minimum
- [ ] Re-verification schedule: annually (6 months for HIGH/VERY_HIGH)

### US-2.4: Client Document Management `P0`
**As a** operations staff **I want to** upload, version, and track documents per client **so that** all evidence is centralized with complete history.

**Acceptance Criteria:**
- [ ] Upload documents (PDF, DOCX, XLSX, images)
- [ ] Full version history (all historical versions retained)
- [ ] Document categories: ID, corporate docs, ownership, financials, engagement, etc.
- [ ] Delete allowed with audit log
- [ ] Expiry date tracking for identity documents
- [ ] Automated alerts: 60/30/7 days before expiry
- [ ] Move to UNDER_REVIEW if document expired and not refreshed

### US-2.5: Corporate Ownership Structure `P0`
**As a** compliance analyst **I want to** model nested corporate ownership chains **so that** I can trace beneficial ownership to natural persons.

**Acceptance Criteria:**
- [ ] Entity can own another entity (parent_entity_id reference)
- [ ] Nested depth up to 10+ layers supported
- [ ] Visual ownership diagram generation
- [ ] Must reach ultimate natural persons showing % holdings/control
- [ ] Required for multi-layered corporate clients

### US-2.6: Client Search & Filtering `P0`
**As a** team member **I want to** search and filter clients across multiple dimensions **so that** I can quickly find who I need.

**Acceptance Criteria:**
- [ ] Search by: name, ID, type, jurisdiction, risk tier, status
- [ ] Filter combinations (AND logic)
- [ ] Sortable columns
- [ ] Pagination with configurable page size
- [ ] Quick filters for common views (e.g., "High Risk Active")

### US-2.7: Client Reactivation `P1`
**As a** Master Admin **I want to** reactivate an EXITED client **so that** returning clients don't need a completely new record.

**Acceptance Criteria:**
- [ ] Transition EXITED → REACTIVATION allowed with audit trail
- [ ] If relationship is materially different, new record created instead
- [ ] Reactivation requires re-screening and risk reassessment
- [ ] Previous history preserved

### US-2.8: Identity Document Expiry Monitoring `P0`
**As a** compliance officer **I want** automated alerts when identity documents are expiring **so that** no client file goes stale.

**Acceptance Criteria:**
- [ ] Alerts at 60, 30, 7 days before expiry
- [ ] Auto-transition to UNDER_REVIEW if expired beyond SLA
- [ ] Escalation to onboarding team for high-risk clients
- [ ] Dashboard widget showing upcoming expirations

### US-2.9: SharePoint/OneDrive Export `P1`
**As a** team member **I want** to export client document folders to a structured format **so that** documents can be shared via SharePoint/OneDrive.

**Acceptance Criteria:**
- [ ] Generate structured folder hierarchy per client
- [ ] Export as ZIP with organized subfolders (by category)
- [ ] Include document index/manifest file
- [ ] Bulk export for multiple clients

---

## EPIC 3: AML & Onboarding

> The highest-impact context — 70% time saving target. Most complex domain invariants.

### US-3.1: Onboarding Case Lifecycle `P0`
**As a** team member **I want to** manage onboarding through defined stages **so that** no step is missed and every decision is documented.

**Acceptance Criteria:**
- [ ] Stages: Client Intake → Screening → Risk Scoring → Due Diligence → Approval
- [ ] File Completion checkpoint embedded before Approval
- [ ] Entry/exit criteria enforced per stage
- [ ] Any internal staff can create an OnboardingCase
- [ ] Default: one active case per client (allow two with justification)
- [ ] SLA: 5-10 business days, flag overdue at 10 days
- [ ] Stage skipping allowed for low-risk repeat clients (screening + score still required)

### US-3.2: Client Intake & Document Requests `P0`
**As a** analyst **I want to** initiate structured intake with automatic document requests **so that** I collect everything needed upfront.

**Acceptance Criteria:**
- [ ] Structured intake form per client type
- [ ] Auto-generate document checklist based on client type and jurisdiction
- [ ] Smart reminders for missing documents
- [ ] Track document receipt status
- [ ] Corporate clients: trade licence, MoA/AoA, share register, board resolution, ownership chart, proof of address, financials, list of directors/shareholders
- [ ] Individual clients: passport (mandatory), Emirates ID, residence visa, other formal ID

### US-3.3: Risk Scoring Engine `P0`
**As a** compliance officer **I want** algorithmic risk scoring based on our existing matrix **so that** every client gets a consistent, documented risk rating.

**Acceptance Criteria:**
- [ ] Risk factors: jurisdiction, product/service, structure complexity, SoW/SoF complexity, PEP, adverse media, sector, transaction size, delivery channel, client type
- [ ] Output: Low / Medium / High / Unacceptable
- [ ] Different scoring models per ClientType (Individual vs Corporate vs Fund)
- [ ] Full scoring rationale documented
- [ ] AI agent drafts score, human reviews and confirms (Phase 1)
- [ ] Recalculate on: onboarding, periodic review, ownership change, sanctions trigger, material business change

### US-3.4: Screening Hub `P0`
**As a** analyst **I want to** run and manage screening checks from a unified console **so that** all screening results are in one place.

**Acceptance Criteria:**
- [ ] Phase 1: manual upload of screening results (CSV/PDF)
- [ ] Phase 2: API integration with World-Check / ComplyAdvantage
- [ ] Mandatory lists: Local UAE, UN, HMT (UK), OFAC (USA), EU sanctions + PEP
- [ ] Adverse media screening
- [ ] Match threshold: ≥85% or probable/high-confidence match → human review
- [ ] Unified results view per client/person

### US-3.5: Screening Hit Management `P0`
**As a** analyst **I want to** triage screening hits, document false positives, and escalate true matches **so that** every hit is properly handled.

**Acceptance Criteria:**
- [ ] Hit dispositions: FALSE_POSITIVE, TRUE_MATCH, PENDING_REVIEW
- [ ] False positive: Analyst drafts rationale (min 3-5 sentences with identifiers + evidence), CO/MLRO signs off
- [ ] Sanctions hits: always require founder sign-off
- [ ] True match: stop onboarding → Analyst → CO/MLRO (same day)
- [ ] Notify and require manual action (no auto-block)
- [ ] AI agent triages in Phase 2 (human confirmation always in Phase 1)

### US-3.6: Due Diligence Workflow `P0`
**As a** analyst **I want to** execute CDD/EDD workflows with structured checklists **so that** due diligence is complete and consistent.

**Acceptance Criteria:**
- [ ] Standard CDD (MEDIUM): IDs, corporate docs, ownership chart, screening, SoW/SoF basic, purpose, approvals
- [ ] Enhanced DD (HIGH/VERY_HIGH): enhanced SoW/SoF, deeper adverse media, independent verification, senior approval, enhanced monitoring plan
- [ ] Document checklist auto-generated per risk tier
- [ ] Track completion per checklist item
- [ ] Video KYC/in-person only when risk triggers demand it

### US-3.7: Source of Wealth/Funds Documentation `P0`
**As a** analyst **I want to** capture and analyze SoW/SoF evidence **so that** the origin of client funds is documented.

**Acceptance Criteria:**
- [ ] Minimum: narrative + supporting evidence (bank statements, payslips, sale agreements, audited FS)
- [ ] Requirements vary by client type and risk tier
- [ ] AI agent can draft SoW/SoF narrative summaries (Phase 2)
- [ ] Gap analysis and follow-up question suggestions

### US-3.8: Onboarding Approval Workflow `P0`
**As a** compliance officer **I want** risk-tiered approval routing **so that** high-risk clients get appropriate sign-off.

**Acceptance Criteria:**
- [ ] Low/Medium: CO or MLRO approves
- [ ] High: requires SE/director sign-off
- [ ] Any team member can approve onboarding-to-active transition
- [ ] Approval logged with approver identity and timestamp
- [ ] Rejection returns case to appropriate stage with reason

### US-3.9: Re-Screening Calendar `P0`
**As a** MLRO **I want** daily automated re-screening **so that** existing clients are continuously monitored against sanctions/PEP lists.

**Acceptance Criteria:**
- [ ] Daily screening runs automatically
- [ ] Re-screening covers all active clients
- [ ] New hits flagged and routed to analysts
- [ ] Assignment tracking per re-screening hit
- [ ] Dashboard showing re-screening status and pending hits

### US-3.10: Risk Score Override `P0`
**As a** compliance officer **I want to** override a machine-generated risk score with documented rationale **so that** professional judgment can adjust algorithmic outputs.

**Acceptance Criteria:**
- [ ] Override by onboarding team with CO verification
- [ ] HIGH+ overrides require dual sign-off + documented rationale
- [ ] Override flag and rationale stored in risk score record
- [ ] Audit trail captures original score, new score, and rationale
- [ ] RBA (Risk-Based Assessment) to downgrade PEP risk if justified

### US-3.11: Professional Client Assessment `P0`
**As a** compliance officer **I want to** assess professional client status for regulated clients **so that** classification requirements are met.

**Acceptance Criteria:**
- [ ] Professional client classification form (ADGM/DFSA specific)
- [ ] Suitability assessment completion
- [ ] Classification stored in client record
- [ ] Market Counterparty notice generation (ADGM/DFSA)

### US-3.12: Auto-Population & Form Filler `P1`
**As a** analyst **I want** AI-powered extraction from uploaded documents **so that** internal DD forms are auto-filled without manual re-entry.

**Acceptance Criteria:**
- [ ] Extract data from passports, trade licences, bank statements
- [ ] Auto-fill internal DD forms
- [ ] Human review before persistence
- [ ] Highlight extracted vs manually entered fields

### US-3.13: Onboarding to Active Client Transition `P0`
**As a** compliance officer **I want** a formal handoff when onboarding is approved **so that** the client record transitions to ACTIVE with all required setup.

**Acceptance Criteria:**
- [ ] OnboardingApproved event triggers Client.status → ACTIVE
- [ ] Initial compliance registers created for the client
- [ ] Re-screening schedule activated based on risk tier
- [ ] Assigned team notified of new active client
- [ ] All onboarding documents linked to client record
- [ ] Audit trail records the full transition chain

### US-3.14: Risk Scoring Configuration (Abstract) `P0`
**As a** admin **I want** configurable risk scoring factors and weights **so that** the scoring engine can be adapted when the CRA matrix is provided.

**Acceptance Criteria:**
- [ ] Pluggable risk factor architecture (add/remove factors without code changes)
- [ ] Factor weights configurable per tenant
- [ ] Score threshold configuration per tenant (Low/Medium/High/Unacceptable boundaries)
- [ ] Fallback to default configuration if tenant-specific not set
- [ ] Risk scoring works with manual factor entry until CRA Excel is imported

---

## EPIC 4: Compliance Framework

> Policies, registers, incidents, training, board reporting — the backbone of ongoing compliance.

### US-4.1: Policy Library `P0`
**As a** compliance officer **I want** a version-controlled policy library **so that** all policies are centrally managed with full amendment history.

**Acceptance Criteria:**
- [ ] Policy types: AML/CFT/CPF, Sanctions, CDD/EDD, Conflicts, Conduct, Complaints, Data Protection, Record Keeping, Outsourcing, Risk Management, Whistleblowing, Gifts, BCP/IT, Training, and more
- [ ] Version numbering: major.minor (e.g., 1.0, 1.1)
- [ ] Full amendment history with tracked changes
- [ ] Review triggers: regulatory change, annual review, audit/CMP findings, material business change
- [ ] Automated review reminders

### US-4.2: Policy Approval Workflow `P0`
**As a** compliance officer **I want** structured approval workflows for policies **so that** every policy change goes through proper governance.

**Acceptance Criteria:**
- [ ] Workflow: Draft → Internal Review → Approval → Publication
- [ ] Review by CO/MLRO, approval by Board
- [ ] Dual sign-off for core AML policy
- [ ] No minimum consultation period (internal review time-boxed)
- [ ] AI agent can draft policies (human triggers all transitions in Phase 1)

### US-4.3: Register Management `P0`
**As a** compliance officer **I want** structured, searchable compliance registers **so that** all events are properly logged and retrievable.

**Acceptance Criteria:**
- [ ] Register types: Breach, Complaints, Gifts/Hospitality, Conflicts of Interest, Screening Log, Regulatory Correspondence, Training, Outsourcing, Risk, Client, CMP Findings, SAR, Marketing, Directors, Shareholders, Employees, UBO, Personal Account Dealing, Outside Interest
- [ ] Mandatory fields per register type
- [ ] Auto-update from case events (e.g., screening hit → screening log)
- [ ] Full change history and export capabilities
- [ ] Admins: full access; Analysts: write operational registers; Sensitive outputs: read-only for most
- [ ] reg_submission flag for registers subject to regulatory submission
- [ ] Export as CSV/PDF

### US-4.4: Incident & Breach Management `P0`
**As a** compliance officer **I want** structured incident workflows with escalation **so that** breaches are captured, assessed, and remediated consistently.

**Acceptance Criteria:**
- [ ] Capture, assess, remediate, close workflow
- [ ] Severity levels: Low, Medium, High, Critical
- [ ] CRITICAL: notification decision before close, reportability assessed by founders
- [ ] Regulatory notification SLA: 24-72 hours after determination
- [ ] Critical escalation: immediate CO/MLRO → board/regulator as required
- [ ] Incidents can be re-opened if new facts arise
- [ ] Escalation rules configurable per severity

### US-4.5: Training Management `P0`
**As a** compliance officer **I want to** track training plans, attendance, and attestations **so that** regulatory training requirements are demonstrably met.

**Acceptance Criteria:**
- [ ] Training records: type/module, completion date, attestation, score, source/trainer, next due date
- [ ] Automated reminders for upcoming training
- [ ] Training register with search and export
- [ ] Attestation capture and storage

### US-4.6: Board Pack Generation `P1`
**As a** founder **I want** auto-generated compliance sections for board packs **so that** board reporting is consistent and fast.

**Acceptance Criteria:**
- [ ] Monthly/quarterly compliance report generation
- [ ] KPIs: onboarding time/volume, high-risk clients, rescreen due/overdue, CMP completion, findings by severity, overdue actions, incidents/complaints, policy reviews due
- [ ] Risk heatmaps
- [ ] Open findings and actions summary
- [ ] Regulatory changes pending review
- [ ] Output: PPTX for editable, PDF for final
- [ ] AI agent assembles (Phase 2)

### US-4.7: Audit & Regulator Visit Prep `P2`
**As a** compliance officer **I want** one-click evidence pack generation **so that** audit and regulatory visits can be prepared quickly.

**Acceptance Criteria:**
- [ ] Pull evidence from all platform data
- [ ] Structured evidence folders
- [ ] Standard index/table of contents
- [ ] Export as ZIP or merged PDF

---

## EPIC 5: Regulatory Intelligence

> Continuous monitoring of regulatory developments with automated analysis and client communications.

### US-5.1: Regulatory Radar (Basic Weekly Scan) `P0`
**As a** compliance officer **I want** automated weekly scans of regulatory sources **so that** no relevant change goes unnoticed.

**Acceptance Criteria:**
- [ ] Sources: DFSA, FSRA, UAE CB, VARA, DIFC, ADGM
- [ ] International baseline: FATF updates, key sanctions list updates
- [ ] Weekly fixed schedule scan + ad-hoc manual scan on demand
- [ ] Structured change log with categorization
- [ ] Manual approval before any publication (no auto-publish)
- [ ] Distribution: Isahaq + Sagheer by default, configurable per topic

### US-5.2: Regulatory Radar (Full) `P2`
**As a** compliance officer **I want** automated weekly scans of regulatory sources **so that** no relevant change goes unnoticed.

**Acceptance Criteria:**
- [ ] Sources: DFSA, FSRA, UAE CB, VARA, DIFC, ADGM
- [ ] International: FATF updates and key sanctions list updates
- [ ] Change types: rulebook/law changes, guidance, enforcement actions, consultation papers, FAQ updates
- [ ] Weekly scan schedule (fixed weekly run)
- [ ] Ad-hoc/manual scan on demand
- [ ] Structured change log with categorization

### US-5.3: Impact Assessment Workflow `P2`
**As a** founder **I want** impact assessments for regulatory changes **so that** I know which changes affect which clients and what actions are needed.

**Acceptance Criteria:**
- [ ] AI agent auto-drafts impact assessment
- [ ] Human review and sign-off (Isahaq/Sagheer)
- [ ] SLA: 3 business days (normal), 24-48 hours (HIGH impact)
- [ ] HIGH impact auto-triggers: policy review task, training item, client alert draft, CMP test update, board pack flag

### US-5.4: Client Alert Management `P2`
**As a** founder **I want** approval-gated client alerts **so that** clients are informed of regulatory changes that affect them.

**Acceptance Criteria:**
- [ ] Draft alert from regulatory change
- [ ] Approval: Isahaq and/or Sagheer (dual for HIGH impact/sensitive)
- [ ] Distribution: email (primary), platform notification (optional)
- [ ] Store copy in platform
- [ ] Retention: minimum 6 years

### US-5.5: Regulatory Change Log `P2`
**As a** team member **I want** a searchable database of all regulatory changes **so that** I can track and reference any change.

**Acceptance Criteria:**
- [ ] Tags by topic and applicability
- [ ] Search by regulator, date, topic, impact level
- [ ] Link to original source publication
- [ ] Downstream triggers visible (policy review, training, CMP)

### US-5.6: Weekly Digest `P2`
**As a** founder **I want** a curated weekly digest of regulatory developments **so that** I stay informed without manual monitoring.

**Acceptance Criteria:**
- [ ] Distribution: Isahaq + Sagheer by default, configurable per topic/regulator
- [ ] Categorized by regulator and topic
- [ ] Delivered via email

### US-5.7: Regulatory Q&A Copilot `P2`
**As a** compliance officer **I want** to ask natural-language questions and get cited answers from the regulatory corpus **so that** I can quickly find applicable rules.

**Acceptance Criteria:**
- [ ] Corpus: DFSA, ADGM, Central Bank, DIFC rules/laws; FATF recommendations
- [ ] Response format: direct answer, exact citations, practical application, limitations
- [ ] No hallucinations — every reference verified, backed by rationale
- [ ] Confidence indicators
- [ ] If can't answer with confidence: search online but always provide source and score
- [ ] Internal-only (Phase 1)
- [ ] Queries and responses logged to audit trail
- [ ] Corpus updates: admin-only, versioned, change-noted

---

## EPIC 6: Authorisations

> Structured workflow management for regulatory licensing applications (6-9 month lifecycle).

### US-6.1: Authorisation Case Management `P1`
**As a** compliance officer **I want** stage-gated application workflows **so that** licensing applications progress through defined quality gates.

**Acceptance Criteria:**
- [ ] Stages: Engagement → Conflicts Check → Doc Collection → Drafting → QA Review → Fee Payment → Submission → RFI → Approval
- [ ] Entry/exit criteria per stage
- [ ] Deadline tracking and automatic assignment
- [ ] Escalation for delays
- [ ] Pipeline dashboard showing all active cases
- [ ] Timeline: 6-9 months typical

### US-6.2: Licence Categories `P1`
**As a** team member **I want** pre-configured licence type templates **so that** each application type has the correct requirements.

**Acceptance Criteria:**
- [ ] DFSA/FSRA categories: Cat 1 (Deposit Taking), Cat 2 (Credit/Dealing), Cat 3A (Brokerage), Cat 3C (Asset Management - Fund/DPM), Cat 3D/3C (Money Services), Cat 4 (Advisory/Arranging)
- [ ] VARA: VASP licence types (Phase 2)
- [ ] Category-specific document checklists and requirements

### US-6.3: Document Pack Generator `P1`
**As a** team member **I want** automated submission pack assembly **so that** application documents are consistent and complete.

**Acceptance Criteria:**
- [ ] Templates: Business Plan, Compliance Arrangements, Fit & Proper packs
- [ ] Auto-populate from platform data
- [ ] Version control per document
- [ ] Output: DOCX editable, PDF final

### US-6.4: RFI Tracking & Response `P1`
**As a** case lead **I want** structured RFI tracking **so that** every regulator query is responded to on time.

**Acceptance Criteria:**
- [ ] Track multiple concurrent RFIs per application (1-3 rounds typical)
- [ ] Each RFI has its own SLA and evidence
- [ ] Target response: within one week
- [ ] Collaborative drafting: case lead + client review
- [ ] AI assistant drafts responses using knowledge base and case data (Phase 2)
- [ ] Precedent library: tagged by regulator, licence type, topic, outcome

### US-6.5: Post-Licensing Handoff `P2`
**As a** compliance officer **I want** automatic handoff to BAU compliance once a licence is granted **so that** ongoing obligations are immediately tracked.

**Acceptance Criteria:**
- [ ] Auto-create: compliance calendar, CMP schedule, registers, training plan, regulatory returns schedule, rescreening schedule
- [ ] Post-licence obligations: governance confirmation, policies/CMP, regulatory returns, capital/insurance, client disclosures, register setup

### US-6.6: QA Review Workflow `P1`
**As a** senior team member **I want** internal QA before submission **so that** application quality is verified.

**Acceptance Criteria:**
- [ ] QA checklist (from QAP) required before submission stage
- [ ] QA reviewer assigned per case
- [ ] QA sign-off required to proceed
- [ ] Issues logged and tracked

---

## EPIC 7: Entity Management (SPVs & Foundations)

> Entity lifecycle management — incorporation through ongoing obligations, renewals, and compliance monitoring.

### US-7.1: Entity Register `P1`
**As a** operations staff **I want** a structured register of all managed entities **so that** every SPV, foundation, and fund is tracked centrally.

**Acceptance Criteria:**
- [ ] Entity types: SPVs, Foundations, Funds/Fund Vehicles, Internal Group Entities, Client Companies
- [ ] Jurisdictions: DIFC, ADGM, offshore
- [ ] Required data: registered office, directors, beneficial owners, service providers, key dates
- [ ] Currently 17 entities, projected 100+
- [ ] Nested ownership structures (parent_entity_id)

### US-7.2: Obligation Tracking `P1`
**As a** operations staff **I want** automated obligation tracking with reminders **so that** no renewal or filing deadline is missed.

**Acceptance Criteria:**
- [ ] Obligation types: annual renewal, registered agent fees, annual returns, UBO updates, director renewals, economic substance filings, FATCA/CRS, licence renewals, board resolutions, service provider renewals
- [ ] Reminder cadence: 60/30/14/7/1 days before due; overdue at +3/+7/+14 days
- [ ] Reminders to: case owner/RM + entity director
- [ ] Escalation: +3 days → owner; +7 → Parastu & Isahaq; +14 → board pack/management report
- [ ] Regulatory filing flag per obligation

### US-7.3: Entity Document Vault `P1`
**As a** operations staff **I want** a document vault per entity **so that** all corporate documents are centralized with version control.

**Acceptance Criteria:**
- [ ] Document categories: incorporation, constitutional docs, registers, resolutions/minutes, licences, filings, ownership charts, KYC/AML, service provider agreements, bank documents, key contracts, ad-hoc documents
- [ ] Full version history
- [ ] Upload/delete by authorized staff with audit log

### US-7.4: Entity Compliance Dashboard `P1`
**As a** founder **I want** a per-entity compliance health view **so that** I can see at a glance which entities need attention.

**Acceptance Criteria:**
- [ ] Traffic-light indicators per entity
- [ ] Renewals due, filings status, missing documents
- [ ] Service provider details
- [ ] Compliance health score

### US-7.5: Renewal & Evidence Pack Generation `P2`
**As a** operations staff **I want** automated renewal packs and bank onboarding evidence packs **so that** recurring outputs are fast and consistent.

**Acceptance Criteria:**
- [ ] Pre-populated renewal documents
- [ ] Bank onboarding evidence pack assembly
- [ ] Corporate registers and resolution generation

---

## EPIC 8: CMP Monitoring Programme

> Digitization of the existing Compliance Monitoring Programme (~175 tests across 15+ domains).

### US-8.1: Test Library Management `P2`
**As a** compliance officer **I want** a structured, searchable test catalogue **so that** our monitoring programme is organized and versioned.

**Acceptance Criteria:**
- [ ] Import from existing CMP spreadsheet
- [ ] Domains: AML/CFT, Conduct, Fund Admin, Corporate Governance, HR, Marketing, Record Keeping, Regulatory, Risk Management, Data Protection, Money Services, Fund/Asset Management, Custody, and more
- [ ] Configurable per company type
- [ ] ~175 tests with sub-steps/sampling within working papers
- [ ] Version tracking

### US-8.2: Test Scheduling `P2`
**As a** compliance officer **I want** calendar-based and event-triggered test scheduling **so that** the monitoring programme runs on time.

**Acceptance Criteria:**
- [ ] Frequency: yearly, half-yearly, quarterly, monthly
- [ ] Event-triggered tests (regulatory change, material incident)
- [ ] Default assignment: system assigns to Risk Owner/function owner
- [ ] Compliance Manager/Admin can override and reassign
- [ ] Rescheduling with reason code; beyond one cycle needs Admin approval
- [ ] Overdue escalation: +3 owner → +7 nominated CO → +14 Isahaq → continuing every 14 days

### US-8.3: Working Paper Execution `P2`
**As a** analyst **I want** structured working papers for each test **so that** test execution is consistent and evidence is linked.

**Acceptance Criteria:**
- [ ] Mandatory structure: scope/objective, rule reference, methodology, sample selection, evidence list, testing performed, findings, conclusion, action points, sign-off (tester + reviewer) + date
- [ ] Sampling: risk-based default, with random/judgmental options; method and rationale documented
- [ ] Evidence: screenshots, PDFs, spreadsheets, emails, system exports, meeting minutes, logs
- [ ] Evidence stored with metadata (source/date/owner)
- [ ] Tester completes; senior reviewer approves
- [ ] HIGH/Critical findings: final sign-off by CO or SEO/Board

### US-8.4: Finding Management `P2`
**As a** compliance officer **I want** structured finding capture with severity ratings **so that** issues are tracked and remediated.

**Acceptance Criteria:**
- [ ] Severity: Observation / Low / Medium / High / Critical
- [ ] Board reporting: High and Critical always; Medium if thematic/repeated
- [ ] Root cause analysis
- [ ] Remediation tracking with action owners and due dates
- [ ] Re-opening allowed if remediation insufficient

### US-8.5: Remediation Tracking `P2`
**As a** compliance officer **I want** SLA-driven remediation workflows **so that** findings are closed on time.

**Acceptance Criteria:**
- [ ] SLAs: Critical — plan 7 days, close 30 days; High — 30 days; Medium — 60 days; Low — 90 days; Observation — 120 days (configurable)
- [ ] Action owners, due dates, escalation rules
- [ ] Closure evidence required
- [ ] Overdue actions visible in board pack

### US-8.6: CMP Reporting `P2`
**As a** compliance officer **I want** automated CMP reports **so that** monitoring programme status is always visible.

**Acceptance Criteria:**
- [ ] Completion rate (tests completed vs scheduled)
- [ ] Findings by severity and theme
- [ ] Overdue actions summary
- [ ] Annual CMP report for the board
- [ ] Downstream triggers to policy updates, training, regulatory reporting

---

## EPIC 9: Document Factory

> Template-driven document generation — the platform's production engine.

### US-9.1: Template Management `P0`
**As a** admin **I want** to manage document templates **so that** all generated documents follow current standards.

**Acceptance Criteria:**
- [ ] Template types: onboarding/KYC checklist, internal DD form, risk assessment matrix, screening memo, engagement letter, authorisation checklists, CMP working paper, board report, policies/manuals
- [ ] DOCX/PPTX templates with merge fields
- [ ] Version control on templates
- [ ] Admin-only template updates

### US-9.2: Document Auto-Population `P0`
**As a** team member **I want** documents auto-filled from platform data **so that** there's zero manual re-entry.

**Acceptance Criteria:**
- [ ] Merge fields populated from client, entity, case, and user data
- [ ] Preview before generation
- [ ] Multiple output formats: DOCX (editable), PDF (final), PPTX (presentations)

### US-9.3: Document Versioning & Retention `P0`
**As a** compliance officer **I want** full document lifecycle management **so that** regulatory retention requirements are met.

**Acceptance Criteria:**
- [ ] All versions retained
- [ ] Minimum 6-year retention
- [ ] Archive option available
- [ ] Delete button only after 6 years elapsed
- [ ] Change log per document

### US-9.4: Evidence Pack Generation `P1`
**As a** team member **I want** assembled evidence packs **so that** audit and onboarding packages are ready quickly.

**Acceptance Criteria:**
- [ ] Format: ZIP or merged PDF (decided per case)
- [ ] Structured folder structure
- [ ] Index/table of contents
- [ ] Pull from all platform data sources

### US-9.5: Digital Signature Integration `P2`
**As a** team member **I want** digital signature support **so that** key documents can be signed within the platform.

**Acceptance Criteria:**
- [ ] Engagement letters: client + team signature
- [ ] Internal approvals: Isahaq/Sagheer
- [ ] Board resolutions: entity directors/beneficiaries
- [ ] Working paper sign-offs: e-sign within platform

---

## EPIC 10: Workflow Engine (Cross-Cutting)

> Shared workflow infrastructure used by all bounded contexts.

### US-10.1: Task Management `P0`
**As a** team member **I want** assigned tasks with deadlines **so that** I know what I need to do and when.

**Acceptance Criteria:**
- [ ] Tasks linked to cases, entities, policies, tests
- [ ] Assignment to users
- [ ] Due dates and priority
- [ ] Status tracking (open, in_progress, completed, overdue)
- [ ] Team dashboard: "My Tasks"

### US-10.2: Approval Workflows `P0`
**As a** team member **I want** configurable approval workflows with stage gates **so that** nothing progresses without proper sign-off.

**Acceptance Criteria:**
- [ ] Configurable per vertical/context
- [ ] Single or dual approval
- [ ] Approval routing by risk tier
- [ ] Rejection with reason returns to appropriate stage
- [ ] Full audit trail

### US-10.3: Reminder & Escalation Engine `P0`
**As a** compliance officer **I want** automated reminders and escalations **so that** nothing falls through the cracks.

**Acceptance Criteria:**
- [ ] Configurable reminder cadence (60/30/14/7/1 days)
- [ ] Overdue escalation path (owner → manager → founder)
- [ ] Email delivery
- [ ] In-app notification (Phase 2)
- [ ] Escalation logged in audit trail

### US-10.4: SLA Tracking `P0`
**As a** founder **I want** SLA tracking across all workflows **so that** I can identify bottlenecks.

**Acceptance Criteria:**
- [ ] SLAs configurable per workflow type
- [ ] Dashboard showing SLA compliance %
- [ ] Overdue items highlighted
- [ ] Historical SLA performance

---

## EPIC 11: AI Agent Layer

> Specialized AI agents per vertical — all in "draft + human confirm" mode for Phase 1.

### US-11.1: Agent Autonomy Framework `P0`
**As a** admin **I want** a consistent agent framework with human-in-the-loop **so that** AI assists but never acts without oversight.

**Acceptance Criteria:**
- [ ] All agent actions require human approval (Phase 1)
- [ ] Agent actions labeled in audit trail (agent identity, confidence)
- [ ] Agent accuracy metrics tracked over time
- [ ] Agent outputs are "drafts" until human confirms

### US-11.2: Risk Scoring Agent `P0`
**As a** analyst **I want** AI-drafted risk scores **so that** scoring is faster while maintaining human oversight.

**Acceptance Criteria:**
- [ ] Agent applies risk matrix algorithmically
- [ ] Outputs structured score with full rationale
- [ ] Human reviews and confirms before persistence

### US-11.3: Screening Triage Agent `P2`
**As a** analyst **I want** AI-powered screening hit triage **so that** obvious false positives are flagged automatically.

**Acceptance Criteria:**
- [ ] Agent consumes screening results, triages hits
- [ ] Drafts false-positive rationale for likely non-matches
- [ ] Escalates genuine matches
- [ ] Human confirmation required for all dispositions (Phase 1)
- [ ] Target: 70%+ reduction in analyst screening time

### US-11.4: Policy Drafter Agent `P2`
**As a** compliance officer **I want** AI-drafted policies **so that** policy creation and updates are faster.

**Acceptance Criteria:**
- [ ] Draft policies reflecting regulatory changes
- [ ] Produce versioned documents with tracked changes
- [ ] Human triggers all workflow transitions

### US-11.5: Board Pack Builder Agent `P2`
**As a** founder **I want** AI-assembled board pack sections **so that** report preparation time is halved.

**Acceptance Criteria:**
- [ ] Aggregate KPIs, findings, regulatory changes, risk heatmaps
- [ ] Structured, presentation-ready format
- [ ] Human review before distribution

### US-11.6: Document Auto-Fill Agent `P1`
**As a** analyst **I want** AI extraction from uploaded documents **so that** form filling is automated.

**Acceptance Criteria:**
- [ ] Extract from passports, trade licences, bank statements
- [ ] Auto-fill internal DD forms
- [ ] Human review before save

---

## EPIC 12: Dashboards & Reporting

> Right view for the right role — strategic oversight to daily task management.

### US-12.1: Founder Dashboard `P0`
**As a** founder **I want** a strategic overview dashboard **so that** I see everything that needs my attention at a glance.

**Acceptance Criteria:**
- [ ] Open cases by type and stage
- [ ] High-risk onboardings
- [ ] Upcoming deadlines (all contexts)
- [ ] CMP status summary
- [ ] Regulatory changes pending review
- [ ] Overdue actions and bottlenecks
- [ ] KPI cards with trends

### US-12.2: Team Dashboard `P0`
**As a** team member **I want** a personal task dashboard **so that** I can focus on my daily execution.

**Acceptance Criteria:**
- [ ] My assigned tasks
- [ ] Upcoming deadlines
- [ ] Pending approvals (items I need to approve)
- [ ] Evidence needed (items waiting for my input)
- [ ] Case status updates

### US-12.3: Entity Dashboard `P1`
**As a** operations staff **I want** a per-entity compliance view **so that** I can see each entity's health.

**Acceptance Criteria:**
- [ ] Renewals due and filings status
- [ ] Missing documents
- [ ] Service provider details
- [ ] Traffic-light compliance indicators

### US-12.4: Global Search `P0`
**As a** team member **I want** to search across clients, cases, policies, entities, and documents **so that** I can find anything quickly.

**Acceptance Criteria:**
- [ ] Unified search bar
- [ ] Results categorized by type
- [ ] Full-text search across document content (Phase 2)
- [ ] Recent searches

---

## EPIC 13: Data Migration & Go-Live

> Import existing data and transition from legacy processes.

### US-13.1: Client Data Import `P0`
**As a** admin **I want to** import existing client records from Excel **so that** the platform starts with current data.

**Acceptance Criteria:**
- [ ] Import from Excel/CSV
- [ ] Field mapping UI
- [ ] Validation and error reporting
- [ ] Dry-run mode before commit
- [ ] ~30 current clients + people/UBO records

### US-13.2: CMP Spreadsheet Import `P2`
**As a** compliance officer **I want to** import the existing CMP spreadsheet **so that** the test library is pre-populated.

**Acceptance Criteria:**
- [ ] Import ~175 tests across 15+ domains
- [ ] Map domains, frequencies, and risk owners
- [ ] Preserve test descriptions and methodology

### US-13.3: Template & Policy Import `P0`
**As a** admin **I want to** import existing templates and policies **so that** the document factory is ready from day one.

**Acceptance Criteria:**
- [ ] Import DOCX/XLSX/PPTX templates
- [ ] Categorize and tag
- [ ] Set initial version

### US-13.4: Parallel Run Support `P0`
**As a** founder **I want** a 2-4 week parallel run **so that** we can verify the platform before full cutover.

**Acceptance Criteria:**
- [ ] New platform runs alongside old processes
- [ ] Soft launch: Isahaq/Sagheer + 1-2 support users
- [ ] Export all data from new platform (rollback capability)
- [ ] Feedback loop during first month
- [ ] 2 training sessions (admin + users) + quick-start SOPs

---

## EPIC 14: Light Workflows (Phase 1 Tracking)

> Simple CRUD tracking for Authorisations and Entity Management. Not full bounded contexts — just structured tracking with dates and audit trail. Deepen in Phase 2.

### US-14.1: Authorisation Case Tracking (Light) `P0`
**As a** team member **I want** to track regulatory applications through stages **so that** all active cases are visible in one place.

**Acceptance Criteria:**
- [ ] Create/edit authorisation case records
- [ ] Track stages: Engagement → Conflicts Check → Doc Collection → Drafting → QA → Fee Payment → Submission → RFI → Approval
- [ ] Record: applicant name, regulator, licence category, assigned team, key dates
- [ ] Kanban board view (from wireframe 05-auth-pipeline)
- [ ] Stage transition with audit trail
- [ ] Basic notes and activity log per case

### US-14.2: Entity Register Tracking (Light) `P0`
**As a** operations staff **I want** to track all managed entities with key dates **so that** renewals and obligations are not missed.

**Acceptance Criteria:**
- [ ] Create/edit entity records (SPV, Foundation, Fund, Holding, Internal)
- [ ] Track: name, type, jurisdiction, registration date, licence expiry, next renewal
- [ ] Directors and beneficial owners (link to Person records)
- [ ] Key dates with reminder alerts (60/30/14/7/1 days)
- [ ] Traffic-light status based on upcoming/overdue dates
- [ ] List view with filters (from wireframe 20-entity-register)

### US-14.3: Post-Licence Handoff (Light) `P0`
**As a** compliance officer **I want** automatic setup when a licence is granted **so that** ongoing compliance infrastructure is created immediately.

**Acceptance Criteria:**
- [ ] When authorisation case reaches APPROVED, trigger handoff
- [ ] Create: compliance calendar entries, register entries, training plan items, rescreening schedule
- [ ] Create tasks for: governance confirmation, policy finalization, initial regulatory returns
- [ ] Handoff completion tracked and auditable

---

## Story Count Summary

| Epic | P0 | P1 | P2 | Total |
|------|----|----|-----|-------|
| 1. Platform Foundation | 4 | 1 | 0 | 5 |
| 2. CRM / Client | 6 | 2 | 0 | 8 |
| 3. AML & Onboarding | 13 | 1 | 0 | 14 |
| 4. Compliance Framework | 5 | 1 | 1 | 7 |
| 5. Regulatory Intelligence | 1 | 0 | 6 | 7 |
| 6. Authorisations | 0 | 5 | 1 | 6 |
| 7. Entity Management | 0 | 4 | 1 | 5 |
| 8. CMP Monitoring | 0 | 0 | 6 | 6 |
| 9. Document Factory | 3 | 1 | 1 | 5 |
| 10. Workflow Engine | 4 | 0 | 0 | 4 |
| 11. AI Agent Layer | 2 | 1 | 3 | 6 |
| 12. Dashboards | 3 | 1 | 0 | 4 |
| 13. Data Migration | 3 | 0 | 1 | 4 |
| 14. Light Workflows | 3 | 0 | 0 | 3 |
| **TOTAL** | **47** | **17** | **20** | **84** |

> Phase 1 (P0): 47 stories — the complete Phase 1 as confirmed by client. CRM + AML + Document Factory + Compliance baseline + Regulatory Radar basic + Light Auth/Entity tracking + Agent draft mode.
> Phase 1+ (P1): 17 stories — enhanced features for Phase 1 expansion.
> Phase 2 (P2): 20 stories — Regulatory Intelligence full, CMP full, advanced AI, digital signatures.
