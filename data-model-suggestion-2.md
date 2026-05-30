# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Environmental Health & Safety (EHS) · Created: 2026-05-22

## Philosophy

The Event-Sourced / Audit-First model treats every change to EHS data as an immutable domain event stored in an append-only event store. The current state of any entity (an incident, a corrective action, a chemical inventory level) is derived by replaying its event history. Read-optimised materialised views (projections) serve the application's query needs, while the event store remains the single source of truth. This is the CQRS (Command Query Responsibility Segregation) pattern: writes go to the event store, reads come from projections.

This architecture is a natural fit for EHS because the domain has an inherent requirement for complete audit trails. ISO 45001 requires traceable records of hazard identification, risk assessment changes, incident investigation progression, and corrective action lifecycle. OSHA mandates that injury records be maintained and accessible for five years. GDPR requires demonstrable data processing history for personal health data. An event-sourced system satisfies all of these requirements by design — the audit trail is not a secondary feature bolted on after the fact; it is the primary data structure.

The key differentiator from the normalised relational model is temporal querying. An event-sourced EHS platform can answer questions like "What was the risk assessment for Site A on 1 March 2025?" or "Show me the full investigation timeline for Incident #3847" by replaying events up to a specific point in time. This capability is extremely valuable for regulatory investigations, insurance claims, and AI-powered trend analysis. The trade-off is increased implementation complexity: developers must think in terms of events and projections rather than simple CRUD operations.

**Best for:** Organisations requiring bulletproof audit trails, temporal queries ("what was true at time X?"), and AI/ML-powered predictive analytics that benefit from rich event history data.

**Trade-offs:**
- (+) Complete, immutable audit trail by design — satisfies ISO 45001, OSHA, and GDPR requirements without additional audit log tables
- (+) Temporal queries are trivial — replay events to any point in time
- (+) Rich event data feeds predictive SIF modelling and AI analytics
- (+) Schema evolution is easier — add new event types without migrating existing data
- (+) Event replay enables debugging and root cause analysis of data issues
- (-) Higher implementation complexity — developers must learn event sourcing patterns
- (-) Eventual consistency between event store and projections requires careful handling
- (-) Projection rebuilds can be slow for large event histories
- (-) Querying across entities requires well-designed projections or CQRS read models
- (-) More storage consumed than a mutable-state model (events are never deleted)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 45001:2018 | Every clause 10 corrective action, clause 6 risk assessment change, and clause 10 incident investigation step is captured as a discrete event, providing the traceable records ISO 45001 requires |
| ISO 14001:2015 | Environmental aspect changes, legal compliance updates, and operational control modifications each produce events that form the environmental management audit trail |
| OSHA 29 CFR 1904 | OSHA recordkeeping events (case opened, classification changed, days counted, 300A submitted) are first-class domain events; the OSHA 300/301/300A projections materialise from these events |
| GHS Rev.10 | Chemical registration, SDS version updates, hazard reclassifications, and inventory changes are captured as chemical management events |
| GHG Protocol | Emission source configuration changes and periodic activity data entries are events; the emissions projection calculates cumulative CO2e from event history |
| GDPR | Data subject events (consent given, data accessed, data exported, data deleted) are explicit event types, providing the processing history GDPR Article 30 requires |
| OCSF | Event schema draws inspiration from the Open Cybersecurity Schema Framework's event categorisation for structured security and compliance event logging |

---

## Event Store Schema

