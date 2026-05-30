# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Environmental Health & Safety (EHS) · Created: 2026-05-22

## Philosophy

This model follows classical third-normal-form (3NF) relational design. Every concept in the EHS domain — incidents, inspections, chemicals, training records, emissions, corrective actions — gets its own dedicated table with explicit foreign key relationships. Reference data (hazard categories, injury types, GHS classifications, regulatory frameworks) is modelled as lookup tables rather than embedded enums, making the system extensible without schema changes when new regulations or classification systems emerge.

The approach mirrors how enterprise EHS platforms like VelocityEHS and Intelex structure their data internally: a stable, query-friendly schema where any cross-entity report (e.g., "show all incidents at sites that also have open CAPA items and overdue training certifications") can be answered with standard SQL joins. Audit history is maintained through separate `_audit` tables rather than event streams, keeping the operational schema clean while satisfying ISO 45001 and OSHA recordkeeping requirements.

This is the most conventional choice and will be immediately familiar to any backend developer with SQL experience. It works best when the domain is well-understood (which EHS is — decades of regulatory frameworks have stabilised the core entities) and when data integrity and referential consistency are paramount.

**Best for:** Teams prioritising data integrity, regulatory compliance, and complex cross-entity reporting in a well-understood domain.

**Trade-offs:**
- Pro: Maximum referential integrity — foreign keys prevent orphaned records
- Pro: Standard SQL tooling, ORMs, and reporting tools work out of the box
- Pro: Straightforward to map to OSHA 300/300A/301 export formats
- Pro: Easy to reason about for new developers
- Con: High table count (~65-75 tables) increases schema complexity
- Con: Schema migrations required for new entity types or regulatory fields
- Con: Audit history via separate tables is less flexible than event sourcing
- Con: Jurisdiction-specific field variations require nullable columns or extension tables
- Con: Many-to-many junction tables add query complexity

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 45001:2018 | Hazard identification, risk assessment, incident investigation, and CAPA workflows map directly to dedicated tables with ISO-aligned status enums |
| ISO 14001:2015 | Environmental aspects, legal compliance obligations, and improvement programmes each have normalised tables |
| OSHA 29 CFR 1904 | OSHA 300 Log, 300A Summary, and 301 Incident Report fields are modelled as dedicated columns in `osha_300_entry` and related tables, enabling direct ITA CSV/API export |
| GHS Rev.10 | 16-section SDS structure modelled across `sds_document`, `sds_section`, and `chemical_hazard_classification` tables with GHS category codes as reference data |
| REACH / RoHS | Substance tracking with SVHC flag and regulatory status in `chemical_substance` table |
| GHG Protocol | Scope 1/2/3 emissions modelled in `emission_record` with source categorisation and emission factor reference tables |
| GRI Standards | GRI indicator codes stored in `esg_metric_definition` reference table for mapping collected data to GRI 403 (OHS) and GRI 305 (Emissions) |
| ESRS / CSRD | ESG data points mapped to ESRS disclosure requirements via `esg_framework_mapping` |
| ISO 3166 | Country and subdivision codes used in `jurisdiction` and `site` tables |
| CAS Registry | CAS numbers as unique identifiers in `chemical_substance` table |

---

## Multi-Tenancy & Access Control

```sql
-- ============================================================
-- TENANT & ORGANISATION
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    legal_name      VARCHAR(255),
    industry_code   VARCHAR(20),  -- NAICS or SIC code
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organisation_tenant ON organisation(tenant_id);

-- ============================================================
-- USERS & RBAC
-- ============================================================

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local', -- local, oidc, saml
    external_id     VARCHAR(255),  -- SSO subject identifier
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    is_system       BOOLEAN NOT NULL DEFAULT false,  -- system roles cannot be deleted
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(100) NOT NULL UNIQUE,  -- e.g. 'incident.create', 'audit.approve'
    description     TEXT,
    module          VARCHAR(50) NOT NULL  -- incident, audit, chemical, training, environmental, esg
);

CREATE TABLE role_permission (
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permission(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    site_id         UUID,  -- NULL = all sites; set = scoped to specific site
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id, COALESCE(site_id, '00000000-0000-0000-0000-000000000000'))
);
```

## Location Hierarchy

