# Compliance Brain - DDD Architecture

> Domain-Driven Design architecture for the Compliance Brain Platform.
> Principles: DDD, SOLID, Dependency Injection, Open/Closed. Pragmatic — never over-complicated.

---

## Architecture Approach: Hybrid DDD

Not all bounded contexts need full DDD ceremony. We use three levels based on domain complexity:

| Level | Structure | When to use | Contexts |
|-------|-----------|------------|----------|
| **Full DDD** | domain/ + application/ + infrastructure/ + api/ | Complex invariants, state machines with many states, pluggable strategies | AML Onboarding |
| **Service Layer** | models.py + schemas.py + service.py + routes.py | Moderate logic, simple state machines, approval workflows | Client, Compliance, Workflow |
| **Simple CRUD** | models.py + schemas.py + routes.py | Pure data management, no business rules beyond validation | Identity, Document Factory, Regulatory (basic), Light Tracking |

**Principle**: Start simple. Promote to the next level only when complexity demands it. A context that starts as CRUD can become Service Layer when business rules emerge, and Service Layer can become Full DDD when invariants multiply.

**Key restrictions by level:**
- **Full DDD**: Repository interfaces (ABCs), command/query separation, domain events, mappers
- **Service Layer**: `AsyncSession` used directly in `service.py`. No repository interfaces. No commands/queries classes (just service methods).
- **Simple CRUD**: `AsyncSession` used directly in `routes.py`. No service layer. No repository.

**Why hybrid?** Full DDD for everything adds ~40% more code for ~60% of contexts that are essentially CRUD. With < 10 users and 4 devs (1 junior), the overhead slows delivery without adding safety. Full DDD is reserved for AML where regulatory invariants ARE the product value.

---