```sql
-- =============================================================
-- EVENT STORE — the immutable source of truth
-- =============================================================

-- Stream: a logical sequence of events for one entity
CREATE TABLE event_stream (
    stream_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     VARCHAR(50) NOT NULL,          -- 'incident','inspection','corrective_action','chemical','risk_assessment','training','permit','emission'
    aggregate_id    UUID NOT NULL,                 -- The ID of the domain entity this stream represents
    organisation_id UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_type, aggregate_id)
);

CREATE INDEX idx_stream_org ON event_stream(organisation_id);
CREATE INDEX idx_stream_type ON event_stream(stream_type);

-- Domain event: the atomic unit of change
CREATE TABLE domain_event (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL REFERENCES event_stream(stream_id),
    event_type      VARCHAR(100) NOT NULL,         -- See event type catalogue below
    event_version   INTEGER NOT NULL,              -- Monotonically increasing per stream
    payload         JSONB NOT NULL,                -- Event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',   -- Correlation IDs, causation IDs, user agent
    actor_id        UUID,                          -- User who triggered the event
    actor_ip        INET,
    organisation_id UUID NOT NULL,                 -- Denormalised for RLS
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, event_version)
);

-- Append-only: no UPDATE or DELETE policies
CREATE INDEX idx_event_stream ON domain_event(stream_id, event_version);
CREATE INDEX idx_event_type ON domain_event(event_type);
CREATE INDEX idx_event_org_time ON domain_event(organisation_id, occurred_at DESC);
CREATE INDEX idx_event_actor ON domain_event(actor_id);
CREATE INDEX idx_event_payload ON domain_event USING GIN (payload jsonb_path_ops);

-- Optimistic concurrency: ensure no two writers append to the same stream position
-- The UNIQUE(stream_id, event_version) constraint handles this.

-- Snapshot: periodic state snapshots to avoid replaying full history
CREATE TABLE event_snapshot (
    snapshot_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL REFERENCES event_stream(stream_id),
    stream_type     VARCHAR(50) NOT NULL,
    last_event_version INTEGER NOT NULL,
    state           JSONB NOT NULL,                -- Serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_snapshot_stream ON event_snapshot(stream_id, last_event_version DESC);
```

### Event Type Catalogue

```sql
-- Reference table documenting all known event types
CREATE TABLE event_type_registry (
    event_type      VARCHAR(100) PRIMARY KEY,
    category        VARCHAR(50) NOT NULL,          -- 'incident','inspection','capa','chemical','training','environmental','system'
    description     TEXT NOT NULL,
    payload_schema  JSONB,                         -- JSON Schema for the payload
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Seed data: Incident lifecycle events
INSERT INTO event_type_registry (event_type, category, description) VALUES
-- Incident events
('incident.reported',           'incident', 'A new incident or near-miss was reported'),
('incident.classified',         'incident', 'Incident type and severity were classified'),
('incident.osha_recordable_set','incident', 'OSHA recordability determination was made'),
('incident.sif_potential_flagged','incident','Incident was flagged as SIF-potential'),
('incident.investigation_started','incident','Investigation was initiated'),
('incident.witness_added',      'incident', 'A witness statement was added'),
('incident.person_involved_added','incident','An involved person was added with injury details'),
('incident.root_cause_identified','incident','A root cause finding was recorded'),
('incident.corrective_action_linked','incident','A corrective action was linked to the incident'),
('incident.investigation_completed','incident','Investigation was marked complete'),
('incident.closed',             'incident', 'Incident was closed'),
('incident.reopened',           'incident', 'A closed incident was reopened'),

-- Inspection events
('inspection.scheduled',        'inspection', 'An inspection was scheduled'),
('inspection.started',          'inspection', 'An inspection was started in the field'),
('inspection.response_recorded','inspection', 'A question response was recorded'),
('inspection.photo_attached',   'inspection', 'A photo was attached to a response'),
('inspection.completed',        'inspection', 'An inspection was completed and scored'),
('inspection.finding_raised',   'inspection', 'A non-conformance finding was raised'),

-- CAPA events
('capa.created',                'capa', 'A corrective/preventive action was created'),
('capa.assigned',               'capa', 'A CAPA was assigned to a responsible person'),
('capa.reassigned',             'capa', 'A CAPA was reassigned to a different person'),
('capa.status_changed',         'capa', 'CAPA status was updated'),
('capa.evidence_attached',      'capa', 'Evidence of completion was attached'),
('capa.verification_requested', 'capa', 'Verification of CAPA effectiveness was requested'),
('capa.verified',               'capa', 'CAPA was verified as effective'),
('capa.closed',                 'capa', 'CAPA was closed'),
('capa.overdue_escalated',      'capa', 'An overdue CAPA was escalated'),

-- Chemical management events
('chemical.registered',         'chemical', 'A new chemical was added to the inventory'),
('chemical.sds_uploaded',       'chemical', 'A new SDS version was uploaded'),
('chemical.hazard_reclassified','chemical', 'Chemical hazard classification was updated'),
('chemical.inventory_adjusted', 'chemical', 'Chemical inventory quantity was adjusted'),
('chemical.epcra_threshold_exceeded','chemical','Chemical quantity exceeded EPCRA Tier II threshold'),

-- Training events
('training.course_created',     'training', 'A training course was created'),
('training.enrolled',           'training', 'A user was enrolled in training'),
('training.completed',          'training', 'A user completed training'),
('training.expired',            'training', 'A training certification expired'),
('training.recertification_due','training', 'A recertification reminder was generated'),

-- Environmental events
('emission.source_configured',  'environmental', 'An emission source was configured'),
('emission.activity_recorded',  'environmental', 'Activity data for an emission source was recorded'),
('emission.factor_updated',     'environmental', 'An emission factor was updated'),
('permit.issued',               'environmental', 'An environmental permit was issued'),
('permit.condition_added',      'environmental', 'A permit condition was added'),
('permit.compliance_recorded',  'environmental', 'A permit compliance measurement was recorded'),
('waste.disposed',              'environmental', 'A waste disposal was recorded');
```

