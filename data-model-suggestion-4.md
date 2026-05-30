# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Environmental Health & Safety (EHS) · Created: 2026-05-22

## Philosophy

The Graph-Relational Hybrid model combines a conventional relational database for structured CRUD operations with a graph layer for relationship-intensive queries. The relational tables handle operational data: recording incidents, tracking corrective actions, managing chemical inventories, and generating OSHA reports. The graph layer — implemented either as a property graph in PostgreSQL using `graph_node`/`graph_edge` tables, or as a parallel Neo4j/Apache AGE instance — handles questions that are fundamentally about relationships and traversals.

The EHS domain is rich in interconnected relationships that are difficult to query with standard SQL JOINs. Consider the questions an EHS manager needs to answer: "What is the chain of causation from this near-miss to the root cause, through the failed control, to the training gap, to the supervisor who signed off on the shortcut?" or "Which sites share the same hazard profile as Site A, which had a SIF incident last month?" or "Show me all incidents, inspections, chemicals, and training records connected to Worker X across all sites." These are graph traversal problems. In a normalised relational model, answering them requires complex multi-table JOINs, recursive CTEs, and often multiple queries. In a graph model, they are single-hop or multi-hop traversals.

This architecture is inspired by knowledge graph systems used in healthcare (drug interaction networks), finance (fraud detection), and supply chain (dependency mapping). For EHS, the graph layer enables AI-powered predictive analytics: a graph neural network can identify SIF precursor patterns by traversing connections between near-misses, inspection failures, training gaps, and corrective action delays — connections that are invisible in flat relational queries.

**Best for:** Organisations with complex multi-site operations where relationship analysis (causation chains, risk propagation, organisational influence patterns) and AI-powered predictive analytics are primary value drivers.

**Trade-offs:**
- (+) Relationship queries (causation chains, risk propagation) are natural and fast
- (+) AI/ML models can operate on graph structure for predictive SIF analytics
- (+) Flexible schema for relationships — new edge types require no schema migration
- (+) Visual graph exploration enables intuitive root cause and risk network analysis
- (+) Handles complex organisational hierarchies (matrix structures, contractor chains) natively
- (-) Two data layers (relational + graph) increase operational complexity
- (-) Data must be synchronised between relational tables and graph
- (-) Graph query languages (Cypher, GQL) require specialised developer skills
- (-) Harder to find developers experienced with graph databases in the EHS domain
- (-) Graph databases have weaker transactional guarantees than PostgreSQL for some operations

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 45001:2018 | The graph layer models the relationships between hazards, risks, controls, incidents, and corrective actions as specified in ISO 45001 clauses 6-10, enabling traversal of the safety management system structure |
| ISO 31000:2018 | Risk assessment bowtie analysis (threats -> barriers -> consequences) maps directly to graph edges, making bowtie visualisation a native graph query |
| ISO 14001:2015 | Environmental aspect-impact relationships, legal obligation chains, and operational control dependencies modelled as graph edges |
| OSHA 29 CFR 1904 | OSHA reporting tables remain fully relational — graph is not used for regulatory reporting |
| GHS Rev.10 | Chemical compatibility/incompatibility relationships modelled as graph edges between chemical nodes |
| GHG Protocol | Scope 3 supply chain emission relationships (upstream suppliers, downstream customers) modelled as graph edges |
| ESRS / CSRD | Double materiality relationships (financial risk <-> environmental impact) modelled as bidirectional graph edges |

---

## Relational Layer (Operational CRUD)

### Core Operational Tables

```sql
-- Organisation and site tables (same as other models)
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    country_code    CHAR(2) NOT NULL,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    settings        JSONB NOT NULL DEFAULT '{}',
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
    address         JSONB NOT NULL DEFAULT '{}',
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
    roles           TEXT[] NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, email)
);

CREATE INDEX idx_user_org ON app_user(organisation_id);
```

### Incident Management (Relational)