## Context Map

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        COMPLIANCE BRAIN PLATFORM                        │
│                                                                          │
│  ┌────────────────────┐         ┌──────────────┐                        │
│  │  Identity & Access  │         │  Audit Trail  │                        │
│  │  (IAM + Tenant)     │         │  (Shared Knl) │                        │
│  │  Context 1          │         │               │                        │
│  └─────────┬──────────┘         └───────┬───────┘                        │
│            │ UserId + TenantId flow down │                                │
│  ┌─────────┼────────────────────────────┼─────────────────────────┐     │
│  │         ▼            PHASE 1 CORE    ▼                          │     │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │     │
│  │  │  2. Client    │  │  3. AML      │  │  4. Document  │         │     │
│  │  │  Management   │──►  Onboarding  │──►  Factory      │         │     │
│  │  │  (CRM Core)   │  │              │  │               │         │     │
│  │  └──────┬────────┘  └──────┬───────┘  └───────────────┘         │     │
│  │         │                  │                                     │     │
│  │  ┌──────▼────────┐  ┌─────▼────────┐  ┌──────────────┐         │     │
│  │  │  5. Compliance │  │  6. Workflow  │  │  7. Reg Intel │         │     │
│  │  │  Framework     │  │  Engine       │  │  (basic scan) │         │     │
│  │  │  (baseline)    │  │  (Shared Knl) │  │               │         │     │
│  │  └───────────────┘  └──────────────┘  └───────────────┘         │     │
│  │                                                                  │     │
│  │  PHASE 1 LIGHT (CRUD tracking only)                              │     │
│  │  ┌─────────────────────┐  ┌─────────────────────┐               │     │
│  │  │  Auth Cases (light)  │  │  Entities (light)    │               │     │
│  │  │  → Full BC Phase 2   │  │  → Full BC Phase 2   │               │     │
│  │  └─────────────────────┘  └─────────────────────┘               │     │
│  └──────────────────────────────────────────────────────────────────┘     │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │  PHASE 2+ (full bounded contexts)                                │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │    │
│  │  │ 8. Authorise  │  │ 9. Entity    │  │ 10. CMP      │          │    │
│  │  │ (full)        │  │ Mgmt (full)  │  │ Monitoring   │          │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘          │    │
│  │  ┌──────────────┐  ┌──────────────┐                             │    │
│  │  │ 7. Reg Intel  │  │ 11. AI Agent │                             │    │
│  │  │ (full)        │  │ Layer (full) │                             │    │
│  │  └──────────────┘  └──────────────┘                             │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │  SUPPORTING SERVICES (not bounded contexts)                      │    │
│  │  ┌──────────────┐  ┌──────────────┐                             │    │
│  │  │ Notification  │  │  Storage     │                             │    │
│  │  │ (email/SMTP)  │  │  (Supabase)  │                             │    │
│  │  └──────────────┘  └──────────────┘                             │    │
│  └──────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────┘
```

## Context Relationships

| Upstream | Downstream | Relationship | Notes |
|----------|-----------|-------------|-------|
| Identity & Access (1) | All contexts | Shared Kernel | UserId, TenantId, Role, Permission available everywhere |
| Audit Trail (11) | All contexts | Shared Kernel | Every command emits audit event |
| Client Management (2) | AML Onboarding (3) | Customer-Supplier | AML references ClientId |
| Client Management (2) | Compliance (4) | Customer-Supplier | Compliance references ClientId |
| Client Management (2) | Authorisations (6) | Customer-Supplier | Auth cases link to clients |
| Client Management (2) | Entity Management (7) | Customer-Supplier | Entities link to clients |
| AML Onboarding (3) | Document Factory (9) | Customer-Supplier | Onboarding triggers doc generation |
| Workflow Engine (10) | All domain contexts | Shared Kernel | Tasks, approvals, reminders |
| Compliance (4) | CMP Monitoring (8) | Customer-Supplier | Findings feed compliance registers |
| Regulatory Intelligence (5) | Compliance (4) | Conformist | Reg changes trigger policy reviews |
| Regulatory Intelligence (5) | CMP Monitoring (8) | Conformist | Reg changes trigger test updates |
| Authorisations (6) | Compliance (4) | Published Language | Post-licensing handoff event |
| AI Agent Layer | All domain contexts | Anti-Corruption Layer | Agents invoke domain commands through ACL |

## Integration Strategy

- **Phase 1**: Synchronous (direct service calls between contexts within same process)
- **Phase 2**: Add async events for key cross-context triggers (message queue if needed later)
- **Anti-Corruption Layers**: AI agents access domain through application services, never directly

---

## Bounded Context 1: Identity & Access (IAM + Tenant)

> **Architecture level: Simple CRUD** — User profiles and tenant config. Authentication delegated to Supabase Auth. RBAC enforced via middleware.

### Aggregates

#### User (Aggregate Root)
```
User
├── user_id: UserId (UUID)
├── supabase_auth_id: str  # link to Supabase Auth user
├── email: Email (VO)
├── full_name: str
├── role: UserRole (enum)
├── tenant_assignments: List[TenantAssignment]
├── is_active: bool
├── last_login: Optional[datetime]
├── created_at: datetime
├── updated_at: datetime
├── version: int
│
├── assign_to_tenant(tenant_id, role_override?) → TenantAssignment
├── lock() → None
├── unlock() → None
└── change_role(new_role, by: UserId) → None
```

> **Note**: Authentication (password, MFA, login attempts) is handled entirely by Supabase Auth. The User aggregate only stores identity and authorization data. `supabase_auth_id` links to the Supabase Auth user record.

**Invariants:**
- Master Admin role cannot be removed from founders (Isahaq, Sagheer)

#### Tenant (Aggregate Root)
```
Tenant
├── tenant_id: TenantId (UUID)
├── name: str
├── type: TenantType (CLIENT_FIRM | INTERNAL_ENTITY)
├── active_regulators: List[RegulatorCode]
├── configuration: TenantConfiguration (VO)
├── status: TenantStatus (ACTIVE | SUSPENDED | ARCHIVED)
├── created_at: datetime
│
├── update_configuration(config) → None
├── activate_regulator(code) → None
├── deactivate_regulator(code) → None
├── suspend() → None
└── reactivate() → None
```

### Value Objects
- `UserId` — UUID wrapper
- `Email` — validated email
- `UserRole` — enum: MASTER_ADMIN, COMPLIANCE_OFFICER, MLRO, ANALYST, OPERATIONS, VIEWER
- `TenantAssignment` — { tenant_id, role_override?, assigned_at }
- `Permission` — { context, action, resource }
- `TenantId` — UUID wrapper
- `RegulatorCode` — enum: DFSA, FSRA, UAE_CB, VARA, DIFC, ADGM
- `TenantConfiguration` — { ubo_threshold_pct (10|25), risk_thresholds, sla_defaults, reminder_cadence, retention_years }
- `ReminderCadence` — { days_before: [60,30,14,7,1], overdue_days: [3,7,14] }
- `RiskThresholds` — { low_max, medium_max, high_max } (scores defining tier boundaries)

### Domain Events
- `UserCreated`
- `UserAssignedToTenant`
- `UserRoleChanged`
- `TenantCreated`
- `TenantConfigurationUpdated`
- `TenantSuspended`

---

## Bounded Context 2: Client Management (CRM Core)

> **Architecture level: Service Layer** — SQLAlchemy models as source of truth, Pydantic for validation, service.py for state machine and UBO logic. No separate domain entities or repository interfaces.

> The pivot of the entire data model. Every other context references ClientId.

### Aggregates

#### Client (Aggregate Root)
```
Client
├── client_id: ClientId (UUID)
├── tenant_id: TenantId
├── name: str
├── type: ClientType (VO)
├── status: ClientStatus (VO)
├── jurisdiction: JurisdictionCode (VO)
├── contact_info: ContactInfo (VO)
├── engagement_date: Optional[date]
├── risk_tier: Optional[RiskTier]
├── assigned_team: List[UserId]
├── created_by: UserId
├── created_at: datetime
├── updated_at: datetime
│
├── transition_to(new_status, by: UserId) → None
├── assign_risk_tier(tier, by: UserId) → None
├── assign_team_member(user_id) → None
├── reactivate(by: UserId) → None
└── update_contact(info: ContactInfo) → None
```

**Invariants:**
- Status transitions follow defined state machine (not all transitions valid)
- EXITED → REACTIVATION requires explicit reactivation command
- Only one active OnboardingCase per client at a time (soft rule, allow override)

#### Person (Aggregate Root — Shared Entity)
```
Person
├── person_id: PersonId (UUID)
├── tenant_id: TenantId
├── full_name: str
├── nationality: CountryCode
├── date_of_birth: date
├── id_document: IdentityDocument (VO)
├── address: Address (VO)
├── contact: Optional[ContactInfo]
├── occupation: Optional[str]
├── is_pep: bool
├── pep_details: Optional[PepDetails]
├── screening_status: ScreeningStatus (VO)
├── ubo_links: List[UboLink] (VO)
│
├── mark_as_pep(details: PepDetails) → None
├── update_screening(result: ScreeningResult) → None
├── link_as_ubo(client_id, ownership_pct, role) → UboLink
├── update_id_document(doc: IdentityDocument) → None
└── needs_reverification() → bool
```

**Invariants:**
- `is_pep=True` → linked clients must have risk_tier >= HIGH
- Re-verification required annually (6 months for HIGH/VERY_HIGH)
- UBO ownership_pct must be >= tenant's ubo_threshold

#### ClientDocument (Aggregate Root)
```
ClientDocument
├── document_id: DocumentId (UUID)
├── tenant_id: TenantId
├── client_id: ClientId
├── category: DocumentCategory (VO)
├── filename: str
├── storage_key: str
├── version: int
├── versions: List[DocumentVersion] (VO)
├── expiry_date: Optional[date]
├── uploaded_by: UserId
├── uploaded_at: datetime
│
├── upload_new_version(file, by: UserId) → DocumentVersion
├── is_expired() → bool
├── days_until_expiry() → Optional[int]
└── delete(by: UserId) → None  # soft delete with audit
```

### Value Objects
- `ClientId`, `PersonId`, `DocumentId` — UUID wrappers
- `ClientType` — enum: INDIVIDUAL, CORPORATE, FUND, FOUNDATION, FINTECH_VASP
- `ClientStatus` — enum: PROSPECT, ONBOARDING, ACTIVE, UNDER_REVIEW, SUSPENDED, EXITED, REACTIVATION
- `RiskTier` — enum: LOW, MEDIUM, HIGH, VERY_HIGH, UNACCEPTABLE
- `JurisdictionCode` — enum: DIFC, ADGM, DUBAI_MAINLAND, ABU_DHABI, UAE_ONSHORE, OFFSHORE
- `ContactInfo` — { email, phone, address }
- `IdentityDocument` — { type (PASSPORT, EMIRATES_ID, RESIDENCE_VISA, OTHER), number, issuing_country, issue_date, expiry_date, document_file_id }
- `UboLink` — { client_id, ownership_pct, role, effective_date }
- `PepDetails` — { pep_type, position, country, source, identified_date }
- `ScreeningStatus` — { last_screened_at, result (CLEAR, HIT_PENDING, FALSE_POSITIVE, TRUE_MATCH), next_screening_due }
- `DocumentCategory` — enum: IDENTITY, CORPORATE, OWNERSHIP, FINANCIAL, ENGAGEMENT, SCREENING, OTHER
- `DocumentVersion` — { version_number, storage_key, uploaded_by, uploaded_at, change_note }
- `Address` — { line1, line2, city, state, country, postal_code }

### Domain Events
- `ClientCreated`
- `ClientStatusChanged { from_status, to_status, reason }`
- `ClientRiskTierAssigned { tier, previous_tier }`
- `PersonCreated`
- `PersonMarkedAsPep`
- `PersonLinkedAsUbo { client_id, ownership_pct }`
- `DocumentUploaded { client_id, category }`
- `DocumentExpiring { client_id, document_id, days_until_expiry }`
- `DocumentExpired { client_id, document_id }`
- `ClientRiskTierEscalated { client_id, previous_tier, new_tier, reason }`
- `UboLinkCreated { person_id, client_id, ownership_pct }`
- `UboLinkRemoved { person_id, client_id, reason }`
- `DocumentExpiredAndNotRenewed { client_id, document_id }`

---

## Bounded Context 3: AML & Onboarding

> **Architecture level: Full DDD** — This is the most complex context with 8-stage state machine, pluggable risk scoring, screening hit workflow with regulatory invariants. Full domain/application/infrastructure/api layers justified.

> Most complex domain invariants. 70% time saving target.

### Aggregates

#### OnboardingCase (Aggregate Root)

> **Design note**: Embedded collections are acceptable for Phase 1 scale (< 100 clients). If performance degrades, extract ScreeningResults and ChecklistItems as separate aggregates in Phase 2.

```
OnboardingCase
├── case_id: CaseId (UUID)
├── tenant_id: TenantId
├── client_id: ClientId
├── stage: OnboardingStage (VO)
├── client_type: ClientType
├── assigned_analyst: UserId
├── assigned_approver: Optional[UserId]
├── risk_score: Optional[RiskScore] (VO)
├── screening_results: List[ScreeningResult]
├── document_checklist: List[ChecklistItem]
├── sla_deadline: date
├── created_by: UserId
├── created_at: datetime
├── completed_at: Optional[datetime]
├── notes: List[CaseNote]
│
├── advance_stage(by: UserId) → None
├── skip_to_stage(stage, justification, by: UserId) → None
├── submit_risk_score(score: RiskScore, by: UserId) → None
├── override_risk_score(new_score, rationale, by: UserId, approved_by: UserId) → None
├── add_screening_result(result: ScreeningResult) → None
├── approve(by: UserId) → None
├── reject(reason, by: UserId) → None
├── mark_overdue() → None
└── check_sla_breach() → bool
│
│  Invariants:
│  - Optimistic locking via version field on every save
│  - All commands require idempotency_key
```

**Invariants:**
- Stage transitions must follow defined order (no skipping screening or risk score)
- Stage skip for low-risk only if DD still valid and no triggers; screening + risk score still required
- HIGH risk override requires dual sign-off + documented rationale
- PEP as UBO → risk_tier must be >= HIGH (auto-EDD)
- SLA breach at 10 business days → escalation
- Cannot approve if risk_tier == UNACCEPTABLE (unless explicit exception)

#### ScreeningCase (Aggregate Root)
```
ScreeningCase
├── screening_id: ScreeningId (UUID)
├── tenant_id: TenantId
├── subject_type: ScreeningSubjectType (CLIENT | UBO | PERSON)
├── subject_id: str (ClientId or PersonId)
├── screening_type: ScreeningType (INITIAL | PERIODIC | TRIGGERED)
├── provider: Optional[ScreeningProvider]
├── lists_checked: List[ScreeningList]
├── hits: List[ScreeningHit]
├── status: ScreeningStatus
├── screened_at: datetime
├── screened_by: UserId
├── next_screening_due: date
│
├── record_hit(hit: ScreeningHit) → None
├── mark_false_positive(hit_id, rationale, by: UserId, approved_by: UserId) → None
├── confirm_true_match(hit_id, by: UserId) → None
├── mark_clear(by: UserId) → None
└── requires_founder_signoff(hit: ScreeningHit) → bool
```

**Invariants:**
- Sanctions hits: ALWAYS require founder sign-off for false positive
- False positive rationale minimum: 3-5 sentences with identifiers checked + why mismatch + evidence
- True match: onboarding must be stopped; notify CO/MLRO same day
- Do not auto-block — only notify for manual action

### Value Objects
- `CaseId`, `ScreeningId` — UUID wrappers
- `OnboardingStage` — enum: CLIENT_INTAKE, SCREENING, RISK_SCORING, DUE_DILIGENCE, FILE_COMPLETION, APPROVAL, COMPLETED, REJECTED
- `RiskScore` — { total_score, tier (RiskTier), factors: List[RiskFactor], calculated_by (UserId|AgentId), calculated_at, is_override, override_rationale? }
- `RiskFactor` — { name, weight, score, evidence }
- `ScreeningHit` — { hit_id, list, matched_name, match_score, disposition (PENDING, FALSE_POSITIVE, TRUE_MATCH), rationale?, reviewed_by?, approved_by? }
- `ScreeningList` — enum: UAE_LOCAL, UN, HMT_UK, OFAC_USA, EU, PEP, ADVERSE_MEDIA
- `ScreeningProvider` — enum: WORLD_CHECK, COMPLY_ADVANTAGE, MANUAL
- `ChecklistItem` — { item_name, required, status (PENDING, RECEIVED, WAIVED), document_id? }
- `CaseNote` — { note, author: UserId, created_at }
- `DueDiligenceLevel` — enum: STANDARD_CDD, ENHANCED_EDD

### Domain Events
- `OnboardingCaseCreated`
- `OnboardingStageAdvanced { from_stage, to_stage }`
- `RiskScoreSubmitted { case_id, tier, score }`
- `RiskScoreOverridden { case_id, original_tier, new_tier, rationale }`
- `ScreeningCompleted { subject_id, status }`
- `ScreeningHitRecorded { subject_id, list, match_score }`
- `ScreeningHitDisposed { hit_id, disposition }`
- `TrueMatchConfirmed { subject_id, hit_id }` → triggers escalation
- `OnboardingApproved { case_id, client_id }`
- `OnboardingRejected { case_id, reason }`
- `OnboardingSlaBreached { case_id }`

---

## Bounded Context 4: Compliance Framework

> **Architecture level: Service Layer** — SQLAlchemy models with service.py for policy approval workflow, incident state machine, and register management logic.

### Aggregates

#### Policy (Aggregate Root)
```
Policy
├── policy_id: PolicyId (UUID)
├── tenant_id: TenantId
├── title: str
├── type: PolicyType
├── version: PolicyVersion (VO)
├── status: PolicyStatus
├── content_document_id: DocumentId
├── review_triggers: List[ReviewTrigger]
├── next_review_date: date
├── approval_workflow: ApprovalWorkflow (VO)
├── amendment_history: List[Amendment]
├── created_by: UserId
│
├── submit_for_review(by: UserId) → None
├── approve(by: UserId) → None
├── publish(by: UserId) → None
├── create_amendment(content, by: UserId) → Amendment
├── schedule_review(date) → None
└── trigger_review(reason: ReviewTrigger) → None
```

#### ComplianceRegister (Aggregate Root)
```
ComplianceRegister
├── register_id: RegisterId (UUID)
├── tenant_id: TenantId
├── type: RegisterType
├── is_reg_submission: bool
├── schema_config: Optional[dict]  # defines required fields per type
├── version: int
├── created_at: datetime
│
├── update_config(config, by: UserId) → None
├── mark_for_submission() → None
└── export(format: ExportFormat) → bytes
```

#### RegisterEntry (Aggregate Root)
```
RegisterEntry
├── entry_id: EntryId (UUID)
├── tenant_id: TenantId
├── register_id: RegisterId
├── register_type: RegisterType
├── data: dict  # flexible schema per register type
├── severity: Optional[str]  # for breach/incident registers
├── status: Optional[str]
├── reported_by: UserId
├── version: int
├── created_at: datetime
├── updated_at: datetime
│
├── update(data: dict, by: UserId) → None
└── close(by: UserId) → None
```

> **Training records**: Modeled as entries in the TRAINING register (RegisterType.TRAINING). Fields: user_id, module, completion_date, attestation, score, trainer, next_due_date. No separate aggregate needed.

#### IncidentCase (Aggregate Root)
```
IncidentCase
├── incident_id: IncidentId (UUID)
├── tenant_id: TenantId
├── title: str
├── description: str
├── severity: BreachSeverity
├── status: IncidentStatus
├── regulatory_notification_required: bool
├── notification_deadline: Optional[datetime]
├── remediation_actions: List[RemediationAction]
├── escalation_log: List[EscalationEntry]
│
├── assess_severity(severity, by: UserId) → None
├── escalate(to: UserId, reason) → None
├── add_remediation(action: RemediationAction) → None
├── close(evidence, by: UserId) → None
├── reopen(reason, by: UserId) → None
└── requires_regulatory_notification() → bool
```

**Invariants:**
- `severity == CRITICAL` → `regulatory_notification_required` must be assessed before close
- CRITICAL escalation: immediate CO/MLRO, then board/regulator
- Notification SLA: 24-72 hours after determination
- Re-opening allowed with append-only audit trail

### Value Objects
- `PolicyType` — enum: AML_CFT, SANCTIONS, CDD_EDD, CONFLICTS, CONDUCT, COMPLAINTS, DATA_PROTECTION, RECORD_KEEPING, OUTSOURCING, RISK_MANAGEMENT, WHISTLEBLOWING, GIFTS, BCP_IT, TRAINING, CORPORATE_GOVERNANCE, REMUNERATION, FRAUD, SOCIAL_MEDIA, SECURITY
- `PolicyStatus` — enum: DRAFT, UNDER_REVIEW, APPROVED, PUBLISHED, ARCHIVED
- `PolicyVersion` — { major, minor } (e.g., 1.0, 1.1)
- `ReviewTrigger` — enum: ANNUAL, REGULATORY_CHANGE, AUDIT_FINDING, CMP_FINDING, BUSINESS_CHANGE
- `RegisterType` — enum: BREACH, COMPLAINTS, GIFTS, CONFLICTS, SCREENING_LOG, REGULATORY_CORRESPONDENCE, TRAINING, OUTSOURCING, RISK, CLIENT, CMP_FINDINGS, SAR, MARKETING, DIRECTORS, SHAREHOLDERS, EMPLOYEES, UBO, PAD, OUTSIDE_INTEREST
- `BreachSeverity` — enum: LOW, MEDIUM, HIGH, CRITICAL
- `IncidentStatus` — enum: OPEN, UNDER_ASSESSMENT, REMEDIATION, CLOSED, REOPENED

### Domain Events
- `PolicyCreated`
- `PolicySubmittedForReview`
- `PolicyApproved`
- `PolicyPublished`
- `RegisterEntryAdded { register_type }`
- `IncidentCreated { severity }`
- `IncidentEscalated { to_user, severity }`
- `IncidentClosed`
- `IncidentReopened`
- `RegulatoryNotificationRequired { incident_id, deadline }`
- `TrainingCompleted { user_id, module }`
- `TrainingComplianceBreached { user_id, module, days_overdue }`

---

## Bounded Context 5: Regulatory Intelligence

> **Architecture level: Simple CRUD** (Phase 1) — Basic change log tracking. Promotes to Service Layer when impact assessment workflow is added in Phase 2.

### Aggregates

#### RegulatoryChange (Aggregate Root)
```
RegulatoryChange
├── change_id: ChangeId (UUID)
├── regulator: RegulatorCode
├── category: ChangeCategory
├── title: str
├── summary: str
├── source_url: str
├── detected_at: datetime
├── effective_date: Optional[date]
├── implementation_deadline: Optional[date]
├── grace_period_days: Optional[int]
├── impact_assessment: Optional[ImpactAssessment] (VO)
├── downstream_triggers: List[DownstreamTrigger]
├── status: ChangeStatus
│
├── draft_impact_assessment(assessment, by: UserId|AgentId) → None
├── approve_assessment(by: UserId) → None
├── trigger_downstream_actions() → List[DownstreamTrigger]
└── archive() → None
```

#### ClientAlert (Aggregate Root)
```
ClientAlert
├── alert_id: AlertId (UUID)
├── tenant_id: TenantId
├── change_id: ChangeId
├── title: str
├── content: str
├── affected_clients: List[ClientId]
├── status: AlertStatus (DRAFT, PENDING_APPROVAL, APPROVED, SENT)
├── approval: Optional[AlertApproval]
│
├── submit_for_approval(by: UserId) → None
├── approve(by: UserId) → None
├── dual_approve(by: UserId) → None  # for HIGH impact
├── send(via: NotificationChannel) → None
└── archive() → None
```

### Value Objects
- `ChangeCategory` — enum: RULE_CHANGE, GUIDANCE, ENFORCEMENT, CONSULTATION, FAQ_NOTICE
- `ChangeStatus` — enum: DETECTED, ASSESSMENT_DRAFT, ASSESSED, ACTIONED, ARCHIVED
- `ImpactAssessment` — { impact_level (LOW, MEDIUM, HIGH), affected_firm_types, required_actions, assessed_by, assessed_at }
- `DownstreamTrigger` — enum: POLICY_REVIEW, TRAINING_SCHEDULE, CLIENT_ALERT, CMP_UPDATE, BOARD_PACK_FLAG
- `AlertApproval` — { approved_by: List[UserId], approved_at, requires_dual: bool }

### Domain Events
- `RegulatoryChangeDetected`
- `ImpactAssessmentDrafted`
- `ImpactAssessmentApproved`
- `DownstreamTriggersCreated { triggers: List[DownstreamTrigger] }`
- `ClientAlertApproved`
- `ClientAlertSent { alert_id, client_ids }`

---

## Bounded Context 6: Authorisations

> **Phase 1**: Implemented as light CRUD tracking (simple table + API). Full DDD aggregate methods activated in Phase 2.
> This aligns with the **Simple CRUD** architecture level for Phase 1.

> **Phase 1 code**: Implemented as `light_tracking/models.py` → `light_auth_cases` table. See `06-data-model.md`.
> **Phase 2 promotion**: Migrate data from `light_auth_cases` to `auth_cases` when full workflows are implemented.

### Aggregates

#### AuthorisationCase (Aggregate Root)
```
AuthorisationCase
├── case_id: AuthCaseId (UUID)
├── tenant_id: TenantId
├── client_id: ClientId
├── regulator: RegulatorCode
├── licence_category: LicenceCategory (VO)
├── stage: AuthorisationStage (VO)
├── stage_history: List[StageTransition]
├── documents: List[CaseDocument]
├── rfis: List[RfiId]
├── timeline_estimate: Optional[TimePeriod]
├── assigned_team: List[UserId]
│
├── advance_stage(by: UserId) → None
├── add_document(doc: CaseDocument) → None
├── submit_to_regulator(by: UserId) → None
├── receive_rfi(rfi: RfiReference) → None
├── grant_licence(licence_details, by: UserId) → None  # triggers handoff
├── reject(reason, by: UserId) → None
└── withdraw(reason, by: UserId) → None
```

#### Rfi (Aggregate Root)
```
Rfi
├── rfi_id: RfiId (UUID)
├── auth_case_id: AuthCaseId
├── regulator: RegulatorCode
├── questions: List[RfiQuestion]
├── response_deadline: Optional[date]
├── status: RfiStatus
├── response_document_id: Optional[DocumentId]
│
├── draft_response(content, by: UserId) → None
├── submit_response(by: UserId) → None
├── close(by: UserId) → None
└── link_precedent(precedent_id: RfiId) → None
```

### Value Objects
- `AuthorisationStage` — enum: ENGAGEMENT, CONFLICTS_CHECK, DOC_COLLECTION, DRAFTING, QA_REVIEW, FEE_PAYMENT, SUBMISSION, RFI_HANDLING, REGULATOR_REVIEW, APPROVED, REJECTED, WITHDRAWN
- `LicenceCategory` — enum: CAT_1_DEPOSIT, CAT_2_CREDIT_DEALING, CAT_3A_BROKERAGE, CAT_3C_ASSET_MGMT, CAT_3D_MONEY_SERVICES, CAT_4_ADVISORY, VARA_VASP
- `StageTransition` — { from_stage, to_stage, transitioned_by, transitioned_at, notes }
- `RfiStatus` — enum: RECEIVED, IN_PROGRESS, RESPONSE_DRAFTED, RESPONSE_SUBMITTED, CLOSED

### Domain Events
- `AuthorisationCaseCreated`
- `AuthorisationStageAdvanced { from, to }`
- `SubmissionCompleted { case_id, regulator }`
- `RfiReceived { case_id, rfi_id }`
- `RfiResponseSubmitted`
- `LicenceGranted { case_id, client_id }` → triggers compliance handoff
- `LicenceRejected { case_id, reason }`
- `PostLicenceHandoffCompleted { case_id, items_created: List[str] }`
- `PostLicenceHandoffFailed { case_id, reason }`

---

## Bounded Context 7: Entity Management

> **Phase 1**: Implemented as light CRUD tracking (simple table + API). Full DDD aggregate methods activated in Phase 2.
> This aligns with the **Simple CRUD** architecture level for Phase 1.

> **Phase 1 code**: Implemented as `light_tracking/models.py` → `light_entities` table. See `06-data-model.md`.
> **Phase 2 promotion**: Migrate data from `light_entities` to `ent_legal_entities` when full entity management is implemented.

### Aggregates

#### LegalEntity (Aggregate Root — slim)
```
LegalEntity
├── entity_id: EntityId (UUID)
├── tenant_id: TenantId
├── name: str
├── type: EntityType (VO)
├── jurisdiction: JurisdictionCode
├── parent_entity_id: Optional[EntityId]
├── children: List[EntityId]
├── registered_office: Address
├── key_dates: EntityKeyDates (VO)
├── status: EntityStatus
├── version: int
│
├── update_details(name, address, by: UserId) → None
├── set_parent(parent_id: EntityId) → None
└── archive(by: UserId) → None
```

#### EntityGovernance (Aggregate Root)
```
EntityGovernance
├── entity_id: EntityId
├── tenant_id: TenantId
├── directors: List[PersonId]
├── beneficial_owners: List[UboLink]
├── service_providers: List[ServiceProvider] (VO)
├── version: int
│
├── update_directors(directors: List[PersonId], by: UserId) → None
├── update_beneficial_owners(owners: List[UboLink], by: UserId) → None
├── add_service_provider(sp: ServiceProvider) → None
└── remove_service_provider(sp_id, by: UserId) → None
```

#### EntityObligations (Aggregate Root)
```
EntityObligations
├── entity_id: EntityId
├── tenant_id: TenantId
├── obligations: List[Obligation]
├── compliance_health: ComplianceHealth (VO)
├── version: int
│
├── add_obligation(obligation: Obligation) → None
├── complete_obligation(obligation_id, evidence, by: UserId) → None
├── calculate_compliance_health() → ComplianceHealth
├── generate_renewal_pack() → DocumentId
└── get_overdue_obligations() → List[Obligation]
```

### Value Objects
- `EntityType` — enum: SPV, FOUNDATION, FUND, HOLDING_COMPANY, INTERNAL_GROUP, CLIENT_COMPANY
- `Obligation` — { obligation_id, name, type, frequency, due_date, status, regulatory_filing: bool, last_completed, next_due }
- `ObligationType` — enum: ANNUAL_RENEWAL, REGISTERED_AGENT_FEE, ANNUAL_RETURN, UBO_UPDATE, DIRECTOR_RENEWAL, ES_FILING, FATCA_CRS, LICENCE_RENEWAL, BOARD_RESOLUTION, SERVICE_PROVIDER_RENEWAL, AUDIT
- `ObligationStatus` — enum: UPCOMING, DUE, OVERDUE, COMPLETED, WAIVED
- `ServiceProvider` — { name, type, contact, contract_start, contract_end }
- `EntityKeyDates` — { incorporation_date, licence_expiry, next_renewal, audit_period_end }
- `ComplianceHealth` — { score, traffic_light (GREEN, AMBER, RED), overdue_obligations, missing_documents }

### Domain Events
- `LegalEntityCreated`
- `ObligationDue { entity_id, obligation_id, days_until }`
- `ObligationOverdue { entity_id, obligation_id, days_overdue }`
- `ObligationCompleted { entity_id, obligation_id }`
- `ComplianceHealthChanged { entity_id, traffic_light }`

---

## Bounded Context 8: CMP Monitoring

### Aggregates

#### MonitoringTest (Aggregate Root)
```
MonitoringTest
├── test_id: TestId (UUID)
├── tenant_id: TenantId
├── domain: ComplianceDomain (VO)
├── title: str
├── description: str
├── methodology: str
├── rule_reference: str
├── frequency: TestFrequency (VO)
├── trigger_type: TriggerType (CALENDAR | EVENT)
├── risk_owner: UserId
├── version: int
│
├── schedule(due_date: date) → ScheduledTest
├── update_methodology(text, by: UserId) → None
└── archive(by: UserId) → None
```

#### ScheduledTest (Aggregate Root)
```
ScheduledTest
├── scheduled_id: ScheduledTestId (UUID)
├── test_id: TestId
├── tenant_id: TenantId
├── assigned_to: UserId
├── due_date: date
├── status: ScheduledTestStatus
├── working_paper: Optional[WorkingPaper] (VO)
├── findings: List[Finding]
│
├── assign(user_id: UserId) → None
├── reschedule(new_date, reason_code, by: UserId) → None
├── start_execution(by: UserId) → None
├── submit_working_paper(paper: WorkingPaper, by: UserId) → None
├── add_finding(finding: Finding) → None
├── review(by: UserId) → None
├── complete(by: UserId) → None
└── mark_overdue() → None
```

#### Finding (Aggregate Root)
```
Finding
├── finding_id: FindingId (UUID)
├── tenant_id: TenantId
├── scheduled_test_id: ScheduledTestId
├── title: str
├── description: str
├── severity: FindingSeverity
├── root_cause: Optional[str]
├── remediation: RemediationPlan (VO)
├── status: FindingStatus
│
├── assign_remediation(plan: RemediationPlan) → None
├── update_status(status, evidence, by: UserId) → None
├── close(evidence, by: UserId) → None
├── reopen(reason, by: UserId) → None
└── check_sla_breach() → bool
```

### Value Objects
- `ComplianceDomain` — enum: AML_CFT, CONDUCT, FUND_ADMIN, CORPORATE_GOVERNANCE, HR, MARKETING, RECORD_KEEPING, REGULATORY, RISK_MANAGEMENT, ACCOUNTING, EXTERNAL_AUDIT, INTERNAL_AUDIT, DATA_PROTECTION, MONEY_SERVICES, FUND_MANAGEMENT, ASSET_MANAGEMENT, CUSTODY
- `TestFrequency` — enum: MONTHLY, QUARTERLY, HALF_YEARLY, YEARLY
- `WorkingPaper` — { scope, rule_reference, methodology, sample_selection, sample_rationale, evidence_list: List[EvidenceItem], testing_performed, findings_summary, conclusion, action_points, tester_sign_off, reviewer_sign_off }
- `EvidenceItem` — { type, source, date, owner, file_id }
- `FindingSeverity` — enum: OBSERVATION, LOW, MEDIUM, HIGH, CRITICAL
- `FindingStatus` — enum: OPEN, REMEDIATION_PLANNED, IN_REMEDIATION, CLOSED, REOPENED
- `RemediationPlan` — { action_owner: UserId, due_date, description, sla_days, evidence?: DocumentId }
- `ScheduledTestStatus` — enum: SCHEDULED, IN_PROGRESS, UNDER_REVIEW, COMPLETED, OVERDUE

**Remediation SLAs:**
- Critical: plan within 7 days, close within 30 days
- High: 30 days
- Medium: 60 days
- Low: 90 days
- Observation: 120 days (configurable)

### Domain Events
- `TestScheduled`
- `TestStarted`
- `WorkingPaperSubmitted`
- `TestCompleted`
- `TestOverdue { test_id, days_overdue }`
- `FindingCreated { severity }`
- `FindingClosed`
- `FindingReopened`
- `RemediationSlaBreached { finding_id }`

---

## Bounded Context 9: Document Factory

> **Architecture level: Simple CRUD** — Template management and document generation. Business logic in generator functions, not domain entities.

### Aggregates

#### DocumentTemplate (Aggregate Root)
```
DocumentTemplate
├── template_id: TemplateId (UUID)
├── tenant_id: Optional[TenantId]  # null = global template
├── name: str
├── type: TemplateType
├── category: DocumentCategory
├── merge_fields: List[MergeField]
├── file_id: str (storage key to DOCX/PPTX template)
├── version: int
├── updated_by: UserId
│
├── generate(data: dict, output_format: OutputFormat) → GeneratedDocument
├── update_template(file, by: UserId) → None
├── list_merge_fields() → List[MergeField]
└── archive(by: UserId) → None
```

#### GeneratedDocument (Aggregate Root)
```
GeneratedDocument
├── document_id: GeneratedDocId (UUID)
├── tenant_id: TenantId
├── template_id: TemplateId
├── source_context: SourceContext (VO)  # which case/client/entity
├── output_format: OutputFormat
├── file_id: str (storage key)
├── version: int
├── versions: List[DocumentVersion]
├── retention_until: date
├── created_by: UserId
├── created_at: datetime
│
├── regenerate(data: dict) → None
├── archive() → None
├── delete_if_retention_expired() → bool
└── export(format: OutputFormat) → bytes
```

### Value Objects
- `TemplateType` — enum: ONBOARDING_CHECKLIST, DD_FORM, RISK_MATRIX, SCREENING_MEMO, ENGAGEMENT_LETTER, AUTH_CHECKLIST, CMP_WORKING_PAPER, BOARD_REPORT, POLICY_MANUAL, EVIDENCE_PACK
- `OutputFormat` — enum: DOCX, PDF, PPTX, ZIP
- `MergeField` — { field_name, source_path, type }
- `SourceContext` — { context_type (CLIENT, CASE, ENTITY, TEST), context_id }
- `DocumentVersion` — { version, file_id, created_by, created_at }

### Domain Events
- `DocumentGenerated { template_type, source_context }`
- `DocumentVersionCreated`
- `DocumentRetentionExpired { document_id }`

---

## Bounded Context 10: Workflow Engine (Shared Kernel)

> **Architecture level: Service Layer** — SQLAlchemy models with service.py for task assignment, approval workflows, and escalation logic.

> Used by all domain contexts for task management, approvals, and reminders.

### Aggregates

#### Task (Aggregate Root)
```
Task
├── task_id: TaskId (UUID)
├── tenant_id: TenantId
├── title: str
├── description: Optional[str]
├── type: TaskType
├── source_context: SourceContext (VO)
├── assigned_to: UserId
├── due_date: date
├── priority: TaskPriority
├── status: TaskStatus
├── approval: Optional[ApprovalRequest]
├── reminders_sent: List[ReminderRecord]
│
├── assign(user_id: UserId) → None
├── start() → None
├── complete(evidence?: str) → None
├── mark_overdue() → None
├── request_approval(approvers: List[UserId], dual: bool) → None
├── approve(by: UserId) → None
├── reject(reason, by: UserId) → None
└── escalate(to: UserId, reason) → None
```

### Value Objects
- `TaskType` — enum: DOCUMENT_REVIEW, APPROVAL, SCREENING_REVIEW, POLICY_REVIEW, TRAINING, OBLIGATION, REMEDIATION, RFI_RESPONSE, GENERIC
- `TaskPriority` — enum: LOW, MEDIUM, HIGH, URGENT
- `TaskStatus` — enum: OPEN, IN_PROGRESS, PENDING_APPROVAL, COMPLETED, OVERDUE, CANCELLED
- `ApprovalRequest` — { approvers, requires_dual, approvals_received: List[Approval], status }
- `Approval` — { approved_by, approved_at, decision (APPROVED|REJECTED), reason? }
- `ReminderRecord` — { sent_at, channel, days_before_due }

### Domain Events
- `TaskCreated`
- `TaskAssigned`
- `TaskCompleted`
- `TaskOverdue { task_id, days_overdue }`
- `ApprovalRequested { task_id, approvers }`
- `ApprovalGranted { task_id, by }`
- `ApprovalRejected { task_id, by, reason }`
- `ReminderSent { task_id, channel }`
- `EscalationTriggered { task_id, to_user }`

---

## Bounded Context 11: Audit Trail (Shared Kernel)

### Aggregates

#### AuditEntry (Append-Only — NOT an aggregate in traditional sense)
```
AuditEntry
├── entry_id: UUID
├── tenant_id: TenantId
├── actor_type: ActorType (USER | AGENT | SYSTEM)
├── actor_id: str
├── timestamp: datetime (UTC)
├── action: str (command name)
├── resource_type: str
├── resource_id: str
├── before_snapshot: Optional[dict]
├── after_snapshot: Optional[dict]
├── source: ActionSource (UI | API | AGENT | SYSTEM)
├── correlation_id: UUID
├── ip_address: Optional[str]
├── metadata: Optional[dict]
```

**Rules:**
- APPEND-ONLY: no update, no delete
- Minimum 6-year retention (configurable)
- Exportable as CSV/PDF with filters
- Every domain command creates an AuditEntry
- Agent actions include agent identity and confidence score

---

## Light Tracking (Phase 1 — CRUD Only)

> These features are implemented as simple CRUD tables in Phase 1. No bounded context, no domain logic, no events. They will be promoted to full bounded contexts in Phase 2 when business rules are added.

### Auth Case Light
Simple tracking of regulatory applications through stages.

```
Table: light_auth_cases
├── id: UUID
├── tenant_id: TenantId
├── client_name: str
├── regulator: RegulatorCode
├── licence_category: str
├── stage: str  # free-text or enum from AuthorisationStage
├── assigned_to: Optional[UserId]
├── key_dates: JSONB  # { engagement_date, submission_date, expected_decision }
├── notes: str
├── created_by: UserId
├── created_at: datetime
├── updated_at: datetime
```

### Entity Light
Simple tracking of SPVs, foundations, and entities with key dates.

```
Table: light_entities
├── id: UUID
├── tenant_id: TenantId
├── name: str
├── type: str  # SPV, FOUNDATION, FUND, etc.
├── jurisdiction: str
├── registration_date: Optional[date]
├── licence_expiry: Optional[date]
├── next_renewal: Optional[date]
├── directors: JSONB  # [{ name, person_id? }]
├── beneficial_owners: JSONB  # [{ name, person_id?, ownership_pct }]
├── status: str  # GREEN, AMBER, RED (computed from dates)
├── notes: str
├── created_by: UserId
├── created_at: datetime
├── updated_at: datetime
```

These tables have:
- Basic CRUD API endpoints
- Audit trail (via middleware, same as all tables)
- RLS for tenant isolation
- No domain events, no invariants, no state machines
- Promoted to full bounded contexts in Phase 2

---

## Shared Kernel Summary

These concepts are shared across ALL bounded contexts:

> **Pydantic everywhere**: All value objects, domain events, and commands use Pydantic BaseModel (frozen). AML domain entities use Pydantic BaseModel. Service Layer and CRUD contexts use SQLAlchemy models directly. No dataclasses in the project.

```python
# shared/domain/base.py
from pydantic import BaseModel, ConfigDict
from uuid import UUID
from datetime import datetime
from typing import Optional