## Materialised Projections (Read Models)

```sql
-- =============================================================
-- PROJECTIONS — mutable read models rebuilt from events
-- =============================================================

-- Current incident state (materialised from incident.* events)
CREATE TABLE proj_incident (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    site_id         UUID NOT NULL,
    incident_number VARCHAR(50) NOT NULL,
    incident_type   VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    incident_date   DATE NOT NULL,
    incident_time   TIME,
    location_detail VARCHAR(500),
    status          VARCHAR(30) NOT NULL,
    severity        VARCHAR(20),
    probability     VARCHAR(20),
    risk_score      INTEGER,
    is_osha_recordable BOOLEAN NOT NULL DEFAULT false,
    is_sif_potential BOOLEAN NOT NULL DEFAULT false,
    reported_by     UUID NOT NULL,
    investigated_by UUID,
    root_cause_count INTEGER DEFAULT 0,
    capa_count      INTEGER DEFAULT 0,
    event_count     INTEGER DEFAULT 0,             -- Total events in this stream
    last_event_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_incident_org ON proj_incident(organisation_id, incident_date DESC);
CREATE INDEX idx_proj_incident_status ON proj_incident(status);
CREATE INDEX idx_proj_incident_sif ON proj_incident(is_sif_potential) WHERE is_sif_potential = true;

-- Persons involved in incidents
CREATE TABLE proj_incident_person (
    id              UUID PRIMARY KEY,
    incident_id     UUID NOT NULL REFERENCES proj_incident(id),
    person_name     VARCHAR(255) NOT NULL,
    role_in_incident VARCHAR(50) NOT NULL,
    injury_type     VARCHAR(100),
    body_part       VARCHAR(100),
    treatment_type  VARCHAR(50),
    days_away       INTEGER DEFAULT 0,
    days_restricted INTEGER DEFAULT 0
);

-- OSHA 300 Log projection (materialised from incident + osha events)
CREATE TABLE proj_osha_300 (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    site_id         UUID NOT NULL,
    incident_id     UUID NOT NULL,
    calendar_year   INTEGER NOT NULL,
    case_number     VARCHAR(20) NOT NULL,
    employee_name   VARCHAR(255),
    job_title       VARCHAR(255),
    injury_date     DATE NOT NULL,
    description     TEXT NOT NULL,
    is_death        BOOLEAN DEFAULT false,
    is_days_away    BOOLEAN DEFAULT false,
    is_restricted   BOOLEAN DEFAULT false,
    is_other_recordable BOOLEAN DEFAULT false,
    days_away_count INTEGER DEFAULT 0,
    days_restricted_count INTEGER DEFAULT 0,
    injury_type     VARCHAR(50),
    is_privacy_case BOOLEAN DEFAULT false,
    submitted_to_ita BOOLEAN DEFAULT false
);

CREATE INDEX idx_proj_osha300_site_year ON proj_osha_300(site_id, calendar_year);

-- Current CAPA state
CREATE TABLE proj_corrective_action (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    action_number   VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    action_type     VARCHAR(20) NOT NULL,
    priority        VARCHAR(20) NOT NULL,
    status          VARCHAR(30) NOT NULL,
    source_type     VARCHAR(30),
    source_id       UUID,
    assigned_to     UUID NOT NULL,
    due_date        DATE NOT NULL,
    completed_date  DATE,
    verified_by     UUID,
    is_overdue      BOOLEAN DEFAULT false,
    escalation_count INTEGER DEFAULT 0,
    event_count     INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_capa_org ON proj_corrective_action(organisation_id, status);
CREATE INDEX idx_proj_capa_assigned ON proj_corrective_action(assigned_to);
CREATE INDEX idx_proj_capa_overdue ON proj_corrective_action(due_date) WHERE is_overdue = true;

-- Current inspection state
CREATE TABLE proj_inspection (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    site_id         UUID NOT NULL,
    template_name   VARCHAR(255),
    inspection_number VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    scheduled_date  DATE,
    completed_at    TIMESTAMPTZ,
    inspector_id    UUID NOT NULL,
    score_percent   DECIMAL(5, 2),
    total_items     INTEGER,
    passed_items    INTEGER,
    failed_items    INTEGER,
    finding_count   INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

-- Chemical inventory projection
CREATE TABLE proj_chemical_inventory (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    site_id         UUID NOT NULL,
    chemical_name   VARCHAR(500) NOT NULL,
    cas_number      VARCHAR(20),
    current_quantity DECIMAL(12, 4),
    unit            VARCHAR(20),
    is_hazardous    BOOLEAN DEFAULT false,
    signal_word     VARCHAR(20),
    current_sds_version INTEGER,
    sds_revision_date DATE,
    epcra_threshold_exceeded BOOLEAN DEFAULT false,
    last_event_at   TIMESTAMPTZ
);

CREATE INDEX idx_proj_chem_site ON proj_chemical_inventory(site_id);

-- Emissions summary projection (GHG Protocol aligned)
CREATE TABLE proj_emissions_summary (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    site_id         UUID NOT NULL,
    ghg_scope       INTEGER NOT NULL,
    scope_3_category INTEGER,
    source_name     VARCHAR(255),
    reporting_year  INTEGER NOT NULL,
    reporting_month INTEGER,                       -- NULL for annual totals
    total_co2e_kg   DECIMAL(14, 4) NOT NULL,
    activity_total  DECIMAL(14, 4),
    activity_unit   VARCHAR(30),
    data_quality    VARCHAR(20),
    last_updated    TIMESTAMPTZ
);

CREATE INDEX idx_proj_emissions ON proj_emissions_summary(organisation_id, reporting_year, ghg_scope);

-- Training compliance projection
CREATE TABLE proj_training_compliance (
    user_id         UUID NOT NULL,
    organisation_id UUID NOT NULL,
    course_id       UUID NOT NULL,
    course_title    VARCHAR(255) NOT NULL,
    is_mandatory    BOOLEAN NOT NULL,
    status          VARCHAR(20) NOT NULL,          -- 'completed','expired','overdue','not_started'
    completion_date DATE,
    expiry_date     DATE,
    days_until_expiry INTEGER,
    PRIMARY KEY (user_id, course_id)
);

CREATE INDEX idx_proj_training_expiry ON proj_training_compliance(expiry_date)
    WHERE status IN ('completed') AND expiry_date IS NOT NULL;
```

