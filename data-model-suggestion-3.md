# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Environmental Health & Safety (EHS) · Created: 2026-05-22

## Philosophy

The Hybrid Relational + JSONB model keeps core structural fields (IDs, foreign keys, dates, statuses, and fields required for regulatory reporting) in typed relational columns, while placing variable, jurisdiction-specific, and rapidly evolving fields in JSONB columns. This is the "relational spine with flexible edges" pattern. The relational columns handle indexing, filtering, JOINs, and regulatory exports; the JSONB columns handle the reality that EHS requirements vary dramatically by country, industry, and organisational maturity.

This pattern is particularly well suited to the EHS domain because an incident in a US chemical plant requires different data fields (OSHA 300 log fields, EPA EPCRA thresholds) than an incident in a German manufacturing facility (DGUV reporting, BetrSichV compliance) or a UK construction site (RIDDOR reporting, CDM 2015). Rather than creating a separate table or set of columns for every jurisdiction — which would create massive schema sprawl — jurisdiction-specific fields live in JSONB columns with JSON Schema validation enforced at the application layer.

This approach is used by modern multi-tenant SaaS platforms (Stripe, Shopify, Notion) that need to accommodate diverse customer configurations without per-customer schema changes. For an EHS platform targeting both US OSHA compliance and EU CSRD reporting, the hybrid approach provides regulatory reporting accuracy for known fields while maintaining extensibility for fields that vary by customer, jurisdiction, or industry. The trade-off is that JSONB fields lack database-level type enforcement and referential integrity — but this is acceptable for fields that change frequently and are validated by the application.

**Best for:** Rapid MVP development, multi-jurisdiction deployments where field requirements vary by country/industry, and organisations that need to add custom fields without database migrations.

**Trade-offs:**
- (+) Significantly fewer tables than a fully normalised model (~25 vs ~42)
- (+) Custom fields and jurisdiction-specific data require no schema migration
- (+) Faster time to MVP — core entities are simple, extensibility is built in
- (+) JSONB GIN indexes enable efficient querying of flexible fields
- (+) Well suited to multi-tenant SaaS where each organisation has different field needs
- (-) JSONB fields lack database-level type constraints and foreign keys
- (-) Complex queries on JSONB fields are slower than on indexed relational columns
- (-) JSON Schema validation must be enforced at the application layer, not the database
- (-) Reporting on JSONB fields requires understanding their structure — less self-documenting
- (-) Risk of inconsistent data if application-layer validation is bypassed

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 45001:2018 | Core incident, risk assessment, and corrective action fields are relational columns; ISO 45001 clause-specific evidence requirements stored in `compliance_data` JSONB |
| OSHA 29 CFR 1904 | All OSHA 300/301/300A required fields are named relational columns on the incident table; OSHA-specific classification stored in `regulatory_data` JSONB |
| GHS Rev.10 | Chemical identity (CAS, name) and top-level hazard classification are relational; full 16-section SDS content stored as structured JSONB |
| GHG Protocol | Scope, source type, and CO2e total are relational columns; detailed calculation inputs and methodology notes in JSONB |
| ESRS E1-E5 | ESRS data point mappings stored in `esrs_data` JSONB on environmental records, enabling flexible alignment as ESRS evolves |
| ISO 3166 | Country and subdivision codes are relational columns; jurisdiction-specific regulatory metadata in JSONB |
| RIDDOR (UK) | UK RIDDOR reporting fields stored in `regulatory_data` JSONB alongside OSHA fields, enabling multi-jurisdiction support without schema changes |
| DGUV (Germany) | German DGUV reporting fields stored in the same `regulatory_data` JSONB pattern |

---

## Core Tables