```sql
CREATE TABLE incident (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    incident_number VARCHAR(50) NOT NULL,
    incident_type   VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    incident_date   DATE NOT NULL,
    incident_time   TIME,
    location_detail VARCHAR(500),
    status          VARCHAR(30) NOT NULL DEFAULT 'reported',
    severity        INTEGER,
    likelihood      INTEGER,
    risk_score      INTEGER,
    is_osha_recordable BOOLEAN DEFAULT false,
    is_sif_potential BOOLEAN DEFAULT false,
    reported_by     UUID NOT NULL REFERENCES app_user(id),
    investigated_by UUID REFERENCES app_user(id),
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, incident_number)
);

CREATE INDEX idx_incident_org_date ON incident(organisation_id, incident_date DESC);
CREATE INDEX idx_incident_site ON incident(site_id);
CREATE INDEX idx_incident_status ON incident(status);
CREATE INDEX idx_incident_sif ON incident(is_sif_potential) WHERE is_sif_potential = true;

CREATE TABLE incident_person (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incident(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES app_user(id),
    person_name     VARCHAR(255) NOT NULL,
    role_in_incident VARCHAR(50) NOT NULL,
    injury_type     VARCHAR(100),
    body_part       VARCHAR(100),
    treatment_type  VARCHAR(50),
    days_away       INTEGER DEFAULT 0,
    days_restricted INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_incident_person ON incident_person(incident_id);
```

### Corrective Actions, Inspections, Chemicals, Training (Relational)

```sql
-- Corrective action (simplified — same structure as Model 1)
CREATE TABLE corrective_action (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    action_number   VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    action_type     VARCHAR(20) NOT NULL,
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium',
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    source_type     VARCHAR(30) NOT NULL,
    source_id       UUID,
    assigned_to     UUID NOT NULL REFERENCES app_user(id),
    due_date        DATE NOT NULL,
    completed_date  DATE,
    verified_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, action_number)
);

CREATE INDEX idx_capa_org_status ON corrective_action(organisation_id, status);
CREATE INDEX idx_capa_assigned ON corrective_action(assigned_to);

-- Inspection (simplified)
CREATE TABLE inspection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    inspection_number VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    template_data   JSONB NOT NULL DEFAULT '{}',
    responses       JSONB NOT NULL DEFAULT '{}',
    status          VARCHAR(20) NOT NULL DEFAULT 'in_progress',
    inspector_id    UUID NOT NULL REFERENCES app_user(id),
    score_percent   DECIMAL(5, 2),
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, inspection_number)
);

-- Chemical (simplified)
CREATE TABLE chemical (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    cas_number      VARCHAR(20),
    chemical_name   VARCHAR(500) NOT NULL,
    is_hazardous    BOOLEAN NOT NULL DEFAULT false,
    signal_word     VARCHAR(20),
    ghs_data        JSONB NOT NULL DEFAULT '{}',
    sds_data        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_chemical_cas ON chemical(cas_number);
CREATE INDEX idx_chemical_org ON chemical(organisation_id);

CREATE TABLE chemical_inventory (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    chemical_id     UUID NOT NULL REFERENCES chemical(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    storage_location VARCHAR(255) NOT NULL,
    quantity        DECIMAL(12, 4) NOT NULL,
    unit            VARCHAR(20) NOT NULL,
    epcra_threshold_exceeded BOOLEAN DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Training
CREATE TABLE training_course (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    code            VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    is_mandatory    BOOLEAN NOT NULL DEFAULT false,
    recertification_months INTEGER,
    regulatory_reference VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE training_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    course_id       UUID NOT NULL REFERENCES training_course(id),
    completion_date DATE NOT NULL,
    expiry_date     DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'completed',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Risk assessment
CREATE TABLE risk_assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    site_id         UUID REFERENCES site(id),
    assessment_number VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    assessment_type VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    hazards         JSONB NOT NULL DEFAULT '[]',
    assessed_by     UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, assessment_number)
);

-- Environmental records
CREATE TABLE environmental_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    record_type     VARCHAR(30) NOT NULL,
    ghg_scope       INTEGER,
    quantity        DECIMAL(14, 4),
    unit            VARCHAR(30),
    co2e_kg         DECIMAL(14, 4),
    record_date     DATE NOT NULL,
    record_data     JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- OSHA reporting
CREATE TABLE osha_300_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    incident_id     UUID NOT NULL REFERENCES incident(id),
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
    submitted_to_ita BOOLEAN DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_osha300_site_year ON osha_300_entry(site_id, calendar_year);
```

---

## Graph Layer

### Property Graph Schema (PostgreSQL Implementation)