## Projection Rebuild Infrastructure

```sql
-- Tracks the last processed event for each projection
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID REFERENCES domain_event(event_id),
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    event_count     BIGINT NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'active', -- 'active','rebuilding','paused'
    error_message   TEXT
);

-- Subscription: which projections process which event types
CREATE TABLE projection_subscription (
    projection_name VARCHAR(100) NOT NULL REFERENCES projection_checkpoint(projection_name),
    event_type      VARCHAR(100) NOT NULL,
    PRIMARY KEY (projection_name, event_type)
);

-- Example subscriptions
INSERT INTO projection_subscription (projection_name, event_type) VALUES
('proj_incident', 'incident.reported'),
('proj_incident', 'incident.classified'),
('proj_incident', 'incident.investigation_started'),
('proj_incident', 'incident.root_cause_identified'),
('proj_incident', 'incident.corrective_action_linked'),
('proj_incident', 'incident.investigation_completed'),
('proj_incident', 'incident.closed'),
('proj_incident', 'incident.reopened'),
('proj_osha_300', 'incident.reported'),
('proj_osha_300', 'incident.osha_recordable_set'),
('proj_osha_300', 'incident.person_involved_added'),
('proj_corrective_action', 'capa.created'),
('proj_corrective_action', 'capa.assigned'),
('proj_corrective_action', 'capa.status_changed'),
('proj_corrective_action', 'capa.verified'),
('proj_corrective_action', 'capa.closed'),
('proj_corrective_action', 'capa.overdue_escalated');
```