```sql
-- ============================================================
-- SITES & LOCATIONS (hierarchical)
-- ============================================================

CREATE TABLE jurisdiction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    CHAR(2) NOT NULL,       -- ISO 3166-1 alpha-2
    subdivision_code VARCHAR(6),            -- ISO 3166-2
    name            VARCHAR(255) NOT NULL,
    regulatory_body VARCHAR(255),           -- e.g. 'OSHA', 'HSE UK', 'Safe Work Australia'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdiction(id),
    name            VARCHAR(255) NOT NULL,
    site_code       VARCHAR(50),
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) NOT NULL,       -- ISO 3166-1
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    naics_code      VARCHAR(10),            -- for OSHA ITA submission
    establishment_size INTEGER,             -- employee count for OSHA threshold
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_site_tenant ON site(tenant_id);
CREATE INDEX idx_site_organisation ON site(organisation_id);

CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES site(id),
    parent_id       UUID REFERENCES location(id),  -- adjacency list for hierarchy
    name            VARCHAR(255) NOT NULL,
    location_type   VARCHAR(50) NOT NULL,  -- building, floor, area, zone, line
    location_code   VARCHAR(50),
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_location_site ON location(site_id);
CREATE INDEX idx_location_parent ON location(parent_id);
```

## Incident Management

```sql
-- ============================================================
-- INCIDENTS & NEAR-MISSES
-- ============================================================

CREATE TABLE incident_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(50) NOT NULL,  -- injury, illness, near_miss, property_damage, environmental
    is_osha_recordable BOOLEAN NOT NULL DEFAULT false
);

CREATE TABLE body_part (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(100) NOT NULL,
    body_region     VARCHAR(50) NOT NULL  -- head, torso, upper_extremity, lower_extremity
);

CREATE TABLE injury_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL  -- e.g. fracture, laceration, burn, sprain
);

CREATE TABLE incident (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    location_id     UUID REFERENCES location(id),
    incident_type_id UUID NOT NULL REFERENCES incident_type(id),
    reference_number VARCHAR(50) NOT NULL,  -- human-readable, auto-generated
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    occurred_at     TIMESTAMPTZ NOT NULL,
    reported_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    reported_by     UUID NOT NULL REFERENCES app_user(id),
    severity        VARCHAR(20) NOT NULL,   -- minor, moderate, serious, critical, fatal
    psif_potential  BOOLEAN NOT NULL DEFAULT false,  -- Potential Serious Injury/Fatality
    status          VARCHAR(30) NOT NULL DEFAULT 'reported',
        -- reported, under_investigation, root_cause_identified, corrective_action_assigned, closed
    assigned_to     UUID REFERENCES app_user(id),
    closed_at       TIMESTAMPTZ,
    closed_by       UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_incident_tenant ON incident(tenant_id);
CREATE INDEX idx_incident_site ON incident(site_id);
CREATE INDEX idx_incident_status ON incident(status);
CREATE INDEX idx_incident_occurred ON incident(occurred_at);
CREATE INDEX idx_incident_type ON incident(incident_type_id);

CREATE TABLE incident_involved_person (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id) ON DELETE CASCADE,
    person_name     VARCHAR(255) NOT NULL,
    job_title       VARCHAR(255),
    employee_id     VARCHAR(100),
    person_type     VARCHAR(50) NOT NULL,  -- employee, contractor, visitor, public
    injury_type_id  UUID REFERENCES injury_type(id),
    body_part_id    UUID REFERENCES body_part(id),
    days_away       INTEGER DEFAULT 0,
    days_restricted INTEGER DEFAULT 0,
    outcome         VARCHAR(50),  -- first_aid, medical_treatment, lost_time, fatality
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_involved_person_incident ON incident_involved_person(incident_id);

CREATE TABLE incident_attachment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id) ON DELETE CASCADE,
    file_name       VARCHAR(500) NOT NULL,
    file_type       VARCHAR(100),
    file_size_bytes BIGINT,
    storage_key     VARCHAR(1000) NOT NULL,  -- S3/GCS/local path
    uploaded_by     UUID NOT NULL REFERENCES app_user(id),
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- ROOT CAUSE ANALYSIS
-- ============================================================

CREATE TABLE root_cause_method (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(100) NOT NULL  -- five_why, fishbone, fault_tree, bowtie, icam
);

CREATE TABLE investigation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id),
    method_id       UUID NOT NULL REFERENCES root_cause_method(id),
    lead_investigator UUID NOT NULL REFERENCES app_user(id),
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    findings        TEXT,
    root_cause_summary TEXT,
    ai_suggested_causes TEXT,  -- AI-generated suggestions stored for reference
    status          VARCHAR(30) NOT NULL DEFAULT 'in_progress',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_investigation_incident ON investigation(incident_id);

CREATE TABLE investigation_finding (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    investigation_id UUID NOT NULL REFERENCES investigation(id) ON DELETE CASCADE,
    finding_type    VARCHAR(50) NOT NULL,  -- direct_cause, contributing_factor, root_cause, systemic_issue
    description     TEXT NOT NULL,
    sequence_order  INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Corrective & Preventive Actions (CAPA)

```sql
-- ============================================================
-- CAPA
-- ============================================================