The graph layer is implemented using two tables in PostgreSQL: `graph_node` and `graph_edge`. This approach avoids the operational complexity of a separate graph database while providing the core graph traversal capability. For organisations that outgrow this pattern, the same data can be migrated to Neo4j or Apache AGE.

```sql
-- =============================================================
-- GRAPH LAYER — relationship-centric queries
-- =============================================================

-- Graph node: a reference to an entity in the relational layer
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,
    -- Node types:
    -- 'incident', 'near_miss', 'inspection', 'inspection_finding',
    -- 'corrective_action', 'risk_assessment', 'hazard',
    -- 'chemical', 'training_course', 'training_gap',
    -- 'person', 'site', 'department', 'equipment',
    -- 'regulation', 'permit', 'control_measure'
    entity_id       UUID NOT NULL,                 -- FK to the relational table
    label           VARCHAR(500) NOT NULL,          -- Human-readable label
    properties      JSONB NOT NULL DEFAULT '{}',   -- Indexed node properties for graph queries
    -- {
    --   "severity": 4,
    --   "date": "2026-05-20",
    --   "status": "open",
    --   "site_id": "uuid",
    --   "risk_score": 12
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(node_type, entity_id)
);

CREATE INDEX idx_gnode_org ON graph_node(organisation_id);
CREATE INDEX idx_gnode_type ON graph_node(node_type);
CREATE INDEX idx_gnode_entity ON graph_node(entity_id);
CREATE INDEX idx_gnode_props ON graph_node USING GIN (properties jsonb_path_ops);

-- Graph edge: a typed, directed relationship between two nodes
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type       VARCHAR(100) NOT NULL,
    -- Edge types:
    -- Incident causation:
    --   'CAUSED_BY', 'CONTRIBUTED_TO', 'RESULTED_IN',
    --   'INVESTIGATED_BY', 'REPORTED_BY'
    -- Control relationships:
    --   'CONTROLLED_BY', 'FAILED_CONTROL', 'MITIGATED_BY'
    -- Corrective action:
    --   'ADDRESSES', 'ASSIGNED_TO', 'VERIFIED_BY'
    -- Hazard/risk:
    --   'EXPOSES_TO', 'ASSESSED_IN', 'HAS_HAZARD'
    -- Chemical:
    --   'INCOMPATIBLE_WITH', 'STORED_AT', 'REQUIRES_PPE'
    -- Training:
    --   'REQUIRES_TRAINING', 'COMPLETED_TRAINING', 'MISSING_TRAINING'
    -- Organisational:
    --   'WORKS_AT', 'SUPERVISES', 'REPORTS_TO', 'MEMBER_OF'
    -- Temporal:
    --   'PRECEDED_BY', 'FOLLOWED_BY', 'CONCURRENT_WITH'
    -- Regulatory:
    --   'REGULATED_BY', 'COMPLIANT_WITH', 'VIOLATES'
    properties      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "confidence": 0.85,
    --   "identified_by": "ai",
    --   "identified_at": "2026-05-20T14:00:00Z",
    --   "weight": 3,
    --   "notes": "Root cause analysis identified training gap as primary factor"
    -- }
    weight          DECIMAL(5, 2) DEFAULT 1.0,     -- Edge weight for graph algorithms
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gedge_org ON graph_edge(organisation_id);
CREATE INDEX idx_gedge_source ON graph_edge(source_node_id);
CREATE INDEX idx_gedge_target ON graph_edge(target_node_id);
CREATE INDEX idx_gedge_type ON graph_edge(edge_type);
CREATE INDEX idx_gedge_src_type ON graph_edge(source_node_id, edge_type);
CREATE INDEX idx_gedge_tgt_type ON graph_edge(target_node_id, edge_type);
CREATE INDEX idx_gedge_props ON graph_edge USING GIN (properties jsonb_path_ops);

-- Composite index for traversal queries
CREATE INDEX idx_gedge_traverse ON graph_edge(source_node_id, edge_type, target_node_id);

-- Row-Level Security on graph tables
ALTER TABLE graph_node ENABLE ROW LEVEL SECURITY;
ALTER TABLE graph_edge ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON graph_node
    USING (organisation_id = current_setting('app.current_org_id')::UUID);

CREATE POLICY tenant_isolation ON graph_edge
    USING (organisation_id = current_setting('app.current_org_id')::UUID);
```

### Graph Synchronisation