### Multi-Tenancy & Identity

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    country_code    CHAR(2) NOT NULL,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    industry_code   VARCHAR(10),
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "risk_matrix_size": 5,
    --   "osha_reporting_enabled": true,
    --   "riddor_reporting_enabled": false,
    --   "default_language": "en",
    --   "custom_incident_fields": [
    --     {"key": "weather_conditions", "label": "Weather Conditions", "type": "select", "options": ["clear","rain","snow","fog"]},
    --     {"key": "shift_type", "label": "Shift Type", "type": "select", "options": ["day","evening","night"]}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    address         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "line1": "123 Industrial Pkwy",
    --   "city": "Houston",
    --   "state": "TX",
    --   "postal_code": "77001",
    --   "country": "US",
    --   "subdivision": "US-TX",
    --   "latitude": 29.7604,
    --   "longitude": -95.3698
    -- }
    naics_code      VARCHAR(10),
    employee_count  INTEGER,
    site_metadata   JSONB NOT NULL DEFAULT '{}',   -- Site-specific custom fields
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_site_org ON site(organisation_id);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    job_title       VARCHAR(255),
    department      VARCHAR(255),
    site_id         UUID REFERENCES site(id),
    roles           TEXT[] NOT NULL DEFAULT '{}',   -- ['admin','safety_manager','inspector']
    permissions     TEXT[] NOT NULL DEFAULT '{}',   -- Derived or explicit
    profile_data    JSONB NOT NULL DEFAULT '{}',    -- Employee ID, phone, certifications
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, email)
);

CREATE INDEX idx_user_org ON app_user(organisation_id);
CREATE INDEX idx_user_roles ON app_user USING GIN (roles);
```

### Incident Management

```sql
CREATE TABLE incident (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    incident_number VARCHAR(50) NOT NULL,

    -- Core relational fields (always present, always queryable)
    incident_type   VARCHAR(50) NOT NULL,          -- 'injury','illness','near_miss','property_damage','env_release'
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    incident_date   DATE NOT NULL,
    incident_time   TIME,
    location_detail VARCHAR(500),
    status          VARCHAR(30) NOT NULL DEFAULT 'reported',
    severity        INTEGER,                       -- 1-5
    likelihood      INTEGER,                       -- 1-5
    risk_score      INTEGER,                       -- severity * likelihood

    -- OSHA-specific relational fields (always present for US reporting)
    is_osha_recordable BOOLEAN DEFAULT false,
    is_sif_potential   BOOLEAN DEFAULT false,
    osha_case_number   VARCHAR(20),
    osha_classification VARCHAR(30),               -- 'death','days_away','restricted','other_recordable'
    days_away       INTEGER DEFAULT 0,
    days_restricted INTEGER DEFAULT 0,

    -- Workflow
    reported_by     UUID NOT NULL REFERENCES app_user(id),
    investigated_by UUID REFERENCES app_user(id),
    closed_by       UUID REFERENCES app_user(id),
    closed_at       TIMESTAMPTZ,

    -- JSONB flexible fields
    persons_involved JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {
    --     "name": "Jane Smith",
    --     "role": "injured",
    --     "user_id": "uuid",
    --     "job_title": "Forklift Operator",
    --     "injury_type": "laceration",
    --     "body_part": "right_hand",
    --     "treatment": "first_aid"
    --   }
    -- ]

    root_causes     JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {
    --     "method": "5_why",
    --     "finding": "Operator had not completed refresher training",
    --     "category": "human_factor",
    --     "is_primary": true
    --   }
    -- ]

    regulatory_data JSONB NOT NULL DEFAULT '{}',
    -- US OSHA:
    -- {
    --   "osha_301": {
    --     "regular_job_duties": "...",
    --     "activity_at_time": "...",
    --     "event_sequence": "...",
    --     "object_substance": "...",
    --     "physician_name": "...",
    --     "facility_name": "..."
    --   }
    -- }
    -- UK RIDDOR:
    -- {
    --   "riddor": {
    --     "riddor_reference": "R2026-0042",
    --     "report_type": "specified_injury",
    --     "reported_to_hse": true,
    --     "hse_report_date": "2026-05-21"
    --   }
    -- }

    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- Organisation-specific fields defined in organisation.settings
    -- {
    --   "weather_conditions": "rain",
    --   "shift_type": "night",
    --   "contractor_involved": true
    -- }

    attachments     JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"type": "photo", "url": "...", "caption": "Scene photo", "uploaded_at": "..."},
    --   {"type": "document", "url": "...", "filename": "witness_statement.pdf"}
    -- ]

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, incident_number)
);

