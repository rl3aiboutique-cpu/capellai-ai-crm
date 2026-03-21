# Compliance Brain - Data Model

> Core database schema for PostgreSQL 16. Multi-tenant with Row-Level Security.
> All tables include `tenant_id` for isolation. UUIDs for primary keys.

---

## Entity Relationship Overview

```
PHASE 1 TABLES:
═══════════════

tenants ◄────────────────────────────────────────────────────────┐
  │                                                               │
  ├─► users ──► user_tenant_assignments                           │
  │                                                               │
  │  (tenant_id on ALL tables below)                              │
  │                                                               │
  ├─► crm_clients ◄──────────────────────────────────┐           │
  │     ├─► crm_client_team_assignments               │           │
  │     ├─► crm_client_documents ──► crm_doc_versions │           │
  │     ├─► crm_ubo_links ──► crm_persons             │           │
  │     └─► aml_onboarding_cases                      │           │
  │           ├─► aml_screening_cases ──► aml_hits     │           │
  │           ├─► aml_risk_scores                      │           │
  │           └─► aml_checklist_items                  │           │
  │                                                    │           │
  ├─► cpl_policies                                     │           │
  ├─► cpl_registers ──► cpl_register_entries           │           │
  ├─► cpl_incidents                                    │           │
  │                                                    │           │
  ├─► doc_templates ──► doc_generated                  │           │
  ├─► wf_tasks ──► wf_approvals                       │           │
  ├─► reg_changes                                      │           │
  │                                                    │           │
  ├─► light_auth_cases                                 │           │
  ├─► light_entities                                   │           │
  │                                                    │           │
  ├─► audit_entries (append-only, partitioned)         │           │
  ├─► idempotency_keys                                 │           │
  ├─► error_log                                        │           │
  ├─► agent_actions                                    │           │
  └─► mv_founder_dashboard (materialized view)         │           │
                                                       │           │
  All tables reference tenant_id ──────────────────────┘───────────┘

PHASE 2 TABLES (not implemented in Phase 1):
════════════════════════════════════════════

  auth_cases ──► auth_rfis ──► auth_rfi_questions
  ent_legal_entities ──► ent_obligations ──► ent_entity_documents
  cmp_monitoring_tests ──► cmp_scheduled_tests ──► cmp_findings
  reg_impact_assessments, reg_client_alerts
```

---

## Core Tables

### Identity & Tenancy

```sql
-- Tenants (client firms + internal entities)
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    type VARCHAR(20) NOT NULL CHECK (type IN ('CLIENT_FIRM', 'INTERNAL_ENTITY')),
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'SUSPENDED', 'ARCHIVED')),
    active_regulators JSONB NOT NULL DEFAULT '[]',
    configuration JSONB NOT NULL DEFAULT '{}',
    -- configuration includes: ubo_threshold_pct, risk_thresholds, sla_defaults, reminder_cadence, retention_years
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Users
-- Auth (login, MFA, password) managed by Supabase Auth. This table is profile + authorization only.
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    full_name VARCHAR(255) NOT NULL,
    role VARCHAR(30) NOT NULL CHECK (role IN ('MASTER_ADMIN', 'COMPLIANCE_OFFICER', 'MLRO', 'ANALYST', 'OPERATIONS', 'VIEWER')),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    supabase_auth_id VARCHAR(255) UNIQUE, -- Supabase Auth user ID
    last_login TIMESTAMPTZ,
    version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- User-Tenant assignments (many-to-many)
CREATE TABLE user_tenant_assignments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    role_override VARCHAR(30), -- optional per-tenant role override
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(user_id, tenant_id)
);
```

### Client Management (CRM)