```sql
-- Tracks which relational records have been synchronised to the graph
CREATE TABLE graph_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    sync_action     VARCHAR(20) NOT NULL,          -- 'create_node', 'update_node', 'create_edge', 'delete_node'
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(entity_type, entity_id, sync_action, synced_at)
);

-- Function to create a graph node from a relational entity
CREATE OR REPLACE FUNCTION sync_incident_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO graph_node (organisation_id, node_type, entity_id, label, properties)
    VALUES (
        NEW.organisation_id,
        CASE WHEN NEW.incident_type = 'near_miss' THEN 'near_miss' ELSE 'incident' END,
        NEW.id,
        NEW.incident_number || ': ' || NEW.title,
        jsonb_build_object(
            'severity', NEW.severity,
            'likelihood', NEW.likelihood,
            'risk_score', NEW.risk_score,
            'date', NEW.incident_date,
            'status', NEW.status,
            'site_id', NEW.site_id,
            'is_sif_potential', NEW.is_sif_potential,
            'incident_type', NEW.incident_type
        )
    )
    ON CONFLICT (node_type, entity_id) DO UPDATE
    SET label = EXCLUDED.label,
        properties = EXCLUDED.properties;

    -- Create edge: incident -> site
    INSERT INTO graph_edge (organisation_id, source_node_id, target_node_id, edge_type)
    SELECT
        NEW.organisation_id,
        gn_inc.id,
        gn_site.id,
        'OCCURRED_AT'
    FROM graph_node gn_inc, graph_node gn_site
    WHERE gn_inc.entity_id = NEW.id AND gn_inc.node_type IN ('incident', 'near_miss')
      AND gn_site.entity_id = NEW.site_id AND gn_site.node_type = 'site'
    ON CONFLICT DO NOTHING;

    -- Create edge: incident -> reporter
    INSERT INTO graph_edge (organisation_id, source_node_id, target_node_id, edge_type)
    SELECT
        NEW.organisation_id,
        gn_inc.id,
        gn_person.id,
        'REPORTED_BY'
    FROM graph_node gn_inc, graph_node gn_person
    WHERE gn_inc.entity_id = NEW.id AND gn_inc.node_type IN ('incident', 'near_miss')
      AND gn_person.entity_id = NEW.reported_by AND gn_person.node_type = 'person'
    ON CONFLICT DO NOTHING;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_incident_graph
    AFTER INSERT OR UPDATE ON incident
    FOR EACH ROW EXECUTE FUNCTION sync_incident_to_graph();
```

---

## Example Graph Queries

### Incident causation chain traversal

```sql
-- Traverse the full causation chain for an incident
-- Starting from the incident, follow CAUSED_BY edges to find root causes
WITH RECURSIVE causation_chain AS (
    -- Start node: the incident
    SELECT
        gn.id AS node_id,
        gn.node_type,
        gn.label,
        gn.properties,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_node gn
    WHERE gn.entity_id = 'incident-uuid-here'
      AND gn.node_type = 'incident'

    UNION ALL

    -- Traverse CAUSED_BY and CONTRIBUTED_TO edges
    SELECT
        gn_target.id,
        gn_target.node_type,
        gn_target.label,
        gn_target.properties,
        cc.depth + 1,
        cc.path || gn_target.id
    FROM causation_chain cc
    JOIN graph_edge ge ON ge.source_node_id = cc.node_id
    JOIN graph_node gn_target ON ge.target_node_id = gn_target.id
    WHERE ge.edge_type IN ('CAUSED_BY', 'CONTRIBUTED_TO', 'FAILED_CONTROL')
      AND gn_target.id <> ALL(cc.path)  -- Prevent cycles
      AND cc.depth < 10                  -- Limit depth
)
SELECT depth, node_type, label, properties
FROM causation_chain
ORDER BY depth ASC;
```

### Find SIF precursor patterns across the graph