TenantId = UUID
UserId = UUID

class ValueObject(BaseModel):
    """Immutable, compared by value. Pydantic handles validation."""
    model_config = ConfigDict(frozen=True)

class DomainEvent(BaseModel):
    """Base for all domain events."""
    model_config = ConfigDict(frozen=True)

    event_id: UUID
    tenant_id: TenantId
    occurred_at: datetime
    correlation_id: UUID
    caused_by: UserId | str  # str for agent identity
    idempotency_key: Optional[UUID] = None

class DomainException(Exception):
    """Base for all domain errors."""

class UnauthorizedActionError(DomainException): ...
class InvalidStateTransitionError(DomainException): ...
class InvariantViolationError(DomainException): ...
class EntityNotFoundError(DomainException): ...
class ConcurrencyConflictError(DomainException): ...
```

> **Note**: No `Entity` or `AggregateRoot` base classes. In the hybrid approach, SQLAlchemy models ARE the entities for Service Layer/CRUD, and AML's domain entities use Pydantic BaseModel directly. The `version` field and `_domain_events` list are on the AML domain entities, not on a shared base class.

### State Machine Pattern

All aggregates with lifecycle states use a simple transition dictionary pattern:

```python
# shared/domain/state_machine.py
from typing import Dict, Set
from shared.domain.exceptions import InvalidStateTransitionError