CREATE INDEX idx_incident_org_date ON incident(organisation_id, incident_date DESC);
CREATE INDEX idx_incident_site ON incident(site_id);
CREATE INDEX idx_incident_status ON incident(status);
CREATE INDEX idx_incident_type ON incident(incident_type);
CREATE INDEX idx_incident_sif ON incident(is_sif_potential) WHERE is_sif_potential = true;
CREATE INDEX idx_incident_osha ON incident(is_osha_recordable) WHERE is_osha_recordable = true;
CREATE INDEX idx_incident_persons ON incident USING GIN (persons_involved jsonb_path_ops);
CREATE INDEX idx_incident_custom ON incident USING GIN (custom_fields jsonb_path_ops);
```

### Corrective & Preventive Actions

```sql
CREATE TABLE corrective_action (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    action_number   VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    action_type     VARCHAR(20) NOT NULL,          -- 'corrective','preventive','improvement'
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium',
    status          VARCHAR(30) NOT NULL DEFAULT 'open',

    -- Source linkage (polymorphic)
    source_type     VARCHAR(30) NOT NULL,          -- 'incident','inspection','risk_assessment','audit','management_review'
    source_id       UUID NOT NULL,

    -- Assignment
    assigned_to     UUID NOT NULL REFERENCES app_user(id),
    assigned_by     UUID NOT NULL REFERENCES app_user(id),
    due_date        DATE NOT NULL,
    completed_date  DATE,
    verified_by     UUID REFERENCES app_user(id),
    verified_date   DATE,

    -- Flexible fields
    evidence        JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"type": "photo", "url": "...", "description": "After photo showing guard installed"},
    --   {"type": "document", "url": "...", "filename": "training_record.pdf"}
    -- ]

    workflow_history JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"action": "created", "by": "uuid", "at": "2026-05-20T10:00:00Z"},
    --   {"action": "assigned", "by": "uuid", "to": "uuid", "at": "2026-05-20T10:05:00Z"},
    --   {"action": "status_changed", "from": "open", "to": "in_progress", "by": "uuid", "at": "2026-05-21T09:00:00Z"}
    -- ]

    custom_fields   JSONB NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, action_number)
);

CREATE INDEX idx_capa_org_status ON corrective_action(organisation_id, status);
CREATE INDEX idx_capa_assigned ON corrective_action(assigned_to);
CREATE INDEX idx_capa_due ON corrective_action(due_date) WHERE status NOT IN ('verified', 'closed');
CREATE INDEX idx_capa_source ON corrective_action(source_type, source_id);
```

### Inspections & Audits

```sql
-- Template (structure is inherently hierarchical — good fit for JSONB)
CREATE TABLE inspection_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    template_type   VARCHAR(30) NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- The entire template structure in JSONB — sections, questions, options
    structure       JSONB NOT NULL,
    -- {
    --   "sections": [
    --     {
    --       "id": "s1",
    --       "title": "Personal Protective Equipment",
    --       "questions": [
    --         {
    --           "id": "q1",
    --           "text": "Are all workers wearing required PPE?",
    --           "type": "yes_no",
    --           "required": true,
    --           "fail_triggers_capa": true
    --         },
    --         {
    --           "id": "q2",
    --           "text": "Describe any PPE deficiencies observed",
    --           "type": "text",
    --           "required": false,
    --           "conditional_on": {"q1": "no"}
    --         }
    --       ]
    --     }
    --   ]
    -- }

    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_template_org ON inspection_template(organisation_id);