```sql
-- Clients
CREATE TABLE crm_clients (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(255) NOT NULL,
    type VARCHAR(30) NOT NULL CHECK (type IN ('INDIVIDUAL', 'CORPORATE', 'FUND', 'FOUNDATION', 'FINTECH_VASP')),
    status VARCHAR(20) NOT NULL DEFAULT 'PROSPECT' CHECK (status IN ('PROSPECT', 'ONBOARDING', 'ACTIVE', 'UNDER_REVIEW', 'SUSPENDED', 'EXITED', 'REACTIVATION')),
    jurisdiction VARCHAR(30),
    risk_tier VARCHAR(20) CHECK (risk_tier IN ('LOW', 'MEDIUM', 'HIGH', 'VERY_HIGH', 'UNACCEPTABLE')),
    suspension_reason VARCHAR(255),
    contact_info JSONB, -- { email, phone, address }
    engagement_date DATE,
    -- assigned_team moved to junction table
    created_by UUID NOT NULL REFERENCES users(id),
    version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_crm_clients_tenant ON crm_clients(tenant_id);
CREATE INDEX idx_crm_clients_status ON crm_clients(tenant_id, status);
CREATE INDEX idx_crm_clients_risk ON crm_clients(tenant_id, risk_tier);
CREATE INDEX idx_crm_clients_name ON crm_clients USING gin(to_tsvector('english', name));

ALTER TABLE crm_clients ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON crm_clients
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- Client Team Assignments (normalized, replaces UUID array)
CREATE TABLE crm_client_team_assignments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    client_id UUID NOT NULL REFERENCES crm_clients(id),
    user_id UUID NOT NULL REFERENCES users(id),
    role VARCHAR(50), -- e.g. 'relationship_manager', 'analyst'
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(client_id, user_id)
);

ALTER TABLE crm_client_team_assignments ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON crm_client_team_assignments
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- Persons (shared entity for UBOs, directors, etc.)
CREATE TABLE crm_persons (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    full_name VARCHAR(255) NOT NULL,
    nationality VARCHAR(3), -- ISO 3166-1 alpha-3
    date_of_birth DATE,
    id_doc_type VARCHAR(30), -- PASSPORT, EMIRATES_ID, RESIDENCE_VISA, OTHER
    id_doc_number VARCHAR(100),
    id_doc_expiry DATE, -- extracted for efficient expiry queries
    id_document_extra JSONB, -- { issuing_country, issue_date, file_id, additional docs }
    address JSONB, -- { line1, line2, city, state, country, postal_code }
    contact JSONB, -- { email, phone }
    occupation VARCHAR(255),
    is_pep BOOLEAN NOT NULL DEFAULT FALSE,
    pep_details JSONB, -- { pep_type, position, country, source, identified_date }
    screening_status JSONB, -- { last_screened_at, result, next_screening_due }
    version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_crm_persons_tenant ON crm_persons(tenant_id);
CREATE INDEX idx_crm_persons_name ON crm_persons USING gin(to_tsvector('english', full_name));
CREATE INDEX idx_crm_persons_doc_expiry ON crm_persons(tenant_id, id_doc_expiry) WHERE id_doc_expiry IS NOT NULL;
ALTER TABLE crm_persons ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON crm_persons
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- UBO Links (person to client relationship)
CREATE TABLE crm_ubo_links (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    person_id UUID NOT NULL REFERENCES crm_persons(id),
    client_id UUID NOT NULL REFERENCES crm_clients(id),
    ownership_pct DECIMAL(5,2) NOT NULL,
    role VARCHAR(100), -- e.g. "Director", "Shareholder", "Beneficiary"
    effective_date DATE NOT NULL DEFAULT CURRENT_DATE,
    end_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE crm_ubo_links ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON crm_ubo_links
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- Client Documents
CREATE TABLE crm_client_documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    client_id UUID NOT NULL REFERENCES crm_clients(id),
    category VARCHAR(30) NOT NULL CHECK (category IN ('IDENTITY', 'CORPORATE', 'OWNERSHIP', 'FINANCIAL', 'ENGAGEMENT', 'SCREENING', 'AD_HOC', 'OTHER')),
    filename VARCHAR(500) NOT NULL,
    storage_key VARCHAR(1000) NOT NULL, -- Supabase Storage path
    current_version INT NOT NULL DEFAULT 1,
    expiry_date DATE,
    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    uploaded_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE crm_client_documents ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON crm_client_documents
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- Document Versions
CREATE TABLE crm_document_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL REFERENCES crm_client_documents(id),
    version_number INT NOT NULL,
    storage_key VARCHAR(1000) NOT NULL,
    change_note TEXT,
    uploaded_by UUID NOT NULL REFERENCES users(id),
    uploaded_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(document_id, version_number)
);
```

### AML & Onboarding