class StateMachine:
    """Simple state machine using transition dictionary. No external dependencies."""

    def __init__(self, transitions: Dict[str, Set[str]]):
        self._transitions = transitions

    def validate_transition(self, current: str, target: str) -> None:
        valid = self._transitions.get(current, set())
        if target not in valid:
            raise InvalidStateTransitionError(
                f"Cannot transition from {current} to {target}. "
                f"Valid transitions: {valid}"
            )

# Usage in domain:
CLIENT_TRANSITIONS = {
    "PROSPECT": {"ONBOARDING"},
    "ONBOARDING": {"ACTIVE", "SUSPENDED", "EXITED"},
    "ACTIVE": {"UNDER_REVIEW", "SUSPENDED", "EXITED"},
    "UNDER_REVIEW": {"ACTIVE", "SUSPENDED", "EXITED"},
    "SUSPENDED": {"ACTIVE", "UNDER_REVIEW", "EXITED"},
    "EXITED": {"REACTIVATION"},
    "REACTIVATION": {"ONBOARDING"},
}

ONBOARDING_TRANSITIONS = {
    "CLIENT_INTAKE": {"SCREENING"},
    "SCREENING": {"RISK_SCORING"},
    "RISK_SCORING": {"DUE_DILIGENCE"},
    "DUE_DILIGENCE": {"FILE_COMPLETION"},
    "FILE_COMPLETION": {"APPROVAL"},
    "APPROVAL": {"COMPLETED", "REJECTED"},
    "REJECTED": set(),  # terminal
    "COMPLETED": set(),  # terminal
}