```sql
-- Find all near-misses, inspection failures, and training gaps
-- connected to SIF-potential incidents within 2 hops
WITH sif_incidents AS (
    SELECT gn.id AS node_id
    FROM graph_node gn
    WHERE gn.node_type = 'incident'
      AND gn.organisation_id = 'org-uuid'
      AND (gn.properties->>'is_sif_potential')::boolean = true
),
precursors AS (
    -- 1-hop: direct connections
    SELECT
        ge.edge_type,
        gn_related.node_type,
        gn_related.label,
        gn_related.properties,
        1 AS hops
    FROM sif_incidents si
    JOIN graph_edge ge ON ge.source_node_id = si.node_id OR ge.target_node_id = si.node_id
    JOIN graph_node gn_related ON gn_related.id = CASE
        WHEN ge.source_node_id = si.node_id THEN ge.target_node_id
        ELSE ge.source_node_id
    END
    WHERE gn_related.node_type IN ('near_miss', 'inspection_finding', 'training_gap', 'hazard')

    UNION ALL

    -- 2-hop: indirect connections
    SELECT
        ge2.edge_type,
        gn_2hop.node_type,
        gn_2hop.label,
        gn_2hop.properties,
        2 AS hops
    FROM sif_incidents si
    JOIN graph_edge ge1 ON ge1.source_node_id = si.node_id OR ge1.target_node_id = si.node_id
    JOIN graph_node gn_1hop ON gn_1hop.id = CASE
        WHEN ge1.source_node_id = si.node_id THEN ge1.target_node_id
        ELSE ge1.source_node_id
    END
    JOIN graph_edge ge2 ON ge2.source_node_id = gn_1hop.id OR ge2.target_node_id = gn_1hop.id
    JOIN graph_node gn_2hop ON gn_2hop.id = CASE
        WHEN ge2.source_node_id = gn_1hop.id THEN ge2.target_node_id
        ELSE ge2.source_node_id
    END
    WHERE gn_2hop.node_type IN ('near_miss', 'inspection_finding', 'training_gap', 'hazard')
      AND gn_2hop.id <> si.node_id
)
SELECT
    node_type,
    COUNT(*) AS occurrence_count,
    jsonb_agg(DISTINCT label) AS examples
FROM precursors
GROUP BY node_type
ORDER BY occurrence_count DESC;
```

### Chemical incompatibility network

```sql
-- Find all chemicals incompatible with a given chemical at a site
SELECT
    c_target.chemical_name,
    c_target.cas_number,
    ci.storage_location,
    ci.quantity,
    ci.unit,
    ge.properties->>'incompatibility_type' AS incompatibility
FROM graph_node gn_source
JOIN graph_edge ge ON ge.source_node_id = gn_source.id
    AND ge.edge_type = 'INCOMPATIBLE_WITH'
JOIN graph_node gn_target ON ge.target_node_id = gn_target.id
JOIN chemical c_target ON gn_target.entity_id = c_target.id
LEFT JOIN chemical_inventory ci ON ci.chemical_id = c_target.id
    AND ci.site_id = 'site-uuid'
WHERE gn_source.entity_id = 'chemical-uuid'
  AND gn_source.node_type = 'chemical';
```

### Bowtie analysis visualisation data

```sql
-- Generate bowtie diagram data for a hazard
-- Left side: threats -> barriers -> hazard
-- Right side: hazard -> barriers -> consequences
WITH bowtie AS (
    -- Threats (left side)
    SELECT
        'threat' AS side,
        gn_threat.label AS entity_label,
        gn_threat.node_type AS entity_type,
        ge.edge_type,
        ge.properties
    FROM graph_node gn_hazard
    JOIN graph_edge ge ON ge.target_node_id = gn_hazard.id
        AND ge.edge_type IN ('THREATENS', 'CAUSED_BY')
    JOIN graph_node gn_threat ON ge.source_node_id = gn_threat.id
    WHERE gn_hazard.entity_id = 'hazard-uuid'
      AND gn_hazard.node_type = 'hazard'

    UNION ALL

    -- Preventive barriers (left side)
    SELECT
        'preventive_barrier' AS side,
        gn_barrier.label,
        gn_barrier.node_type,
        ge.edge_type,
        ge.properties
    FROM graph_node gn_hazard
    JOIN graph_edge ge ON ge.target_node_id = gn_hazard.id
        AND ge.edge_type = 'CONTROLLED_BY'
    JOIN graph_node gn_barrier ON ge.source_node_id = gn_barrier.id
    WHERE gn_hazard.entity_id = 'hazard-uuid'
      AND gn_hazard.node_type = 'hazard'

    UNION ALL

    -- Consequences (right side)
    SELECT
        'consequence' AS side,
        gn_consequence.label,
        gn_consequence.node_type,
        ge.edge_type,
        ge.properties
    FROM graph_node gn_hazard
    JOIN graph_edge ge ON ge.source_node_id = gn_hazard.id
        AND ge.edge_type IN ('RESULTS_IN', 'EXPOSES_TO')
    JOIN graph_node gn_consequence ON ge.target_node_id = gn_consequence.id
    WHERE gn_hazard.entity_id = 'hazard-uuid'
      AND gn_hazard.node_type = 'hazard'

    UNION ALL

    -- Mitigating barriers (right side)
    SELECT
        'mitigating_barrier' AS side,
        gn_barrier.label,
        gn_barrier.node_type,
        ge.edge_type,
        ge.properties
    FROM graph_node gn_hazard
    JOIN graph_edge ge ON ge.source_node_id = gn_hazard.id
        AND ge.edge_type = 'MITIGATED_BY'
    JOIN graph_node gn_barrier ON ge.target_node_id = gn_barrier.id
    WHERE gn_hazard.entity_id = 'hazard-uuid'
      AND gn_hazard.node_type = 'hazard'
)
SELECT * FROM bowtie ORDER BY side, entity_label;
```