```sql
-- Onboarding Cases
CREATE TABLE aml_onboarding_cases (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    client_id UUID NOT NULL REFERENCES crm_clients(id),
    stage VARCHAR(30) NOT NULL DEFAULT 'CLIENT_INTAKE' CHECK (stage IN ('CLIENT_INTAKE', 'SCREENING', 'RISK_SCORING', 'DUE_DILIGENCE', 'FILE_COMPLETION', 'APPROVAL', 'COMPLETED', 'REJECTED')),
    client_type VARCHAR(30) NOT NULL,
    dd_level VARCHAR(20) NOT NULL DEFAULT 'STANDARD_CDD' CHECK (dd_level IN ('STANDARD_CDD', 'ENHANCED_EDD')),
    assigned_analyst UUID REFERENCES users(id),
    assigned_approver UUID REFERENCES users(id),
    sla_deadline DATE,
    is_overdue BOOLEAN NOT NULL DEFAULT FALSE,
    notes JSONB DEFAULT '[]',
    created_by UUID NOT NULL REFERENCES users(id),
    version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE aml_onboarding_cases ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON aml_onboarding_cases
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- Risk Scores
CREATE TABLE aml_risk_scores (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    case_id UUID NOT NULL REFERENCES aml_onboarding_cases(id),
    client_id UUID NOT NULL REFERENCES crm_clients(id),
    total_score DECIMAL(5,2) NOT NULL,
    tier VARCHAR(20) NOT NULL CHECK (tier IN ('LOW', 'MEDIUM', 'HIGH', 'VERY_HIGH', 'UNACCEPTABLE')),
    factors JSONB NOT NULL, -- [{ name, weight, score, evidence }]
    is_override BOOLEAN NOT NULL DEFAULT FALSE,
    override_rationale TEXT,
    calculated_by VARCHAR(255) NOT NULL, -- UserId or AgentId
    approved_by UUID REFERENCES users(id),
    calculated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE aml_risk_scores ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON aml_risk_scores
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- Screening Cases
CREATE TABLE aml_screening_cases (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    subject_type VARCHAR(20) NOT NULL CHECK (subject_type IN ('CLIENT', 'UBO', 'PERSON')),
    subject_id UUID NOT NULL,
    onboarding_case_id UUID REFERENCES aml_onboarding_cases(id),
    screening_type VARCHAR(20) NOT NULL CHECK (screening_type IN ('INITIAL', 'PERIODIC', 'TRIGGERED')),
    provider VARCHAR(30) CHECK (provider IN ('WORLD_CHECK', 'COMPLY_ADVANTAGE', 'MANUAL')),
    lists_checked JSONB NOT NULL DEFAULT '[]',
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING' CHECK (status IN ('PENDING', 'CLEAR', 'HIT_PENDING', 'RESOLVED')),
    screened_by UUID REFERENCES users(id),
    screened_at TIMESTAMPTZ,
    next_screening_due DATE,
    version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE aml_screening_cases ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON aml_screening_cases
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- Screening Hits
CREATE TABLE aml_screening_hits (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    screening_case_id UUID NOT NULL REFERENCES aml_screening_cases(id),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    list VARCHAR(30) NOT NULL CHECK (list IN ('UAE_LOCAL', 'UN', 'HMT_UK', 'OFAC_USA', 'EU', 'PEP', 'ADVERSE_MEDIA')),
    matched_name VARCHAR(500),
    match_score DECIMAL(5,2),
    disposition VARCHAR(20) NOT NULL DEFAULT 'PENDING' CHECK (disposition IN ('PENDING', 'FALSE_POSITIVE', 'TRUE_MATCH')),
    rationale TEXT, -- min 3-5 sentences for FALSE_POSITIVE
    reviewed_by UUID REFERENCES users(id),
    approved_by UUID REFERENCES users(id), -- required for sanctions FALSE_POSITIVE
    reviewed_at TIMESTAMPTZ,
    raw_data JSONB, -- original screening provider data
    last_re_screened_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE aml_screening_hits ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON aml_screening_hits
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- Checklist Items
CREATE TABLE aml_checklist_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    case_id UUID NOT NULL REFERENCES aml_onboarding_cases(id),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    item_name VARCHAR(500) NOT NULL,
    required BOOLEAN NOT NULL DEFAULT TRUE,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING' CHECK (status IN ('PENDING', 'RECEIVED', 'WAIVED')),
    document_id UUID REFERENCES crm_client_documents(id),
    notes TEXT,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE aml_checklist_items ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON aml_checklist_items
    USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

### Compliance Framework

```sql
-- Policies
CREATE TABLE cpl_policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    title VARCHAR(500) NOT NULL,
    type VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'DRAFT' CHECK (status IN ('DRAFT', 'UNDER_REVIEW', 'APPROVED', 'PUBLISHED', 'ARCHIVED')),
    version_major INT NOT NULL DEFAULT 1,
    version_minor INT NOT NULL DEFAULT 0,
    content_document_id UUID, -- reference to stored document
    next_review_date DATE,
    review_triggers JSONB DEFAULT '[]',
    created_by UUID NOT NULL REFERENCES users(id),
    version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE cpl_policies ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON cpl_policies
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- Compliance Registers
CREATE TABLE cpl_registers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    type VARCHAR(50) NOT NULL,
    is_reg_submission BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Register Entries