-- Completed inspection
CREATE TABLE inspection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    template_id     UUID NOT NULL REFERENCES inspection_template(id),
    inspection_number VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'in_progress',
    scheduled_date  DATE,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    inspector_id    UUID NOT NULL REFERENCES app_user(id),

    -- Scoring summary (relational for querying)
    score_percent   DECIMAL(5, 2),
    total_items     INTEGER,
    passed_items    INTEGER,
    failed_items    INTEGER,

    -- All responses in JSONB (mirrors template structure with answers)
    responses       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "q1": {"value": "no", "score": 0, "notes": "Two workers missing hard hats", "photos": ["url1"]},
    --   "q2": {"value": "Workers in Zone C not wearing hard hats despite posted signage"}
    -- }

    findings        JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {
    --     "question_id": "q1",
    --     "finding_type": "non_conformance",
    --     "description": "PPE non-compliance in Zone C",
    --     "capa_id": "uuid-of-linked-capa"
    --   }
    -- ]

    location_data   JSONB,                         -- GPS, floor plan pin
    custom_fields   JSONB NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, inspection_number)
);

CREATE INDEX idx_inspection_org_date ON inspection(organisation_id, scheduled_date DESC);
CREATE INDEX idx_inspection_site ON inspection(site_id);
CREATE INDEX idx_inspection_status ON inspection(status);
```

### Risk Assessment

```sql
CREATE TABLE risk_assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    site_id         UUID REFERENCES site(id),
    assessment_number VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    assessment_type VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    assessed_by     UUID NOT NULL REFERENCES app_user(id),
    reviewed_by     UUID REFERENCES app_user(id),
    next_review_date DATE,

    -- Hazards as JSONB array (avoids separate hazard table)
    hazards         JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {
    --     "id": "h1",
    --     "description": "Exposure to welding fumes",
    --     "consequence": "Respiratory illness",
    --     "existing_controls": "Local exhaust ventilation",
    --     "initial": {"likelihood": 4, "severity": 3, "score": 12, "level": "high"},
    --     "residual": {"likelihood": 2, "severity": 3, "score": 6, "level": "medium"},
    --     "additional_controls": "Enforce respiratory protection programme"
    --   }
    -- ]

    risk_matrix_config JSONB,                      -- Override org default if needed
    custom_fields   JSONB NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, assessment_number)
);

CREATE INDEX idx_risk_org ON risk_assessment(organisation_id);
CREATE INDEX idx_risk_site ON risk_assessment(site_id);
```

### Chemical Management

```sql
CREATE TABLE chemical (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    cas_number      VARCHAR(20),
    chemical_name   VARCHAR(500) NOT NULL,
    physical_state  VARCHAR(20),
    is_hazardous    BOOLEAN NOT NULL DEFAULT false,
    signal_word     VARCHAR(20),

    -- GHS classification data
    ghs_data        JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "hazard_statements": ["H225", "H319", "H336"],
    --   "precautionary_statements": ["P210", "P233", "P240"],
    --   "hazard_classes": ["flammable_liquid_2", "eye_irritation_2A"],
    --   "pictograms": ["GHS02", "GHS07"]
    -- }

    -- Current SDS data (structured from the 16 GHS sections)
    sds_data        JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "version": 3,
    --   "revision_date": "2025-11-15",
    --   "supplier": {"name": "...", "address": "...", "phone": "...", "emergency_phone": "..."},
    --   "product_identifier": "...",
    --   "recommended_use": "Industrial solvent",
    --   "first_aid": {"inhalation": "...", "skin": "...", "eyes": "...", "ingestion": "..."},
    --   "firefighting": {"suitable_media": "...", "unsuitable_media": "..."},
    --   "exposure_limits": {"oel_ppm": 500, "oel_mg_m3": 1188, "source": "OSHA PEL"},
    --   "physical_properties": {"boiling_point": "56.2°C", "flash_point": "-20°C", "vapour_pressure": "24 kPa"},
    --   "stability": {"conditions_to_avoid": "...", "incompatible_materials": "..."},
    --   "document_url": "https://..."
    -- }

    -- Regulatory compliance
    regulatory_status JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "reach": {"registered": true, "registration_number": "01-1234567890-12"},
    --   "rohs": {"restricted": false},
    --   "epcra": {"tpq_kg": 4536, "section_302": true},
    --   "tsca": {"listed": true}
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_chemical_cas ON chemical(cas_number);
CREATE INDEX idx_chemical_org ON chemical(organisation_id);
CREATE INDEX idx_chemical_ghs ON chemical USING GIN (ghs_data jsonb_path_ops);