### Worker risk profile (all connected EHS data)

```sql
-- Find everything connected to a specific worker across the EHS graph
SELECT
    ge.edge_type AS relationship,
    gn_related.node_type,
    gn_related.label,
    gn_related.properties
FROM graph_node gn_person
JOIN graph_edge ge ON ge.source_node_id = gn_person.id OR ge.target_node_id = gn_person.id
JOIN graph_node gn_related ON gn_related.id = CASE
    WHEN ge.source_node_id = gn_person.id THEN ge.target_node_id
    ELSE ge.source_node_id
END
WHERE gn_person.entity_id = 'user-uuid'
  AND gn_person.node_type = 'person'
ORDER BY gn_related.node_type, ge.edge_type;
```

### Regulatory compliance gap detection

```sql
-- Find regulations that a site should comply with but has violations or missing controls
SELECT
    gn_reg.label AS regulation,
    ge.edge_type AS relationship,
    gn_entity.node_type AS entity_type,
    gn_entity.label AS entity_label,
    gn_entity.properties
FROM graph_node gn_site
JOIN graph_edge ge_site ON ge_site.source_node_id = gn_site.id
    AND ge_site.edge_type = 'REGULATED_BY'
JOIN graph_node gn_reg ON ge_site.target_node_id = gn_reg.id
LEFT JOIN graph_edge ge ON ge.target_node_id = gn_reg.id
    AND ge.edge_type = 'VIOLATES'
LEFT JOIN graph_node gn_entity ON ge.source_node_id = gn_entity.id
WHERE gn_site.entity_id = 'site-uuid'
  AND gn_site.node_type = 'site'
  AND gn_entity.id IS NOT NULL
ORDER BY gn_reg.label;
```

---

## AI Integration: Graph Features for Predictive Models