CREATE TABLE cpl_register_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    register_id UUID NOT NULL REFERENCES cpl_registers(id),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    data JSONB NOT NULL, -- flexible schema per register type
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE cpl_register_entries ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON cpl_register_entries
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- Incidents
CREATE TABLE cpl_incidents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    title VARCHAR(500) NOT NULL,
    description TEXT,
    severity VARCHAR(20) NOT NULL CHECK (severity IN ('LOW', 'MEDIUM', 'HIGH', 'CRITICAL')),
    status VARCHAR(20) NOT NULL DEFAULT 'OPEN' CHECK (status IN ('OPEN', 'UNDER_ASSESSMENT', 'REMEDIATION', 'CLOSED', 'REOPENED')),
    regulatory_notification_required BOOLEAN,
    notification_deadline TIMESTAMPTZ,
    reported_by UUID NOT NULL REFERENCES users(id),
    version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE cpl_incidents ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON cpl_incidents
    USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

### Workflow Engine

```sql
-- Tasks
CREATE TABLE wf_tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    title VARCHAR(500) NOT NULL,
    description TEXT,
    type VARCHAR(30) NOT NULL,
    source_context_type VARCHAR(30), -- CLIENT, CASE, ENTITY, TEST, POLICY, INCIDENT
    source_context_id UUID,
    assigned_to UUID REFERENCES users(id),
    due_date DATE,
    priority VARCHAR(10) NOT NULL DEFAULT 'MEDIUM' CHECK (priority IN ('LOW', 'MEDIUM', 'HIGH', 'URGENT')),
    status VARCHAR(20) NOT NULL DEFAULT 'OPEN' CHECK (status IN ('OPEN', 'IN_PROGRESS', 'PENDING_APPROVAL', 'COMPLETED', 'OVERDUE', 'CANCELLED')),
    escalation_level INT NOT NULL DEFAULT 0,
    escalated_at TIMESTAMPTZ,
    created_by UUID NOT NULL REFERENCES users(id),
    version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_wf_tasks_assigned ON wf_tasks(tenant_id, assigned_to, status);
CREATE INDEX idx_wf_tasks_due ON wf_tasks(tenant_id, due_date) WHERE status NOT IN ('COMPLETED', 'CANCELLED');

ALTER TABLE wf_tasks ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON wf_tasks
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- Approvals
CREATE TABLE wf_approvals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id UUID NOT NULL REFERENCES wf_tasks(id),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    approver_id UUID NOT NULL REFERENCES users(id),
    decision VARCHAR(10) CHECK (decision IN ('APPROVED', 'REJECTED')),
    reason TEXT,
    decided_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE wf_approvals ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON wf_approvals
    USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

### Document Factory

```sql
-- Document Templates
CREATE TABLE doc_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID, -- NULL = global template
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL, -- ONBOARDING_CHECKLIST, DD_FORM, RISK_MATRIX, etc.
    category VARCHAR(30) NOT NULL,
    description TEXT,
    storage_key VARCHAR(1000) NOT NULL, -- path to DOCX/PPTX template file
    merge_fields JSONB DEFAULT '[]', -- [{ field_name, source_path, type }]
    version INT NOT NULL DEFAULT 1,
    updated_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Generated Documents