CREATE TABLE capa (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    reference_number VARCHAR(50) NOT NULL,
    source_type     VARCHAR(50) NOT NULL,  -- incident, audit, inspection, risk_assessment, complaint, regulatory
    source_id       UUID,  -- polymorphic: points to incident.id, audit.id, etc.
    capa_type       VARCHAR(20) NOT NULL,  -- corrective, preventive
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    priority        VARCHAR(20) NOT NULL,  -- low, medium, high, critical
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
        -- open, in_progress, pending_verification, verified_effective, closed, overdue
    assigned_to     UUID REFERENCES app_user(id),
    due_date        DATE NOT NULL,
    completed_at    TIMESTAMPTZ,
    verified_at     TIMESTAMPTZ,
    verified_by     UUID REFERENCES app_user(id),
    effectiveness_check_date DATE,
    effectiveness_result VARCHAR(50),  -- effective, partially_effective, ineffective
    site_id         UUID REFERENCES site(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_capa_tenant ON capa(tenant_id);
CREATE INDEX idx_capa_status ON capa(status);
CREATE INDEX idx_capa_due_date ON capa(due_date);
CREATE INDEX idx_capa_assigned ON capa(assigned_to);
CREATE INDEX idx_capa_source ON capa(source_type, source_id);

CREATE TABLE capa_comment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    capa_id         UUID NOT NULL REFERENCES capa(id) ON DELETE CASCADE,
    author_id       UUID NOT NULL REFERENCES app_user(id),
    body            TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Audits & Inspections

```sql
-- ============================================================
-- AUDITS & INSPECTIONS
-- ============================================================

CREATE TABLE audit_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    audit_type      VARCHAR(50) NOT NULL,  -- internal, external, regulatory, self_inspection
    standard_ref    VARCHAR(100),  -- e.g. 'ISO 45001:2018', 'OSHA 1910'
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_template_section (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id     UUID NOT NULL REFERENCES audit_template(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    sort_order      INTEGER NOT NULL,
    description     TEXT
);

CREATE TABLE audit_template_question (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    section_id      UUID NOT NULL REFERENCES audit_template_section(id) ON DELETE CASCADE,
    question_text   TEXT NOT NULL,
    response_type   VARCHAR(30) NOT NULL,  -- yes_no, scale, text, photo, multi_choice, numeric
    is_required     BOOLEAN NOT NULL DEFAULT true,
    sort_order      INTEGER NOT NULL,
    guidance_notes  TEXT
);

CREATE TABLE audit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    template_id     UUID NOT NULL REFERENCES audit_template(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    location_id     UUID REFERENCES location(id),
    title           VARCHAR(500) NOT NULL,
    auditor_id      UUID NOT NULL REFERENCES app_user(id),
    scheduled_date  DATE,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    status          VARCHAR(30) NOT NULL DEFAULT 'scheduled',
        -- scheduled, in_progress, completed, cancelled
    overall_score   DECIMAL(5,2),
    summary         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant ON audit(tenant_id);
CREATE INDEX idx_audit_site ON audit(site_id);
CREATE INDEX idx_audit_status ON audit(status);

CREATE TABLE audit_response (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    audit_id        UUID NOT NULL REFERENCES audit(id) ON DELETE CASCADE,
    question_id     UUID NOT NULL REFERENCES audit_template_question(id),
    response_value  TEXT,
    score           DECIMAL(5,2),
    notes           TEXT,
    photo_key       VARCHAR(1000),  -- attachment storage key
    flagged         BOOLEAN NOT NULL DEFAULT false,  -- non-conformance flag
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_response_audit ON audit_response(audit_id);
```

## Risk Assessment

```sql
-- ============================================================
-- RISK ASSESSMENT
-- ============================================================

CREATE TABLE risk_matrix (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    grid_size       INTEGER NOT NULL DEFAULT 5,  -- 3x3, 4x4, or 5x5
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE risk_matrix_cell (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matrix_id       UUID NOT NULL REFERENCES risk_matrix(id) ON DELETE CASCADE,
    likelihood      INTEGER NOT NULL,  -- 1-5
    severity        INTEGER NOT NULL,  -- 1-5
    risk_level      VARCHAR(20) NOT NULL,  -- low, moderate, high, critical
    colour_code     VARCHAR(7),  -- hex colour for UI
    UNIQUE(matrix_id, likelihood, severity)
);

CREATE TABLE hazard_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    domain          VARCHAR(50) NOT NULL  -- safety, health, environmental, ergonomic, psychosocial
);

CREATE TABLE risk_assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    location_id     UUID REFERENCES location(id),
    title           VARCHAR(500) NOT NULL,
    assessment_type VARCHAR(50) NOT NULL,  -- jsa, hra, process_hazard, environmental_aspect
    matrix_id       UUID NOT NULL REFERENCES risk_matrix(id),
    assessor_id     UUID NOT NULL REFERENCES app_user(id),
    assessment_date DATE NOT NULL,
    review_date     DATE,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
        -- draft, in_review, approved, archived
    approved_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_risk_assessment_tenant ON risk_assessment(tenant_id);
CREATE INDEX idx_risk_assessment_site ON risk_assessment(site_id);

CREATE TABLE risk_assessment_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assessment_id   UUID NOT NULL REFERENCES risk_assessment(id) ON DELETE CASCADE,
    hazard_category_id UUID REFERENCES hazard_category(id),
    hazard_description TEXT NOT NULL,
    existing_controls TEXT,
    initial_likelihood INTEGER NOT NULL,
    initial_severity INTEGER NOT NULL,
    initial_risk_level VARCHAR(20) NOT NULL,
    additional_controls TEXT,
    residual_likelihood INTEGER,
    residual_severity INTEGER,
    residual_risk_level VARCHAR(20),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Chemical Management & SDS

```sql
-- ============================================================
-- CHEMICAL MANAGEMENT
-- ============================================================

CREATE TABLE chemical_substance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cas_number      VARCHAR(20),         -- CAS Registry Number
    ec_number       VARCHAR(20),         -- EINECS/ELINCS number (EU)
    name            VARCHAR(500) NOT NULL,
    molecular_formula VARCHAR(200),
    is_svhc         BOOLEAN NOT NULL DEFAULT false,  -- REACH Substance of Very High Concern
    reach_status    VARCHAR(50),         -- registered, pre-registered, restricted, authorised
    rohs_restricted BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_substance_cas ON chemical_substance(cas_number) WHERE cas_number IS NOT NULL;

CREATE TABLE ghs_hazard_statement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(10) NOT NULL UNIQUE,   -- e.g. H200, H301
    phrase          TEXT NOT NULL,
    hazard_class    VARCHAR(100) NOT NULL,
    category        VARCHAR(50),
    signal_word     VARCHAR(20)  -- danger, warning
);

CREATE TABLE ghs_precautionary_statement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(10) NOT NULL UNIQUE,   -- e.g. P201, P301
    phrase          TEXT NOT NULL,
    statement_type  VARCHAR(20) NOT NULL  -- prevention, response, storage, disposal
);