-- Chemical inventory at sites
CREATE TABLE chemical_inventory (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    chemical_id     UUID NOT NULL REFERENCES chemical(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    storage_location VARCHAR(255) NOT NULL,
    quantity        DECIMAL(12, 4) NOT NULL,
    unit            VARCHAR(20) NOT NULL,
    max_quantity    DECIMAL(12, 4),
    epcra_threshold_exceeded BOOLEAN DEFAULT false,
    last_audit_date DATE,
    inventory_data  JSONB NOT NULL DEFAULT '{}',   -- Container count, lot numbers, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inventory_site ON chemical_inventory(site_id);
CREATE INDEX idx_inventory_chemical ON chemical_inventory(chemical_id);
```

### Training Management

```sql
CREATE TABLE training_course (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    code            VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    course_type     VARCHAR(30),
    duration_hours  DECIMAL(5, 2),
    is_mandatory    BOOLEAN NOT NULL DEFAULT false,
    recertification_months INTEGER,
    regulatory_reference VARCHAR(255),
    requirements    JSONB NOT NULL DEFAULT '{}',    -- Who needs this course
    -- {
    --   "applies_to": ["all"],
    --   "departments": ["warehouse", "production"],
    --   "roles": ["forklift_operator"],
    --   "sites": ["site-uuid-1"]
    -- }
    course_content  JSONB NOT NULL DEFAULT '{}',    -- Modules, materials
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE training_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    course_id       UUID NOT NULL REFERENCES training_course(id),
    completion_date DATE NOT NULL,
    expiry_date     DATE,
    score           DECIMAL(5, 2),
    status          VARCHAR(20) NOT NULL DEFAULT 'completed',
    record_data     JSONB NOT NULL DEFAULT '{}',   -- Instructor, location, certificate URL
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_training_user ON training_record(user_id);
CREATE INDEX idx_training_expiry ON training_record(expiry_date) WHERE status = 'completed';
```

### Environmental Compliance & ESG

```sql
CREATE TABLE environmental_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    record_type     VARCHAR(30) NOT NULL,          -- 'emission','waste','water','energy','permit'
    record_date     DATE NOT NULL,
    period_start    DATE,
    period_end      DATE,

    -- Common relational fields
    ghg_scope       INTEGER,                       -- 1, 2, 3 (for emissions)
    quantity        DECIMAL(14, 4),
    unit            VARCHAR(30),
    co2e_kg         DECIMAL(14, 4),

    -- Type-specific data in JSONB
    record_data     JSONB NOT NULL DEFAULT '{}',
    -- Emission example:
    -- {
    --   "source_name": "Natural Gas Boiler #3",
    --   "source_type": "stationary_combustion",
    --   "fuel_type": "natural_gas",
    --   "activity_data": 50000,
    --   "activity_unit": "m3",
    --   "emission_factor": 2.02,
    --   "emission_factor_source": "EPA",
    --   "scope_3_category": null,
    --   "data_quality": "measured"
    -- }
    --
    -- Waste example:
    -- {
    --   "waste_type": "hazardous",
    --   "waste_stream": "Spent solvents",
    --   "disposal_method": "incineration",
    --   "manifest_number": "012345678JJK",
    --   "transporter": "Clean Harbors",
    --   "disposal_facility": "Clean Harbors Deer Park"
    -- }
    --
    -- Permit example:
    -- {
    --   "permit_number": "TX-AIR-2024-0042",
    --   "permit_type": "air",
    --   "issuing_authority": "TCEQ",
    --   "issue_date": "2024-01-15",
    --   "expiry_date": "2029-01-15",
    --   "conditions": [
    --     {"number": "1", "description": "NOx emissions < 50 tpy", "limit": 50, "unit": "tpy"}
    --   ]
    -- }

    -- ESG/ESRS data point mapping
    esrs_data       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "standard": "E1",
    --   "disclosure_requirement": "E1-6",
    --   "data_point": "Gross Scope 1 GHG emissions",
    --   "metric_tonnes_co2e": 125.5
    -- }

    custom_fields   JSONB NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_envrecord_org_type ON environmental_record(organisation_id, record_type);
CREATE INDEX idx_envrecord_site_date ON environmental_record(site_id, record_date DESC);
CREATE INDEX idx_envrecord_scope ON environmental_record(ghg_scope) WHERE record_type = 'emission';
CREATE INDEX idx_envrecord_data ON environmental_record USING GIN (record_data jsonb_path_ops);
```

### Documents & Audit Log

```sql
CREATE TABLE document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    entity_type     VARCHAR(50) NOT NULL,          -- 'incident','inspection','capa','chemical','training'
    entity_id       UUID NOT NULL,
    document_type   VARCHAR(50) NOT NULL,          -- 'photo','pdf','video','certificate','sds'
    title           VARCHAR(500),
    file_url        VARCHAR(1000) NOT NULL,
    file_size_bytes BIGINT,
    mime_type       VARCHAR(100),
    metadata        JSONB NOT NULL DEFAULT '{}',   -- Width, height, GPS, etc.
    uploaded_by     UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_document_entity ON document(entity_type, entity_id);
CREATE INDEX idx_document_org ON document(organisation_id);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID REFERENCES app_user(id),
    action          VARCHAR(30) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID,
    changes         JSONB,                         -- {"field": {"old": "x", "new": "y"}}
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org_date ON audit_log(organisation_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

### Row-Level Security

```sql
ALTER TABLE incident ENABLE ROW LEVEL SECURITY;
ALTER TABLE corrective_action ENABLE ROW LEVEL SECURITY;
ALTER TABLE inspection ENABLE ROW LEVEL SECURITY;
ALTER TABLE chemical ENABLE ROW LEVEL SECURITY;
ALTER TABLE risk_assessment ENABLE ROW LEVEL SECURITY;
ALTER TABLE environmental_record ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_log ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON incident
    USING (organisation_id = current_setting('app.current_org_id')::UUID);

CREATE POLICY tenant_isolation ON corrective_action
    USING (organisation_id = current_setting('app.current_org_id')::UUID);

CREATE POLICY tenant_isolation ON inspection
    USING (organisation_id = current_setting('app.current_org_id')::UUID);
```

---

## Example Queries

### OSHA 300 log generation from hybrid fields

```sql
-- Generate OSHA 300 log data for a site and year
SELECT
    i.osha_case_number AS case_no,
    CASE WHEN i.regulatory_data->'osha_301'->>'is_privacy_case' = 'true'
         THEN 'Privacy Case'
         ELSE (i.persons_involved->0->>'name')
    END AS employee_name,
    (i.persons_involved->0->>'job_title') AS job_title,
    i.incident_date,
    i.location_detail,
    LEFT(i.description, 200) AS description,
    i.osha_classification,
    i.days_away,
    i.days_restricted,
    (i.persons_involved->0->>'injury_type') AS injury_type
FROM incident i
WHERE i.site_id = 'site-uuid'
  AND i.is_osha_recordable = true
  AND EXTRACT(YEAR FROM i.incident_date) = 2026
ORDER BY i.incident_date;
```

### Chemical safety query using JSONB

```sql
-- Find all flammable chemicals at a site
SELECT
    c.chemical_name,
    c.cas_number,
    ci.quantity,
    ci.unit,
    ci.storage_location,
    c.ghs_data->'hazard_statements' AS hazard_codes,
    c.sds_data->'exposure_limits'->>'oel_ppm' AS oel_ppm
FROM chemical c
JOIN chemical_inventory ci ON ci.chemical_id = c.id
WHERE ci.site_id = 'site-uuid'
  AND c.ghs_data @> '{"pictograms": ["GHS02"]}'  -- GHS02 = flame pictogram
ORDER BY ci.quantity DESC;
```

### GHG Protocol emissions summary

```sql
-- Annual emissions by scope using hybrid fields
SELECT
    ghg_scope,
    SUM(co2e_kg) / 1000.0 AS co2e_tonnes,
    COUNT(*) AS record_count,
    record_data->>'data_quality' AS quality
FROM environmental_record
WHERE organisation_id = 'org-uuid'
  AND record_type = 'emission'
  AND period_start >= '2026-01-01'
  AND period_end <= '2026-12-31'
GROUP BY ghg_scope, record_data->>'data_quality'
ORDER BY ghg_scope;
```

### Custom field filtering

```sql
-- Find incidents during night shift with contractor involvement
SELECT incident_number, title, incident_date, severity
FROM incident
WHERE organisation_id = 'org-uuid'
  AND custom_fields @> '{"shift_type": "night", "contractor_involved": true}'
ORDER BY incident_date DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-tenancy & Identity | 3 | organisation, site, app_user (roles as array, no junction tables) |
| Incident Management | 1 | Single table with JSONB for persons, root causes, regulatory data |
| Corrective Actions | 1 | CAPA with JSONB workflow history and evidence |
| Inspections | 2 | inspection_template (JSONB structure), inspection (JSONB responses) |
| Risk Assessment | 1 | Hazards stored as JSONB array within assessment |
| Chemical Management | 2 | chemical (JSONB for GHS, SDS, regulatory), chemical_inventory |
| Training | 2 | training_course, training_record |
| Environmental & ESG | 1 | Single table for emissions, waste, water, permits (type-discriminated with JSONB) |
| Documents & Audit | 2 | document, audit_log |
| **Total** | **~15** | Significantly fewer tables than normalised model |

---

## Key Design Decisions

1. **Relational spine for regulatory fields, JSONB for everything else** — fields that appear on OSHA forms, GHS labels, or GHG Protocol reports are relational columns with proper types and indexes. Fields that vary by jurisdiction, organisation, or evolve rapidly are JSONB. This maximises query performance for regulatory reporting while maintaining flexibility.

2. **Inspection templates as JSONB documents** — an inspection template is inherently a document: a hierarchy of sections containing questions with options and conditional logic. Storing this as JSONB avoids the three-table normalised pattern (template -> section -> question) and makes template versioning trivial (store the whole structure, compare versions as JSON diff).

3. **Persons involved as JSONB array on incident** — rather than a separate `incident_person` table, involved persons are stored as a JSONB array on the incident. This eliminates a JOIN for the most common query pattern (viewing an incident with its people) and simplifies the mobile offline sync model.

4. **Single environmental_record table with type discriminator** — emissions, waste, water, and permit records share a single table with a `record_type` column and type-specific data in JSONB. This reduces table count and simplifies ESG dashboard queries that aggregate across environmental categories.

5. **Roles as PostgreSQL text array** — instead of role/permission/user_role junction tables, roles are stored as a `TEXT[]` column on `app_user` with GIN indexing. This simplifies role checks (`'admin' = ANY(roles)`) and reduces JOIN complexity for the most common authorisation query pattern.

6. **Workflow history as JSONB on CAPA** — the corrective action `workflow_history` JSONB array captures every status transition, assignment change, and escalation as a timestamped entry. This provides an inline audit trail without requiring a separate audit table or event-sourcing infrastructure.

7. **Organisation settings define custom fields** — the `organisation.settings` JSONB includes a `custom_incident_fields` array that defines additional fields for incident capture. The `custom_fields` JSONB on each incident stores the values. This enables per-organisation field configuration without schema changes.

8. **ESRS data point mapping as JSONB** — because the ESRS standard is still evolving (amended in 2025, data point count reduced from 1,100+ to ~330), mapping EHS records to ESRS data points via JSONB provides flexibility to adapt as the standard stabilises without schema migrations.

9. **Multi-jurisdiction support via regulatory_data JSONB** — OSHA 301 fields, RIDDOR fields, and DGUV fields all coexist in the `regulatory_data` JSONB column, keyed by regulation name. An organisation operating in both the US and UK simply populates both `osha_301` and `riddor` keys. No schema change needed.

10. **GIN indexes on all JSONB columns** — every JSONB column used for filtering has a `jsonb_path_ops` GIN index, enabling efficient containment queries (`@>` operator) for custom fields, GHS hazard data, and regulatory data. This keeps JSONB query performance competitive with relational lookups for common access patterns.