## Example: Event Payload Structures

```sql
-- Example: incident.reported event payload
-- {
--   "incident_number": "INC-2026-0042",
--   "site_id": "a1b2c3d4-...",
--   "incident_type": "near_miss",
--   "title": "Forklift near-miss in Warehouse B",
--   "description": "Forklift operator reversed without checking mirrors...",
--   "incident_date": "2026-05-20",
--   "incident_time": "14:30:00",
--   "location_detail": "Warehouse B, Aisle 7",
--   "reported_by": "e5f6g7h8-..."
-- }

-- Example: incident.root_cause_identified event payload
-- {
--   "method": "5_why",
--   "finding_text": "Operator had not completed forklift refresher training",
--   "category": "human_factor",
--   "is_primary": true,
--   "identified_by": "i9j0k1l2-..."
-- }

-- Example: capa.created event payload
-- {
--   "action_number": "CA-2026-0108",
--   "title": "Implement mandatory reverse camera check procedure",
--   "description": "All forklift operators must...",
--   "action_type": "corrective",
--   "priority": "high",
--   "assigned_to": "m3n4o5p6-...",
--   "due_date": "2026-06-20",
--   "source_type": "incident",
--   "source_id": "q7r8s9t0-..."
-- }

-- Example: chemical.inventory_adjusted event payload
-- {
--   "site_id": "a1b2c3d4-...",
--   "chemical_name": "Acetone",
--   "cas_number": "67-64-1",
--   "previous_quantity": 45.0,
--   "new_quantity": 120.0,
--   "unit": "L",
--   "adjustment_reason": "delivery",
--   "epcra_threshold_exceeded": false
-- }

-- Example: emission.activity_recorded event payload
-- {
--   "source_id": "u1v2w3x4-...",
--   "reporting_period_start": "2026-04-01",
--   "reporting_period_end": "2026-04-30",
--   "activity_data": 12500.0,
--   "activity_unit": "kWh",
--   "co2e_kg": 5125.0,
--   "data_quality": "measured"
-- }
```

## Example Queries

### Temporal query: incident state at a specific date

```sql
-- Replay events to reconstruct incident state as of 2026-03-01
SELECT e.event_type, e.payload, e.occurred_at
FROM domain_event e
JOIN event_stream s ON e.stream_id = s.stream_id
WHERE s.stream_type = 'incident'
  AND s.aggregate_id = 'incident-uuid-here'
  AND e.occurred_at <= '2026-03-01T23:59:59Z'
ORDER BY e.event_version ASC;
```

### Investigation timeline

```sql
-- Full timeline of all events for an incident
SELECT
    e.event_type,
    e.occurred_at,
    e.payload->>'title' AS detail,
    u.display_name AS actor
FROM domain_event e
JOIN event_stream s ON e.stream_id = s.stream_id
LEFT JOIN proj_incident pi ON s.aggregate_id = pi.id
LEFT JOIN app_user u ON e.actor_id = u.id
WHERE s.aggregate_id = 'incident-uuid-here'
ORDER BY e.event_version ASC;
```

### SIF precursor pattern analysis (AI feed)

```sql
-- Find patterns in events preceding SIF-potential incidents
SELECT
    e_prior.event_type,
    e_prior.payload,
    e_prior.occurred_at,
    pi.incident_number,
    pi.title
FROM proj_incident pi
JOIN event_stream s ON s.aggregate_id = pi.id AND s.stream_type = 'incident'
JOIN domain_event e_sif ON e_sif.stream_id = s.stream_id
    AND e_sif.event_type = 'incident.sif_potential_flagged'
JOIN domain_event e_prior ON e_prior.organisation_id = pi.organisation_id
    AND e_prior.occurred_at BETWEEN (e_sif.occurred_at - INTERVAL '90 days') AND e_sif.occurred_at
    AND e_prior.event_type IN ('inspection.finding_raised', 'capa.overdue_escalated', 'training.expired')
WHERE pi.is_sif_potential = true
ORDER BY pi.incident_date DESC, e_prior.occurred_at ASC;
```