INCIDENT_TRANSITIONS = {
    "OPEN": {"UNDER_ASSESSMENT"},
    "UNDER_ASSESSMENT": {"REMEDIATION", "CLOSED"},
    "REMEDIATION": {"CLOSED"},
    "CLOSED": {"REOPENED"},
    "REOPENED": {"UNDER_ASSESSMENT"},
}

POLICY_TRANSITIONS = {
    "DRAFT": {"UNDER_REVIEW"},
    "UNDER_REVIEW": {"APPROVED", "DRAFT"},
    "APPROVED": {"PUBLISHED"},
    "PUBLISHED": {"ARCHIVED", "DRAFT"},  # new version starts as DRAFT
    "ARCHIVED": set(),
}
```

Every aggregate with states implements:
```python
class Client(AggregateRoot):
    _state_machine = StateMachine(CLIENT_TRANSITIONS)

    def transition_to(self, new_status: str, by: UserId) -> None:
        self._state_machine.validate_transition(self.status, new_status)
        old_status = self.status
        self.status = new_status
        self.add_event(ClientStatusChanged(
            client_id=self.client_id,
            from_status=old_status,
            to_status=new_status,
        ))
```

### Cross-Context Orchestration (Saga Pattern)

For Phase 1 (modular monolith, everything in-process), complex cross-context flows are orchestrated by **Application Services** that act as lightweight Sagas:

```python
# Example: Post-Licence Handoff Saga
class PostLicenceHandoffSaga:
    """Orchestrates the creation of compliance infrastructure after licence grant."""

    def __init__(self, compliance_service, cmp_service, workflow_service):
        self._compliance = compliance_service
        self._cmp = cmp_service
        self._workflow = workflow_service

    async def execute(self, case: AuthorisationCase, client: Client) -> HandoffResult:
        results = []
        results.append(await self._compliance.create_calendar(client))
        results.append(await self._cmp.create_schedule(client))
        results.append(await self._compliance.create_registers(client))
        results.append(await self._workflow.create_training_plan(client))
        # If any step fails, log partial completion and create remediation task
        return HandoffResult(completed=results)