```sql
-- Extract graph features for SIF prediction ML model
-- These features capture the structural context around each incident

-- Feature 1: Node degree (number of connections)
SELECT
    gn.entity_id AS incident_id,
    COUNT(DISTINCT ge.id) AS total_connections,
    COUNT(DISTINCT ge.id) FILTER (WHERE ge.edge_type = 'CAUSED_BY') AS cause_count,
    COUNT(DISTINCT ge.id) FILTER (WHERE ge.edge_type = 'FAILED_CONTROL') AS failed_control_count,
    COUNT(DISTINCT ge.id) FILTER (WHERE ge.edge_type = 'ADDRESSES') AS capa_count
FROM graph_node gn
LEFT JOIN graph_edge ge ON ge.source_node_id = gn.id OR ge.target_node_id = gn.id
WHERE gn.node_type = 'incident'
  AND gn.organisation_id = 'org-uuid'
GROUP BY gn.entity_id;

-- Feature 2: Neighbourhood similarity (sites with similar hazard profiles)
WITH site_hazards AS (
    SELECT
        gn_site.entity_id AS site_id,
        array_agg(DISTINCT gn_hazard.label ORDER BY gn_hazard.label) AS hazard_set
    FROM graph_node gn_site
    JOIN graph_edge ge ON ge.source_node_id = gn_site.id AND ge.edge_type = 'HAS_HAZARD'
    JOIN graph_node gn_hazard ON ge.target_node_id = gn_hazard.id
    WHERE gn_site.node_type = 'site'
    GROUP BY gn_site.entity_id
)
SELECT
    a.site_id AS site_a,
    b.site_id AS site_b,
    -- Jaccard similarity of hazard sets
    (SELECT COUNT(*) FROM unnest(a.hazard_set) x INTERSECT SELECT COUNT(*) FROM unnest(b.hazard_set) y)::float /
    GREATEST((SELECT COUNT(*) FROM (SELECT unnest(a.hazard_set) UNION SELECT unnest(b.hazard_set)) u), 1) AS hazard_similarity
FROM site_hazards a
CROSS JOIN site_hazards b
WHERE a.site_id < b.site_id;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-tenancy & Identity | 3 | organisation, site, app_user |
| Incident Management | 2 | incident, incident_person |
| Corrective Actions | 1 | corrective_action |
| Inspections | 1 | inspection (JSONB responses and template) |
| Chemical Management | 2 | chemical, chemical_inventory |
| Training | 2 | training_course, training_record |
| Risk Assessment | 1 | risk_assessment (JSONB hazards) |
| Environmental & ESG | 1 | environmental_record |
| OSHA Reporting | 1 | osha_300_entry |
| Graph Layer | 3 | graph_node, graph_edge, graph_sync_log |
| **Total** | **~17** | Relational tables + graph layer tables |

---

## Key Design Decisions

1. **PostgreSQL-native graph implementation** — using `graph_node`/`graph_edge` tables rather than a separate graph database (Neo4j) keeps the stack simple. All data lives in one PostgreSQL instance, transactions span both relational and graph operations, and there is no synchronisation lag. For organisations that outgrow this, the data model maps directly to Neo4j property graph format for future migration.

2. **Graph as a secondary layer, not primary storage** — the relational tables remain the system of record for CRUD operations and regulatory reporting. The graph layer is a derived, queryable representation of relationships. If the graph is corrupted, it can be rebuilt from relational data via the sync triggers and `graph_sync_log`.

3. **Trigger-based synchronisation** — PostgreSQL triggers (`AFTER INSERT OR UPDATE`) automatically create and update graph nodes and edges when relational records change. This eliminates manual synchronisation code and ensures the graph stays current with relational data.

4. **Edge types as a flexible vocabulary** — edge types (`CAUSED_BY`, `CONTROLLED_BY`, `INCOMPATIBLE_WITH`, etc.) are string values, not foreign keys to a reference table. This enables the rapid introduction of new relationship types — including AI-discovered relationships — without schema changes.

5. **Weighted edges for graph algorithms** — the `graph_edge.weight` column enables PageRank, shortest-path, and community detection algorithms. For SIF prediction, weight represents the strength or confidence of a relationship (e.g., an AI-identified causation link might have weight 0.7 while a human-verified one has weight 1.0).

6. **AI-identified edges with confidence scores** — the `graph_edge.properties` JSONB includes `confidence` and `identified_by` fields, distinguishing between human-created relationships and AI-inferred ones. This is essential for the predictive SIF modelling use case: the ML model can create "PRECURSOR_OF" edges between near-misses and hazards with a confidence score, which safety managers can review and confirm.

7. **Bowtie analysis as graph traversal** — ISO 31000 bowtie diagrams (threats -> preventive barriers -> hazard -> mitigating barriers -> consequences) map naturally to graph traversals. The graph query returns the complete bowtie structure in a single query, compared to multiple JOINs in a relational model.

8. **Chemical incompatibility as graph edges** — chemical incompatibility relationships (e.g., "acetone is incompatible with hydrogen peroxide") are naturally bidirectional graph edges. The graph enables queries like "Given all chemicals at Site B, which pairs are stored within the same location and are incompatible?" — a query that would require self-JOINs and cross-products in a relational model.

9. **Worker risk profile as ego network** — a worker's complete EHS profile (incidents involved in, training completed/missing, chemicals exposed to, inspections participated in, corrective actions assigned) is their "ego network" in the graph. This single traversal query replaces 6-8 separate relational queries.

10. **Graph features for ML model input** — the graph structure provides features (node degree, neighbourhood composition, path length to SIF incidents) that are not available in flat relational data. These structural features significantly improve the accuracy of predictive SIF models by capturing the systemic context around each data point.