CREATE TABLE doc_generated (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    template_id UUID REFERENCES doc_templates(id),
    source_context_type VARCHAR(30), -- CLIENT, CASE, ENTITY, TEST
    source_context_id UUID,
    output_format VARCHAR(10) NOT NULL DEFAULT 'PDF' CHECK (output_format IN ('DOCX', 'PDF', 'PPTX', 'ZIP')),
    storage_key VARCHAR(1000) NOT NULL,
    version INT NOT NULL DEFAULT 1,
    retention_until DATE,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE doc_generated ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON doc_generated
    USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

### Regulatory Intelligence (Basic)

```sql
-- Regulatory Changes (basic tracking for Phase 1 weekly scan)
CREATE TABLE reg_changes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID, -- NULL = global (affects all tenants)
    regulator VARCHAR(30) NOT NULL, -- DFSA, FSRA, UAE_CB, VARA, DIFC, ADGM
    category VARCHAR(30) NOT NULL CHECK (category IN ('RULE_CHANGE', 'GUIDANCE', 'ENFORCEMENT', 'CONSULTATION', 'FAQ_NOTICE')),
    title VARCHAR(500) NOT NULL,
    summary TEXT,
    source_url VARCHAR(1000),
    effective_date DATE,
    implementation_deadline DATE,
    impact_level VARCHAR(10) CHECK (impact_level IN ('LOW', 'MEDIUM', 'HIGH')),
    status VARCHAR(20) NOT NULL DEFAULT 'DETECTED' CHECK (status IN ('DETECTED', 'ASSESSED', 'ACTIONED', 'ARCHIVED')),
    detected_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    assessed_by UUID REFERENCES users(id),
    assessed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_reg_changes_status ON reg_changes(status, detected_at DESC);
```

### Light Tracking (Phase 1 — CRUD only)

```sql
-- Light Auth Cases (simple tracking, promoted to full BC in Phase 2)
CREATE TABLE light_auth_cases (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    client_name VARCHAR(255) NOT NULL,
    regulator VARCHAR(30) NOT NULL,
    licence_category VARCHAR(100),
    stage VARCHAR(50) NOT NULL DEFAULT 'ENGAGEMENT',
    assigned_to UUID REFERENCES users(id),
    key_dates JSONB DEFAULT '{}', -- { engagement_date, submission_date, expected_decision }
    notes TEXT,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE light_auth_cases ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON light_auth_cases
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- Light Entities (simple tracking, promoted to full BC in Phase 2)
CREATE TABLE light_entities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL, -- SPV, FOUNDATION, FUND, HOLDING, INTERNAL
    jurisdiction VARCHAR(50),
    registration_date DATE,
    licence_expiry DATE,
    next_renewal DATE,
    directors JSONB DEFAULT '[]', -- [{ name, person_id? }]
    beneficial_owners JSONB DEFAULT '[]', -- [{ name, person_id?, ownership_pct }]
    status VARCHAR(10) NOT NULL DEFAULT 'GREEN' CHECK (status IN ('GREEN', 'AMBER', 'RED')),
    notes TEXT,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE light_entities ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON light_entities
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

CREATE INDEX idx_light_entities_renewal ON light_entities(tenant_id, next_renewal) WHERE next_renewal IS NOT NULL;
```

### Audit Trail

```sql
-- Audit Entries (append-only, partitioned by month)
CREATE TABLE audit_entries (
    id UUID NOT NULL DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    actor_type VARCHAR(20) NOT NULL,
    actor_id VARCHAR(255) NOT NULL,
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    action VARCHAR(255) NOT NULL,
    resource_type VARCHAR(100) NOT NULL,
    resource_id UUID,
    before_snapshot JSONB,
    after_snapshot JSONB,
    source VARCHAR(20) NOT NULL,
    correlation_id UUID NOT NULL,
    ip_address INET,
    metadata JSONB,
    PRIMARY KEY (id, timestamp)
) PARTITION BY RANGE (timestamp);

-- Monthly partitions (auto-create via pg_partman or migration script)
CREATE TABLE audit_entries_2026_03 PARTITION OF audit_entries
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
CREATE TABLE audit_entries_2026_04 PARTITION OF audit_entries
    FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');

-- CRITICAL: Revoke modification permissions
REVOKE UPDATE, DELETE ON audit_entries FROM app_user;

-- Indexes
CREATE INDEX idx_audit_tenant_time ON audit_entries(tenant_id, timestamp DESC);
CREATE INDEX idx_audit_resource ON audit_entries(resource_type, resource_id);
CREATE INDEX idx_audit_actor ON audit_entries(actor_id, timestamp DESC);
```

### Idempotency

```sql
-- Idempotency Keys (prevent duplicate command execution)
CREATE TABLE idempotency_keys (
    key UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    command_type VARCHAR(255) NOT NULL,
    result JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMPTZ NOT NULL DEFAULT NOW() + INTERVAL '24 hours'
);

CREATE INDEX idx_idempotency_expires ON idempotency_keys(expires_at);
-- Cleanup expired keys periodically
```

### Error Log

```sql
-- Error Log (for dashboard display and debugging)
CREATE TABLE error_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID,
    error_code VARCHAR(100) NOT NULL,
    message TEXT NOT NULL,
    details JSONB,
    stack_trace TEXT,
    correlation_id UUID,
    user_id UUID,
    endpoint VARCHAR(255),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_error_log_tenant ON error_log(tenant_id, created_at DESC);
```

### AI Agent Actions

```sql
-- Agent Actions (AI observability)
CREATE TABLE agent_actions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    agent_type VARCHAR(50) NOT NULL, -- 'risk_scoring', 'screening_triage', etc.
    action VARCHAR(100) NOT NULL,
    input_summary JSONB,
    output_summary JSONB,
    confidence_score DECIMAL(3,2), -- 0.00 to 1.00
    human_decision VARCHAR(20), -- 'ACCEPTED', 'REJECTED', 'MODIFIED'
    human_decision_by UUID REFERENCES users(id),
    processing_time_ms INT,
    tokens_used INT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_agent_actions_type ON agent_actions(tenant_id, agent_type, created_at DESC);
```

### Materialized Views (Dashboard)

```sql
-- Founder Dashboard (refreshed every 5 minutes)
CREATE MATERIALIZED VIEW mv_founder_dashboard AS
SELECT
    t.id as tenant_id,
    (SELECT count(*) FROM aml_onboarding_cases o WHERE o.tenant_id = t.id AND o.stage NOT IN ('COMPLETED', 'REJECTED')) as open_onboardings,
    (SELECT count(*) FROM crm_clients c WHERE c.tenant_id = t.id AND c.risk_tier IN ('HIGH', 'VERY_HIGH')) as high_risk_clients,
    (SELECT count(*) FROM wf_tasks tk WHERE tk.tenant_id = t.id AND tk.status = 'OVERDUE') as overdue_tasks,
    (SELECT count(*) FROM cpl_incidents i WHERE i.tenant_id = t.id AND i.status != 'CLOSED') as open_incidents,
    NOW() as refreshed_at
FROM tenants t WHERE t.status = 'ACTIVE';

CREATE UNIQUE INDEX idx_mv_dashboard_tenant ON mv_founder_dashboard(tenant_id);
-- REFRESH MATERIALIZED VIEW CONCURRENTLY mv_founder_dashboard;
```

---

## Naming Conventions

| Item | Convention | Example |
|------|-----------|---------|
| Tables | `{context_prefix}_{plural_noun}` | `crm_clients`, `aml_onboarding_cases` |
| Columns | `snake_case` | `tenant_id`, `created_at` |
| Primary keys | `id` (UUID) | Always `id` |
| Foreign keys | `{referenced_table_singular}_id` | `client_id`, `user_id` |
| Indexes | `idx_{table}_{columns}` | `idx_crm_clients_status` |
| Constraints | `chk_{table}_{column}` | `chk_crm_clients_status` |
| RLS Policies | `tenant_isolation` | Same name on every table |
| JSONB columns | Used for flexible/nested data | `contact_info`, `factors`, `configuration` |

## Context Prefixes

| Prefix | Bounded Context | Phase |
|--------|----------------|-------|
| (none) | Shared (tenants, users, audit_entries) | 1 |
| `crm_` | Client Management | 1 |
| `aml_` | AML & Onboarding | 1 |
| `cpl_` | Compliance Framework | 1 |
| `doc_` | Document Factory | 1 |
| `wf_` | Workflow Engine | 1 |
| `reg_` | Regulatory Intelligence | 1 (basic) |
| `light_` | Light Tracking (auth cases + entities) | 1 |
| `auth_` | Authorisations (full) | 2 |
| `ent_` | Entity Management (full) | 2 |
| `cmp_` | CMP Monitoring | 2 |