---

## Multi-Tenancy & Identity (Shared Tables)

```sql
-- These tables are mutable (not event-sourced) — they are platform infrastructure
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    country_code    CHAR(2) NOT NULL,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    naics_code      VARCHAR(10),
    employee_count  INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    job_title       VARCHAR(255),
    site_id         UUID REFERENCES site(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, email)
);

-- Row-Level Security on the event store
ALTER TABLE domain_event ENABLE ROW LEVEL SECURITY;
ALTER TABLE event_stream ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON domain_event
    USING (organisation_id = current_setting('app.current_org_id')::UUID);

CREATE POLICY tenant_isolation ON event_stream
    USING (organisation_id = current_setting('app.current_org_id')::UUID);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 3 | event_stream, domain_event, event_snapshot |
| Event Infrastructure | 3 | event_type_registry, projection_checkpoint, projection_subscription |
| Projections: Incidents | 2 | proj_incident, proj_incident_person |
| Projections: OSHA | 1 | proj_osha_300 |
| Projections: CAPA | 1 | proj_corrective_action |
| Projections: Inspections | 1 | proj_inspection |
| Projections: Chemical | 1 | proj_chemical_inventory |
| Projections: Emissions | 1 | proj_emissions_summary |
| Projections: Training | 1 | proj_training_compliance |
| Platform (mutable) | 3 | organisation, site, app_user |
| **Total** | **~17** | Event store tables + projection tables; projections are rebuildable |

---

## Key Design Decisions

1. **Append-only event store as single source of truth** — the `domain_event` table is append-only with no UPDATE or DELETE operations permitted. This provides an immutable, tamper-evident audit trail that satisfies ISO 45001 record-keeping requirements and OSHA five-year retention mandates without any additional audit logging mechanism.

2. **Event versioning for optimistic concurrency** — `domain_event.event_version` is monotonically increasing per stream, with a UNIQUE constraint on `(stream_id, event_version)`. This prevents concurrent writes from corrupting aggregate state — if two users try to update the same incident simultaneously, only one succeeds and the other must retry with the current version.

3. **Projections as rebuildable read models** — all `proj_*` tables can be dropped and rebuilt from the event store at any time. This means schema changes to read models are non-destructive: add new columns to a projection, replay events, and the new columns are populated from historical data.

4. **Snapshots for performance** — the `event_snapshot` table periodically captures aggregate state to avoid replaying thousands of events for frequently accessed entities. Snapshots are created every N events (configurable per stream type) and used as a starting point for state reconstruction.

5. **Event type registry with JSON Schema validation** — the `event_type_registry` table documents every known event type and optionally stores a JSON Schema for payload validation. This enables compile-time or runtime validation of event payloads and serves as living documentation of the system's domain language.

6. **Projection subscriptions for selective processing** — the `projection_subscription` table maps event types to projections, so each projection only processes the events it cares about. This makes projection rebuilds efficient and makes it clear which projections are affected when a new event type is introduced.

7. **Organisation-scoped events with RLS** — `organisation_id` is denormalised onto every event (even though it could be derived from the stream). This enables PostgreSQL Row-Level Security policies on the event store itself, providing database-level tenant isolation even for the immutable event data.

8. **Temporal querying by design** — because every state change is an event with a timestamp, questions like "What was the chemical inventory at Site B on 15 January?" are answered by replaying events up to that timestamp. This is invaluable for regulatory investigations, insurance claims, and historical compliance reporting.

9. **AI-friendly event data** — the rich, structured event history provides an ideal training dataset for predictive SIF modelling. The `payload` JSONB field with GIN indexing enables efficient pattern queries across event types (e.g., "find all sites where inspection failures preceded SIF incidents within 90 days").

10. **Mutable platform tables** — `organisation`, `site`, and `app_user` are kept as conventional mutable tables because they are platform infrastructure rather than EHS domain data. Event sourcing the user table would add unnecessary complexity for data that changes infrequently and has no regulatory audit requirement.