```

**Key Sagas:**
- `PostLicenceHandoffSaga` — Authorisations → Compliance + CMP + Workflow
- `TrueMatchEscalationSaga` — AML → Compliance (incident) + Workflow (tasks)
- `HighImpactRegulatoryChangeSaga` — RegIntel → Compliance + CMP + Workflow + Notifications
- `PepEscalationSaga` — Client/Person → AML (risk recalculation) + Compliance (register update)

### Dashboard Read Model (Materialized Views)

The Founder Dashboard requires cross-context data. Instead of violating bounded context boundaries with cross-joins, use PostgreSQL materialized views:

```sql
-- Refreshed every 5 minutes or on key events
CREATE MATERIALIZED VIEW mv_founder_dashboard AS
SELECT
    t.id as tenant_id,
    (SELECT count(*) FROM aml_onboarding_cases o WHERE o.tenant_id = t.id AND o.stage NOT IN ('COMPLETED', 'REJECTED')) as open_onboardings,
    (SELECT count(*) FROM crm_clients c WHERE c.tenant_id = t.id AND c.risk_tier IN ('HIGH', 'VERY_HIGH')) as high_risk_clients,
    (SELECT count(*) FROM wf_tasks tk WHERE tk.tenant_id = t.id AND tk.status = 'OVERDUE') as overdue_tasks,
    (SELECT count(*) FROM cpl_incidents i WHERE i.tenant_id = t.id AND i.status != 'CLOSED') as open_incidents