CREATE TABLE sds_document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    substance_id    UUID REFERENCES chemical_substance(id),
    product_name    VARCHAR(500) NOT NULL,
    manufacturer    VARCHAR(255),
    revision_date   DATE,
    version         VARCHAR(20),
    language        VARCHAR(10) NOT NULL DEFAULT 'en',
    file_storage_key VARCHAR(1000),  -- original PDF location
    status          VARCHAR(30) NOT NULL DEFAULT 'active',  -- active, superseded, archived
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sds_tenant ON sds_document(tenant_id);
CREATE INDEX idx_sds_substance ON sds_document(substance_id);

CREATE TABLE sds_hazard_mapping (
    sds_id          UUID NOT NULL REFERENCES sds_document(id) ON DELETE CASCADE,
    hazard_stmt_id  UUID NOT NULL REFERENCES ghs_hazard_statement(id),
    PRIMARY KEY (sds_id, hazard_stmt_id)
);

CREATE TABLE sds_precaution_mapping (
    sds_id          UUID NOT NULL REFERENCES sds_document(id) ON DELETE CASCADE,
    precaution_id   UUID NOT NULL REFERENCES ghs_precautionary_statement(id),
    PRIMARY KEY (sds_id, precaution_id)
);

CREATE TABLE chemical_inventory (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    location_id     UUID REFERENCES location(id),
    sds_id          UUID NOT NULL REFERENCES sds_document(id),
    product_name    VARCHAR(500) NOT NULL,
    quantity        DECIMAL(12,3),
    unit            VARCHAR(20),  -- kg, L, gal, lb
    max_quantity    DECIMAL(12,3),  -- for EPCRA threshold tracking
    epcra_tpq       DECIMAL(12,3),  -- Threshold Planning Quantity
    last_verified   DATE,
    status          VARCHAR(30) NOT NULL DEFAULT 'in_use',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_chem_inventory_site ON chemical_inventory(site_id);
```

## Training & Certification

```sql
-- ============================================================
-- TRAINING MANAGEMENT
-- ============================================================

CREATE TABLE training_course (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    course_type     VARCHAR(50) NOT NULL,  -- classroom, online, on_the_job, blended
    regulatory_ref  VARCHAR(200),  -- e.g. 'OSHA 1910.120', 'HAZWOPER'
    validity_months INTEGER,  -- NULL = no expiry; otherwise recertification interval
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE training_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    course_id       UUID NOT NULL REFERENCES training_course(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    site_id         UUID REFERENCES site(id),
    completed_date  DATE NOT NULL,
    expiry_date     DATE,  -- computed from completed_date + validity_months
    score           DECIMAL(5,2),
    pass_fail       VARCHAR(10),  -- pass, fail, n/a
    certificate_key VARCHAR(1000),  -- file storage
    instructor      VARCHAR(255),
    status          VARCHAR(30) NOT NULL DEFAULT 'valid',
        -- valid, expiring_soon, expired, revoked
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_training_record_user ON training_record(user_id);
CREATE INDEX idx_training_record_expiry ON training_record(expiry_date);
CREATE INDEX idx_training_record_course ON training_record(course_id);
```

## OSHA Recordkeeping

```sql
-- ============================================================
-- OSHA 300 LOG & ITA SUBMISSION
-- ============================================================

CREATE TABLE osha_300_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    incident_id     UUID REFERENCES incident(id),
    calendar_year   INTEGER NOT NULL,
    case_number     VARCHAR(20) NOT NULL,           -- Column A
    employee_name   VARCHAR(255),                   -- Column B (or 'Privacy Case')
    is_privacy_case BOOLEAN NOT NULL DEFAULT false,
    job_title       VARCHAR(255),                   -- Column C
    date_of_injury  DATE NOT NULL,                  -- Column D
    where_occurred  VARCHAR(500),                   -- Column E
    description     TEXT,                           -- Column F
    -- Columns G-J: classify the case
    resulted_in_death BOOLEAN NOT NULL DEFAULT false,          -- Column G
    days_away_from_work BOOLEAN NOT NULL DEFAULT false,        -- Column H
    job_transfer_restriction BOOLEAN NOT NULL DEFAULT false,   -- Column I
    other_recordable BOOLEAN NOT NULL DEFAULT false,           -- Column J
    -- Columns K-L: days counted
    num_days_away   INTEGER DEFAULT 0,
    num_days_restricted INTEGER DEFAULT 0,
    -- Column M: injury/illness type
    injury_illness_type VARCHAR(10),  -- (1)-(6) per OSHA classification
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_osha300_site_year ON osha_300_entry(site_id, calendar_year);

CREATE TABLE osha_300a_summary (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    calendar_year   INTEGER NOT NULL,
    avg_employees   INTEGER,
    total_hours_worked BIGINT,
    -- Totals from 300 Log
    total_deaths    INTEGER DEFAULT 0,
    total_days_away INTEGER DEFAULT 0,
    total_transfers INTEGER DEFAULT 0,
    total_other     INTEGER DEFAULT 0,
    total_injuries  INTEGER DEFAULT 0,
    total_skin      INTEGER DEFAULT 0,
    total_respiratory INTEGER DEFAULT 0,
    total_poisoning INTEGER DEFAULT 0,
    total_hearing   INTEGER DEFAULT 0,
    total_other_illness INTEGER DEFAULT 0,
    -- Submission
    ita_submitted   BOOLEAN NOT NULL DEFAULT false,
    ita_submitted_at TIMESTAMPTZ,
    ita_confirmation_id VARCHAR(100),
    certified_by    UUID REFERENCES app_user(id),
    certified_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(site_id, calendar_year)
);
```

## Environmental Compliance & ESG

```sql
-- ============================================================
-- ENVIRONMENTAL COMPLIANCE
-- ============================================================

CREATE TABLE environmental_permit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    permit_type     VARCHAR(50) NOT NULL,  -- air, water, waste, stormwater, other
    permit_number   VARCHAR(100),
    issuing_authority VARCHAR(255),
    issued_date     DATE,
    expiry_date     DATE,
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    conditions_summary TEXT,
    file_storage_key VARCHAR(1000),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE permit_obligation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    permit_id       UUID NOT NULL REFERENCES environmental_permit(id) ON DELETE CASCADE,
    obligation_text TEXT NOT NULL,
    frequency       VARCHAR(50),  -- daily, weekly, monthly, quarterly, annually
    next_due_date   DATE,
    assigned_to     UUID REFERENCES app_user(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE waste_manifest (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    manifest_number VARCHAR(100) NOT NULL,
    waste_type      VARCHAR(50) NOT NULL,  -- hazardous, non_hazardous, universal, special
    waste_code      VARCHAR(20),  -- EPA hazardous waste code (e.g. D001)
    quantity        DECIMAL(12,3) NOT NULL,
    unit            VARCHAR(20) NOT NULL,
    generator_name  VARCHAR(255),
    transporter_name VARCHAR(255),
    disposal_facility VARCHAR(255),
    shipped_date    DATE,
    received_date   DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- ESG & EMISSIONS
-- ============================================================

CREATE TABLE emission_source_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    scope           INTEGER NOT NULL CHECK (scope IN (1, 2, 3)),  -- GHG Protocol scope
    category        VARCHAR(100)  -- stationary_combustion, mobile, purchased_electricity, etc.
);

CREATE TABLE emission_factor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_type_id  UUID NOT NULL REFERENCES emission_source_type(id),
    fuel_or_activity VARCHAR(255) NOT NULL,
    factor_value    DECIMAL(15,6) NOT NULL,  -- kg CO2e per unit
    unit            VARCHAR(50) NOT NULL,    -- per_kwh, per_litre, per_kg, per_mile
    region          VARCHAR(100),
    valid_from      DATE NOT NULL,
    valid_to        DATE,
    data_source     VARCHAR(255),  -- EPA, DEFRA, IEA
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE emission_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    source_type_id  UUID NOT NULL REFERENCES emission_source_type(id),
    emission_factor_id UUID REFERENCES emission_factor(id),
    reporting_period_start DATE NOT NULL,
    reporting_period_end DATE NOT NULL,
    activity_value  DECIMAL(15,3) NOT NULL,
    activity_unit   VARCHAR(50) NOT NULL,
    co2e_tonnes     DECIMAL(15,6) NOT NULL,
    data_quality    VARCHAR(20),  -- measured, calculated, estimated
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_emission_site_period ON emission_record(site_id, reporting_period_start);

CREATE TABLE esg_metric_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(100) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    framework       VARCHAR(50) NOT NULL,  -- gri, esrs, issb, cdp, tcfd
    framework_ref   VARCHAR(100),          -- e.g. 'GRI 403-9', 'ESRS S1-14'
    unit            VARCHAR(50),
    data_type       VARCHAR(30) NOT NULL,  -- numeric, text, boolean, date
    description     TEXT
);

CREATE TABLE esg_metric_value (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    metric_id       UUID NOT NULL REFERENCES esg_metric_definition(id),
    site_id         UUID REFERENCES site(id),  -- NULL = org-wide
    reporting_year  INTEGER NOT NULL,
    value_numeric   DECIMAL(15,4),
    value_text      TEXT,
    data_source     VARCHAR(255),
    verified        BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_esg_value_tenant_year ON esg_metric_value(tenant_id, reporting_year);
```

## Audit Trail

```sql
-- ============================================================
-- AUDIT TRAIL (for all tables)
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    action          VARCHAR(10) NOT NULL,  -- INSERT, UPDATE, DELETE
    old_values      JSONB,
    new_values      JSONB,
    changed_by      UUID,
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    ip_address      INET,
    user_agent      TEXT
);

CREATE INDEX idx_audit_log_table_record ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id);
CREATE INDEX idx_audit_log_changed_at ON audit_log(changed_at);

-- Example trigger function for automatic audit logging:
-- CREATE OR REPLACE FUNCTION audit_trigger_func() RETURNS TRIGGER AS $$
-- BEGIN
--   INSERT INTO audit_log (tenant_id, table_name, record_id, action, old_values, new_values, changed_by)
--   VALUES (
--     COALESCE(NEW.tenant_id, OLD.tenant_id),
--     TG_TABLE_NAME,
--     COALESCE(NEW.id, OLD.id),
--     TG_OP,
--     CASE WHEN TG_OP IN ('UPDATE','DELETE') THEN to_jsonb(OLD) END,
--     CASE WHEN TG_OP IN ('INSERT','UPDATE') THEN to_jsonb(NEW) END,
--     current_setting('app.current_user_id', true)::UUID
--   );
--   RETURN COALESCE(NEW, OLD);
-- END;
-- $$ LANGUAGE plpgsql;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-tenancy & RBAC | 6 | tenant, organisation, app_user, role, permission, role_permission, user_role |
| Location hierarchy | 3 | jurisdiction, site, location |
| Incident management | 4 | incident, incident_involved_person, incident_attachment, incident_type + 2 lookup tables |
| Investigation & root cause | 3 | investigation, investigation_finding, root_cause_method |
| CAPA | 2 | capa, capa_comment |
| Audits & inspections | 5 | audit_template, section, question, audit, audit_response |
| Risk assessment | 5 | risk_matrix, risk_matrix_cell, hazard_category, risk_assessment, risk_assessment_item |
| Chemical management | 8 | chemical_substance, ghs_hazard/precautionary statements, sds_document, mappings, chemical_inventory |
| Training | 2 | training_course, training_record |
| OSHA recordkeeping | 2 | osha_300_entry, osha_300a_summary |
| Environmental compliance | 3 | environmental_permit, permit_obligation, waste_manifest |
| ESG & emissions | 5 | emission_source_type, emission_factor, emission_record, esg_metric_definition, esg_metric_value |
| Audit trail | 1 | audit_log |
| **Total** | **~49 core tables** | Plus lookup/reference tables |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation for offline-first mobile clients syncing later, and avoids sequential ID enumeration attacks.

2. **Tenant-scoped with `tenant_id` on all operational tables** — supports PostgreSQL Row-Level Security policies for data isolation. Reference data (GHS statements, OSHA injury types) is shared across tenants.

3. **Adjacency list for location hierarchy** — simple parent_id self-reference on `location`. Supports recursive CTE queries for "all incidents in Building 3 and all its sub-locations." Can be upgraded to ltree if query performance demands it.

4. **Polymorphic CAPA source** — `source_type` + `source_id` allows CAPAs to originate from incidents, audits, inspections, or risk assessments without separate junction tables per source. Trade-off: no foreign key constraint on `source_id`.

5. **OSHA 300/300A as dedicated tables** — rather than deriving from incident data at query time, these tables store the OSHA-specific representation directly, matching the ITA CSV/API submission format field-for-field. This ensures regulatory export accuracy even if the incident record is later amended.

6. **GHS reference data as normalised lookup tables** — all GHS hazard statements (H-codes) and precautionary statements (P-codes) are stored as rows, with many-to-many mappings to SDS documents. This allows regulatory updates (new GHS revision) by inserting new rows rather than schema changes.

7. **Emission factors as versioned reference data** — `valid_from`/`valid_to` ranges on emission factors allow historical calculations to use the factor that was correct at the time, supporting GHG Protocol requirements for consistent year-over-year reporting.

8. **Audit trail via generic `audit_log` table with trigger** — a single table captures all changes across the schema. JSONB `old_values`/`new_values` columns store the full row state before and after changes, satisfying ISO 45001 documentation requirements. Trade-off: less queryable than event sourcing for "what was the state at time X" questions.

9. **Separate `incident_involved_person` table** — allows multiple people to be involved in a single incident with different injury outcomes, matching OSHA Form 301 requirements where each injured person gets a separate report.

10. **ESG metrics mapped to multiple frameworks** — `esg_metric_definition` stores framework-specific codes (GRI, ESRS, ISSB), allowing the same collected data to be reported against multiple frameworks without duplication.