FROM tenants t WHERE t.status = 'ACTIVE';

-- Refresh concurrently (non-blocking)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_founder_dashboard;
```

---

## Domain Event Flow (Key Scenarios)

### Scenario: Client Onboarding (Happy Path)
```
1. ClientCreated → AML Onboarding context listens
2. OnboardingCaseCreated → Task created for analyst
3. ScreeningCompleted(CLEAR) → advance to RISK_SCORING
4. RiskScoreSubmitted(MEDIUM) → advance to DUE_DILIGENCE
5. (all checklist items RECEIVED) → advance to FILE_COMPLETION
6. OnboardingApproved → ClientStatusChanged(ACTIVE)
7. ClientStatusChanged(ACTIVE) → Compliance Framework creates initial registers
```

### Scenario: True Match on Screening
```
1. ScreeningHitRecorded(list=OFAC, score=95%)
2. TrueMatchConfirmed → OnboardingCase stopped
3. EscalationTriggered → CO/MLRO notified (same day)
4. IncidentCreated(severity=CRITICAL) in Compliance context
5. RegulatoryNotificationRequired → founder review
```

### Scenario: Regulatory Change Impact
```
1. RegulatoryChangeDetected(regulator=DFSA, category=RULE_CHANGE)
2. ImpactAssessmentDrafted(impact=HIGH) by AI agent
3. ImpactAssessmentApproved by founder
4. DownstreamTriggersCreated:
   → TaskCreated(POLICY_REVIEW) in Compliance
   → TaskCreated(TRAINING) in Compliance
   → ClientAlert(DRAFT) in Regulatory Intelligence
   → TaskCreated(CMP_UPDATE) in CMP
   → BoardPackFlag in Compliance
```

### Scenario: Licence Granted
```
1. LicenceGranted(case_id, client_id) in Authorisations
2. Compliance Framework receives event:
   → Create compliance calendar
   → Create CMP schedule
   → Create registers
   → Create training plan
   → Create regulatory returns schedule
   → Create rescreening schedule
3. Entity Management: create/update entity record
```
