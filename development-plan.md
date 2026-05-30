# Environmental Health & Safety (EHS) — Phased Development Plan

> Project: 252-environmental-health-safety · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. It builds an AI-native, open-source EHS platform targeting the SMB/mid-market gap between simple checklist tools (iAuditor) and unaffordable enterprise suites (VelocityEHS, Cority, Intelex). The platform is multi-tenant, self-hostable, offline-capable on mobile, standards-aligned (ISO 45001, OSHA 29 CFR 1904, GHS Rev.10, GHG Protocol, GRI/ESRS), and exposes both a published OpenAPI 3.1 REST API and a first-in-category MCP server.

---

## Core Requirements (synthesis)

- **What it does**: Consolidates incident/near-miss management, root-cause + CAPA, audits/inspections, risk assessment, SDS/chemical management, OSHA recordkeeping, training compliance, environmental compliance, and ESG metrics into one AI-augmented system.
- **Who uses it**: EHS managers and corporate safety directors (industrial, manufacturing, construction); sustainability/ESG officers; chemical/compliance teams; field workers (mobile, offline).
- **Differentiators**: Open source (no comparable OSS incumbent), SMB-accessible, offline-first field capture, AI woven through (incident analysis, root-cause suggestion, predictive SIF scoring, permit obligation extraction, NL SDS assistant), and an MCP server — none of which the closed-source incumbents offer openly.
- **MVP**: incidents/near-miss, AI root-cause assist, CAPA, audits/inspections (offline mobile), OSHA 300/300A + ITA export, risk matrix builder, SDS/chemical inventory.
- **Post-MVP**: training/recert, predictive SIF, regulatory gap analysis, voice-to-text, environmental compliance, ESG (GRI/CSRD), public REST API, computer-vision PPE, NL SDS assistant, contractor management, multi-language.
- **Deployment**: Self-hosted (Docker Compose) and cloud, with the same image. API + MCP server are first-class deliverables.
- **Integration surface**: OSHA ITA REST API, ECHA REACH/C&L (public), LLM providers (via a provider-abstraction gateway), HRIS/ERP/LMS via OpenAPI, OIDC SSO, webhooks.
- **Standards**: ISO 45001/14001/31000, OSHA 29 CFR 1904 (300/300A/301), GHS Rev.10, REACH/RoHS, EPCRA Tier II, GHG Protocol, GRI 403/305, ESRS, OpenAPI 3.1, JSON Schema 2020-12, OAuth 2.0/OIDC (RFC 6749/7519), RFC 4180 (CSV), OWASP Top 10, GDPR, MCP.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **Python 3.12** | The differentiating value is AI (LLM root-cause/permit extraction, predictive SIF ML, CV PPE, NL assistant). Python has the richest ML/LLM ecosystem (scikit-learn, transformers, the LLM SDKs, pypdf, ultralytics) and is the language of the only open competitor artefact (Intelex's PyPI SDK). |
| API framework | **FastAPI** | Native async (needed for streaming LLM calls and webhooks), Pydantic v2 models, and automatic OpenAPI 3.1 generation — `standards.md` mandates publishing an OpenAPI 3.1 spec from day one. |
| Data validation | **Pydantic v2** | Single source of truth for request/response schemas, JSON Schema (2020-12) emission for the published schema repo, and application-layer validation of JSONB extension fields. |
| ORM / DB toolkit | **SQLAlchemy 2.0 + Alembic** | 2.0 typed ORM for the relational spine; Alembic for migrations. Supports Postgres `JSONB` and `GIN` indexes for the hybrid model. |
| Database | **PostgreSQL 16** | Hybrid relational + JSONB model (data-model-suggestion-3) requires Postgres `JSONB`, `GIN` indexes, Row-Level Security for tenant isolation, recursive CTEs for location hierarchy, and `pgvector` for the SDS/regulatory RAG assistant. SQLite is used only for unit-test fixtures where Postgres features aren't exercised. |
| Vector store | **pgvector extension** | Keeps embeddings (SDS chunks, regulatory text) inside Postgres — one datastore for self-hosters, no extra service to operate. |
| Task queue | **Celery + Redis** | Async workloads: LLM calls, OSHA ITA submission, ITA polling, recert reminders, SIF model scoring, CV inference, embedding generation, permit parsing. Redis doubles as broker + result backend + cache. |
| Scheduler | **Celery Beat** | Cron-style jobs: nightly recert/permit-due reminders, expiry status recomputation, scheduled SIF re-scoring, annual ITA reminders. |
| LLM access | **Provider-abstraction gateway (`AIProvider` interface)** | Pluggable: Anthropic, OpenAI, and a local Ollama provider so self-hosters can run fully offline. Avoids hard vendor lock-in; defaults read from env. |
| Embeddings | **Configurable embedding provider (same gateway)** | Default to a local sentence-transformers model for self-host; cloud embeddings optional. |
| Object storage | **S3-compatible (boto3) + local-disk driver** | Incident photos, SDS PDFs, audit photos, certificates. Local driver for zero-ops self-host; S3/MinIO for cloud. |
| Frontend | **Next.js 15 (React 19, TypeScript) + Tailwind + shadcn/ui** | Dashboard + role-based UI for managers; server components for fast dashboards. Consumes the same REST API. |
| Mobile / offline field capture | **PWA (Next.js) with IndexedDB + Service Worker + sync engine** | `features.md` flags offline-first as a top underserved area. A PWA avoids dual native codebases; client generates UUID PKs (per data-model decision 1) and syncs via a conflict-aware sync endpoint. |
| Auth | **OAuth 2.0 / OIDC (Authlib) + local password (argon2)** | RFC 6749/7519. Local accounts for SMB self-host; OIDC (Entra ID/Okta) for enterprise SSO, per `standards.md`. JWT access tokens. |
| Containerisation | **Docker + docker-compose** | Self-host is a primary deployment mode; one `docker-compose up` brings up API, worker, beat, Postgres, Redis, MinIO, web. |
| Testing | **pytest + pytest-asyncio + httpx.AsyncClient + testcontainers** | Unit + integration; testcontainers spins real Postgres/Redis for integration tests. `respx`/`responses` mock OSHA ITA, ECHA, and LLM HTTP calls. |
| Frontend testing | **Vitest + Playwright** | Component tests and offline-sync E2E. |
| Lint / format / types | **ruff + black + mypy (strict)** (Python); **eslint + prettier + tsc** (web) | Standard, fast, CI-enforceable. |
| Package manager | **uv** (Python); **pnpm** (web) | Fast, reproducible lockfiles. |
| API spec / schemas | **OpenAPI 3.1 (FastAPI-generated) + exported JSON Schemas** | Published like SafetyCulture's `api-json-schemas`; the canonical interop artefact, addressing the "no open EHS exchange standard" gap. |
| MCP server | **Official Python MCP SDK** | First-in-category MCP server exposing read tools over incidents, CAPA, SDS, risk register, compliance status. |
| Key libraries | `pypdf`/`pdfplumber` (SDS + permit parsing), `python-dateutil`, `croniter`, `ultralytics` (YOLO PPE CV, backlog), `openai-whisper`/provider STT (voice), `sentence-transformers` (embeddings), `reportlab` (300A PDF), `python-jose`/`authlib` (JWT/OIDC), `argon2-cffi` | Domain-specific needs from the feature set. |

### Project Structure

```
ehs/
├── pyproject.toml                     # uv-managed; ruff/black/mypy/pytest config
├── uv.lock
├── Dockerfile                         # multi-stage: api/worker/beat share one image
├── docker-compose.yml                 # api, worker, beat, postgres(pgvector), redis, minio, web
├── docker-compose.override.example.yml
├── .env.example
├── alembic.ini
├── Makefile                           # dev, test, lint, migrate, seed, spec
├── README.md
├── migrations/                        # Alembic
│   └── versions/
├── schemas/                           # exported JSON Schemas + openapi.json (generated)
├── src/
│   └── ehs/
│       ├── main.py                    # FastAPI app factory, router registration
│       ├── config.py                  # Pydantic Settings (env)
│       ├── db/
│       │   ├── base.py                # SQLAlchemy Base, session, engine
│       │   ├── rls.py                 # Row-Level Security helpers / tenant context
│       │   └── models/                # ORM models, grouped by domain
│       │       ├── core.py            # tenant, org, user, role, permission, site, location
│       │       ├── incident.py
│       │       ├── capa.py
│       │       ├── audit.py           # audit templates, runs, responses
│       │       ├── risk.py
│       │       ├── chemical.py
│       │       ├── training.py
│       │       ├── osha.py
│       │       ├── environmental.py
│       │       ├── esg.py
│       │       └── ai.py              # ai_job, sif_score, embeddings
│       ├── schemas/                   # Pydantic request/response models per domain
│       ├── api/
│       │   ├── deps.py                # auth, tenant, pagination, permission deps
│       │   ├── errors.py             # RFC 9457 problem+json handlers
│       │   └── routers/              # one router module per domain + /sync, /ai, /reports
│       ├── services/                  # business logic (framework-agnostic)
│       │   ├── incidents.py
│       │   ├── capa.py
│       │   ├── audits.py
│       │   ├── risk.py
│       │   ├── chemical.py
│       │   ├── training.py
│       │   ├── osha.py
│       │   ├── environmental.py
│       │   ├── esg.py
│       │   └── sync.py               # offline sync reconciliation
│       ├── ai/
│       │   ├── provider.py           # AIProvider ABC + Anthropic/OpenAI/Ollama impls
│       │   ├── prompts/              # versioned prompt templates
│       │   ├── root_cause.py
│       │   ├── permit_extract.py
│       │   ├── sif_model.py          # feature build + scorer (sklearn)
│       │   ├── ppe_vision.py         # YOLO inference wrapper (backlog)
│       │   ├── assistant.py          # RAG NL SDS/compliance assistant
│       │   └── transcribe.py         # voice-to-text
│       ├── integrations/
│       │   ├── osha_ita.py           # OSHA ITA REST client
│       │   ├── echa.py               # REACH/C&L public API client
│       │   ├── storage.py            # S3 + local drivers
│       │   └── webhooks.py
│       ├── tasks/                     # Celery tasks + beat schedule
│       │   ├── app.py
│       │   ├── ai_tasks.py
│       │   ├── reminders.py
│       │   └── ita_tasks.py
│       ├── reports/
│       │   ├── osha_300.py           # 300 log + 300A CSV (RFC 4180) + PDF
│       │   ├── ghg.py                # GHG Protocol calc
│       │   └── esg_export.py         # GRI/ESRS mapping export
│       ├── mcp/
│       │   └── server.py             # MCP server exposing read tools
│       └── seed/
│           └── reference_data.py     # GHS H/P codes, OSHA injury types, emission factors, body parts
├── tests/
│   ├── conftest.py                    # fixtures: db, tenant, auth, mock providers
│   ├── unit/
│   ├── integration/
│   └── e2e/
└── web/
    ├── package.json
    ├── app/                           # Next.js app router
    ├── components/
    ├── lib/api-client.ts              # typed client generated from openapi.json
    ├── lib/offline/                   # IndexedDB store + sync engine + service worker
    └── tests/
```

The structure groups by concern (models, schemas, routers, services, ai, integrations, tasks, reports) so each phase adds files without restructuring.

---

## Phase 1: Foundation — Project Skeleton, Multi-Tenancy, Auth, RBAC

### Purpose
Establish the runnable application shell, the tenant-isolated data foundation, authentication, and role-based access control. Every later phase attaches domain entities to a tenant and gates endpoints behind permissions, so this must be correct and stable first. After this phase a developer can run the stack, register a tenant, log in, and call an authenticated health endpoint.

### Tasks

#### 1.1 — Project scaffolding, config, and containerisation

**What**: Create the repo skeleton, `Settings`, Docker/compose, Makefile, and CI lint/type/test gates.

**Design**:
- `config.py` using `pydantic_settings.BaseSettings`:
```python
class Settings(BaseSettings):
    database_url: str
    redis_url: str = "redis://redis:6379/0"
    jwt_secret: str
    jwt_access_ttl_seconds: int = 3600
    jwt_refresh_ttl_seconds: int = 1209600
    storage_driver: Literal["local", "s3"] = "local"
    s3_endpoint: str | None = None
    s3_bucket: str = "ehs"
    ai_provider: Literal["anthropic", "openai", "ollama", "null"] = "null"
    ai_model: str = "claude-sonnet"
    ai_api_key: str | None = None
    embedding_provider: Literal["local", "openai", "null"] = "local"
    cors_origins: list[str] = []
    model_config = SettingsConfigDict(env_prefix="EHS_", env_file=".env")
```
- `main.py`: `create_app()` factory registering routers, exception handlers (RFC 9457 `application/problem+json`), CORS, and a `/healthz` (liveness) + `/readyz` (DB + Redis ping) endpoint.
- `Dockerfile` multi-stage (builder installs via `uv`, runtime slim). `docker-compose.yml` services: `api`, `worker`, `beat`, `postgres` (image with pgvector), `redis`, `minio`, `web`.
- `Makefile` targets: `dev`, `test`, `lint`, `typecheck`, `migrate`, `seed`, `spec` (dumps `schemas/openapi.json`).
- CI runs ruff + black --check + mypy --strict + pytest.

**Testing**:
- `Unit: Settings loads from env with EHS_ prefix → fields populated, defaults applied for unset optionals.`
- `Unit: missing required EHS_DATABASE_URL → ValidationError naming the field.`
- `Integration: GET /healthz → 200 {"status":"ok"}.`
- `Integration (testcontainers): GET /readyz with DB+Redis up → 200; with Redis stopped → 503 problem+json.`
- `E2E (CI): docker compose build succeeds; make lint/typecheck/test pass.`

#### 1.2 — Database base, tenant context, and Row-Level Security

**What**: SQLAlchemy 2.0 base, async session management, and a tenant-scoping mechanism enforced by Postgres RLS.

**Design**:
- `db/base.py`: declarative `Base`, async engine, `async_sessionmaker`, a `get_session` dependency.
- Shared mixins:
```python
class UUIDPKMixin:
    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())

class TenantMixin:
    tenant_id: Mapped[UUID] = mapped_column(ForeignKey("tenant.id"), index=True)
```
- UUID PKs everywhere (per data-model decision 1 — enables offline clients to generate IDs).
- `db/rls.py`: every request sets `SET LOCAL app.current_tenant_id = :tid` and `app.current_user_id` within the transaction. RLS policies on every tenant-scoped table: `USING (tenant_id = current_setting('app.current_tenant_id')::uuid)`. Reference tables (GHS codes, injury types, emission factors) are tenant-agnostic and have no RLS.
- Alembic configured for autogenerate; first migration creates `tenant`, `organisation`.

**Testing**:
- `Integration (testcontainers): insert rows for tenant A and B; query as tenant A → only A's rows returned (RLS enforced).`
- `Integration: attempt cross-tenant UPDATE by id while tenant context = A on a B row → 0 rows affected.`
- `Unit: TenantMixin/TimestampMixin produce expected columns via metadata reflection.`

#### 1.3 — Users, roles, permissions (RBAC) with site scoping

**What**: User accounts, system + custom roles, permission catalogue, and site-scoped role grants.

**Design**:
- Tables (from data-model-suggestion-1, adapted): `app_user`, `role`, `permission`, `role_permission`, `user_role(user_id, role_id, site_id NULL=all-sites)`.
- Permission codes are `"<module>.<action>"` (e.g., `incident.create`, `audit.approve`, `osha.submit`). Seeded catalogue with `module` grouping.
- Seeded system roles: `admin`, `ehs_manager`, `auditor`, `field_worker`, `viewer`.
- `services/` permission check:
```python
def require_permission(user: User, code: str, site_id: UUID | None = None) -> None:
    # grant matches if user has a role with the permission AND
    # (grant.site_id IS NULL OR grant.site_id == site_id)
```
- Passwords hashed with argon2id.

**Testing**:
- `Unit: user with ehs_manager role (site-scoped to S1) + permission incident.approve → require_permission(...,S1) passes; ...(...,S2) raises Forbidden.`
- `Unit: user with all-sites grant → passes for any site_id.`
- `Unit: argon2 verify of correct/incorrect password → True/False.`

#### 1.4 — Authentication: local login, JWT, OIDC, refresh

**What**: Login endpoints issuing JWT access/refresh tokens; OIDC login; auth dependency that resolves the current user + tenant.

**Design**:
- `POST /auth/login` (email+password) → `{access_token, refresh_token, token_type:"bearer"}`. Access JWT claims: `sub`(user), `tid`(tenant), `roles`, `exp`. RFC 7519.
- `POST /auth/refresh` → new access token.
- `GET /auth/oidc/login` → redirect; `GET /auth/oidc/callback` → provisions/links user by `external_id`, issues tokens. Authlib OIDC client; provider config from env.
- `api/deps.py`: `get_current_user` (decode JWT, load user, set RLS tenant context), `require(code)` dependency factory.

**Testing**:
- `Integration: login with valid creds → 200 + tokens; decoded access token has correct sub/tid.`
- `Integration: login with wrong password → 401 problem+json, generic message (no user enumeration).`
- `Integration: protected endpoint without token → 401; with expired token → 401; with valid token → 200.`
- `Integration (mocked OIDC via respx): callback with valid code → user provisioned, tokens returned.`
- `Integration: refresh with valid refresh token → new access token; with access token used as refresh → 401.`

---

## Phase 2: Core Domain — Sites, Locations, Reference Data, Hybrid Model & Audit Trail

### Purpose
Build the shared spine every module references: the organisation→site→location hierarchy, the seeded reference data (GHS codes, OSHA injury types, body parts, hazard categories, emission factors), the hybrid relational+JSONB pattern with application-layer JSON Schema validation, and the global change-audit trail required by ISO 45001/OSHA. After this phase, domain modules can be added quickly because their foundational dependencies and patterns exist.

### Tasks

#### 2.1 — Organisation, site, location hierarchy

**What**: CRUD for organisations, sites, and a self-referential location tree.

**Design**:
- `site` columns per data-model-suggestion-1 (incl. `naics_code`, `establishment_size`, `country_code`, lat/long) — these are required for OSHA ITA later. `jurisdiction` reference table (ISO 3166 codes + `regulatory_body`).
- `location` adjacency list (`parent_id`); endpoint `GET /sites/{id}/locations?tree=true` returns nested tree via recursive CTE.
- Endpoints: `GET/POST/PATCH /organisations`, `/sites`, `/sites/{id}/locations`. All tenant-scoped and permission-gated.

**Testing**:
- `Integration: create site then nested locations (building→floor→area) → GET ?tree=true returns 3-level nested structure.`
- `Integration: recursive descendant query for Building returns all sub-locations.`
- `Unit: site missing required country_code → 422.`

#### 2.2 — Hybrid relational+JSONB pattern + extension-field validation

**What**: Establish the "relational spine, flexible edges" convention (data-model-suggestion-3) with JSON Schema validation of JSONB columns at the application layer.

**Design**:
- Convention: regulatory/queryable fields are typed columns; jurisdiction/custom fields live in a `regulatory_data JSONB` and/or `custom_fields JSONB` column with a `GIN` index.
- A `custom_field_definition` table per tenant+entity defines allowed keys, types, and a JSON Schema fragment. `services/extension.py::validate_extension(entity, payload)` validates JSONB against the composed JSON Schema (Draft 2020-12) before persist.
- Helper to query JSONB: `WHERE custom_fields @> :filter` with GIN index.

**Testing**:
- `Unit: payload conforming to tenant's custom_field schema → validate_extension passes.`
- `Unit: payload with unknown key / wrong type → ValidationError listing offending key.`
- `Integration: filter incidents by custom_fields @> {"shift":"night"} → returns only matching rows (GIN index used; assert via EXPLAIN in optional real-DB test).`

#### 2.3 — Reference data seeding

**What**: Idempotent seeding of all shared reference data.

**Design**:
- `seed/reference_data.py` populates: `ghs_hazard_statement` (H-codes + signal words), `ghs_precautionary_statement` (P-codes), `injury_type`, `body_part`, `hazard_category` (safety/health/environmental/ergonomic/psychosocial), `incident_type`, `root_cause_method` (5-why, fishbone, fault_tree, bowtie, ICAM), `emission_source_type` (Scope 1/2/3), starter `emission_factor` rows (EPA/DEFRA with `valid_from`), and `esg_metric_definition` rows (GRI 403/305, ESRS E1/S1). Data shipped as committed CSV/JSON under `seed/data/`.
- `make seed` is idempotent (upsert by natural `code`).

**Testing**:
- `Integration: run seed twice → row counts stable, no duplicate-key errors.`
- `Unit: every emission_factor has valid_from and a known scope (1/2/3).`
- `Unit: GHS H-codes have valid signal_word in {danger,warning,null}.`

#### 2.4 — Global change-audit trail

**What**: Generic `audit_log` capturing INSERT/UPDATE/DELETE with before/after JSONB, satisfying ISO 45001/OSHA recordkeeping (data-model-suggestion-1 decision 8).

**Design**:
- `audit_log(tenant_id, table_name, record_id, action, old_values JSONB, new_values JSONB, changed_by, changed_at, ip_address, user_agent)`.
- Implemented via a SQLAlchemy session event hook (`before_flush`) that records dirty/new/deleted instances using the request-scoped user + IP. (Postgres trigger variant documented as alternative.)
- Endpoint `GET /audit-log?table=&record_id=` (admin-only).

**Testing**:
- `Integration: update an incident's severity → audit_log row with old/new severity and changed_by set.`
- `Integration: delete a CAPA comment → audit_log DELETE row with old_values populated.`
- `Integration: GET /audit-log as non-admin → 403.`

---

## Phase 3: Incident & Near-Miss Management + AI Root-Cause (Core Value)

### Purpose
Deliver the heart of the product: structured incident/near-miss capture, multi-person involvement, attachments, investigation with multiple RCA methods, and the first AI feature — AI-assisted cause suggestion from incident text. This is the primary differentiator versus iAuditor and the entry point to OSHA recordkeeping and CAPA.

### Tasks

#### 3.1 — Incident & near-miss capture

**What**: Create/list/update incidents with involved persons and attachments.

**Design**:
- `incident` (data-model-suggestion-1 + hybrid `regulatory_data`/`custom_fields` JSONB): `incident_type_id`, `reference_number` (auto `INC-{site_code}-{YYYY}-{seq}`), `title`, `description`, `occurred_at`, `severity` enum, `psif_potential`, `status` state machine, `assigned_to`.
- State machine: `reported → under_investigation → root_cause_identified → corrective_action_assigned → closed`; illegal transitions rejected by `services/incidents.py::transition()`.
- `incident_involved_person` (multiple, per OSHA 301): name, person_type, `injury_type_id`, `body_part_id`, `days_away`, `days_restricted`, `outcome`.
- `incident_attachment` via storage driver (presigned upload for S3; direct for local).
- Endpoints: `POST /incidents`, `GET /incidents` (filter by site/status/date/severity, paginated), `GET/PATCH /incidents/{id}`, `POST /incidents/{id}/persons`, `POST /incidents/{id}/attachments`, `POST /incidents/{id}/transition`.

**Testing**:
- `Unit: reference_number generator increments per site+year; concurrent inserts don't collide (DB unique constraint test).`
- `Unit: transition reported→closed (skipping states) → InvalidTransition error.`
- `Integration: create incident with 2 involved persons → GET returns both with injury details.`
- `Integration: list filtered by severity=critical&status=reported → only matching.`
- `Integration: attachment upload (local driver) → file stored, metadata row created.`

#### 3.2 — Investigation & root-cause records

**What**: Investigations using a chosen RCA method, with structured findings.

**Design**:
- `investigation(incident_id, method_id, lead_investigator, findings, root_cause_summary, ai_suggested_causes, status)`.
- `investigation_finding(finding_type ∈ {direct_cause, contributing_factor, root_cause, systemic_issue}, description, sequence_order)`.
- Completing an investigation auto-advances incident status to `root_cause_identified`.
- Endpoints: `POST /incidents/{id}/investigation`, `POST /investigations/{id}/findings`, `PATCH /investigations/{id}` (complete).

**Testing**:
- `Integration: open investigation (5-why) + add findings → complete → incident status advances.`
- `Unit: finding_type validated against enum.`

#### 3.3 — AI provider gateway

**What**: Pluggable LLM abstraction used by all AI features.

**Design**:
```python
class AIProvider(ABC):
    @abstractmethod
    async def complete(self, system: str, user: str, *, json_schema: dict | None = None,
                       temperature: float = 0.2) -> AICompletion: ...
    @abstractmethod
    async def embed(self, texts: list[str]) -> list[list[float]]: ...

@dataclass
class AICompletion:
    text: str
    parsed: dict | None      # when json_schema requested (structured output)
    model: str
    tokens_in: int
    tokens_out: int
```
- Implementations: `AnthropicProvider`, `OpenAIProvider`, `OllamaProvider`, `NullProvider` (deterministic stub for tests/CI). Selected by `Settings.ai_provider`.
- All AI calls run as Celery tasks and write to an `ai_job(id, tenant_id, kind, status, input_ref, output JSONB, model, tokens_in, tokens_out, error, created_at)` table for auditability and cost tracking.

**Testing**:
- `Unit: NullProvider.complete returns deterministic stub matching requested json_schema shape.`
- `Integration (respx-mocked Anthropic): complete with json_schema → parsed dict returned; tokens recorded.`
- `Unit: provider factory returns correct impl per settings; "null" in test env.`

#### 3.4 — AI root-cause suggestion

**What**: Generate candidate causes + likely incident classification from free-text incident description.

**Design**:
- `ai/root_cause.py::suggest(incident) -> RootCauseSuggestion` enqueues an `ai_job`; result stored on `investigation.ai_suggested_causes` and returned via `GET /incidents/{id}/ai/root-cause`.
- Prompt template (`ai/prompts/root_cause.md`) — system: "You are an EHS investigator assistant. Given an incident description, classify likely incident type, list 3-5 candidate root causes grouped by category (human, equipment, process, environment, management-system), and cite which words triggered each. Output strict JSON." Structured-output JSON Schema: `{classification, causes:[{category, hypothesis, evidence_span}], confidence}`.
- Endpoint `POST /incidents/{id}/ai/root-cause` (enqueue) + `GET` (poll job/result). Human always confirms; AI output is advisory and flagged as such.

**Testing**:
- `Unit (NullProvider): suggest returns schema-valid RootCauseSuggestion with ≥1 cause.`
- `Integration (respx-mocked LLM): POST enqueues ai_job; on task run, GET returns parsed causes; ai_job.status=completed.`
- `Integration: LLM error → ai_job.status=failed, GET returns 200 with status failed (no crash).`

---

## Phase 4: CAPA, Audits & Inspections, Risk Assessment

### Purpose
Add the management workflows that close the loop on incidents and proactive safety: corrective/preventive actions (assignable, due-date tracked, verification of effectiveness), audit/inspection templates with scored runs and non-conformance flagging, and a configurable risk-matrix builder with a risk register. CAPAs are polymorphic so they can originate from incidents, audits, inspections, or risk assessments.

### Tasks

#### 4.1 — CAPA management

**What**: Polymorphic corrective/preventive actions with lifecycle, assignment, and effectiveness verification.

**Design**:
- `capa` (data-model-suggestion-1): `source_type ∈ {incident,audit,inspection,risk_assessment,complaint,regulatory}` + `source_id` (polymorphic, no FK), `capa_type`, `priority`, `status` state machine (`open → in_progress → pending_verification → verified_effective → closed`, plus derived `overdue`), `assigned_to`, `due_date`, `effectiveness_result`.
- Overdue derivation: a Celery Beat job flips status indicator + emits notification when `due_date < today` and not completed.
- `capa_comment` thread.
- Linking a CAPA to an incident auto-advances incident to `corrective_action_assigned`.
- Endpoints: `POST /capas`, `GET /capas` (filter by status/assignee/due/source), `PATCH /capas/{id}`, `POST /capas/{id}/comments`, `POST /capas/{id}/verify`.

**Testing**:
- `Unit: status transition open→verified_effective (skipping pending_verification) → rejected.`
- `Integration: create CAPA from incident → incident status advances; CAPA appears in source filter.`
- `Integration: beat overdue job marks past-due open CAPA as overdue and creates a notification.`

#### 4.2 — Audit/inspection templates and scored runs (offline-ready data shape)

**What**: Build templates (sections→questions), run audits/inspections with scored responses, photos, and non-conformance flags.

**Design**:
- `audit_template`, `audit_template_section`, `audit_template_question(response_type ∈ {yes_no,scale,text,photo,multi_choice,numeric}, is_required, guidance_notes)`, with `standard_ref` (e.g. `ISO 45001:2018`).
- `audit(template_id, site_id, location_id, auditor_id, scheduled_date, status, overall_score)`; `audit_response(question_id, response_value, score, notes, photo_key, flagged)`.
- Scoring service computes `overall_score` from weighted responses; `flagged` responses can spawn CAPAs (`source_type=audit`).
- Responses carry client-generated UUIDs and `client_updated_at` to support Phase 6 offline sync.
- Endpoints: `POST /audit-templates` (+sections/questions), `POST /audits`, `PATCH /audits/{id}` (submit responses), `POST /audits/{id}/complete`, `POST /audit-responses/{id}/create-capa`.

**Testing**:
- `Integration: create template (2 sections, 5 questions) → run audit → submit responses → overall_score computed correctly.`
- `Unit: required question unanswered on complete → validation error listing question.`
- `Integration: flag a response → create-capa → CAPA with source_type=audit, source_id=audit.id.`

#### 4.3 — Risk matrix builder and risk register

**What**: Configurable NxN risk matrices and risk assessments with initial vs residual scoring, aligned to ISO 31000.

**Design**:
- `risk_matrix(grid_size 3/4/5)`, `risk_matrix_cell(likelihood, severity, risk_level, colour_code)`.
- `risk_assessment(assessment_type ∈ {jsa,hra,process_hazard,environmental_aspect}, matrix_id, status)`, `risk_assessment_item(hazard_category_id, hazard_description, existing_controls, initial_likelihood/severity/level, additional_controls, residual_*)`.
- `risk_level` derived from matrix cell at write time.
- Risk register = `GET /risk-items?site=&level=&category=` aggregation across assessments.
- Endpoints: `POST /risk-matrices`, `POST /risk-assessments` (+items), `GET /risk-register`.

**Testing**:
- `Unit: 5x5 matrix cell lookup (likelihood=4,severity=5) → returns configured risk_level.`
- `Integration: assessment item with initial high + residual low after additional controls → both levels stored and shown in register.`
- `Unit: likelihood/severity out of grid range → 422.`

---

## Phase 5: OSHA Recordkeeping, ITA Export, SDS & Chemical Management

### Purpose
Complete the MVP scope with the two regulatory pillars: US OSHA recordkeeping (300 log, 300A summary, ITA CSV/API submission) derived from incident data, and GHS-aligned SDS + chemical inventory management with EPCRA threshold tracking. These are table-stakes that iAuditor lacks and that justify the platform for industrial/chemical operations.

### Tasks

#### 5.1 — OSHA 300 log generation from incidents

**What**: Materialise OSHA 300 entries from recordable incidents, with manual override.

**Design**:
- `osha_300_entry` stored as dedicated columns (data-model decision 5) field-for-field with OSHA Form 300 (columns A–M): case_number, employee_name/`is_privacy_case`, job_title, date_of_injury, where_occurred, description, classification flags (death/days-away/transfer/other), num_days_away/restricted, `injury_illness_type` (1–6).
- `services/osha.py::generate_300(site, year)` maps recordable incidents (via `incident_type.is_osha_recordable` and involved-person outcomes) into entries; existing entries are preserved (regulatory snapshot independent of later incident edits).
- Endpoints: `POST /osha/300/{site_id}/{year}/generate`, `GET /osha/300/{site_id}/{year}`, `PATCH /osha/300-entries/{id}`.

**Testing**:
- `Unit: incident with lost_time outcome (5 days away) → 300 entry days_away_from_work=true, num_days_away=5, correct column flags.`
- `Unit: is_privacy_case=true → employee_name omitted, rendered as "Privacy Case".`
- `Integration: regenerate after incident description edit → existing entry unchanged unless re-synced explicitly.`

#### 5.2 — 300A summary, CSV (RFC 4180) and PDF export, ITA submission

**What**: Annual 300A summary with certification, OSHA-format CSV + printable PDF, and automated ITA submission via the OSHA ITA REST API.

**Design**:
- `osha_300a_summary` with all totals (deaths, days-away, transfers, other; injury/skin/respiratory/poisoning/hearing/other-illness), `avg_employees`, `total_hours_worked`, certification fields, and ITA submission tracking (`ita_submitted`, `ita_submitted_at`, `ita_confirmation_id`).
- `services/osha.py::compute_300a(site, year)` aggregates 300 entries into totals.
- `reports/osha_300.py`: CSV writer conforming to RFC 4180 and OSHA ITA bulk-upload column spec; PDF via reportlab.
- `integrations/osha_ita.py`: client for ITA REST API (auth via employer account/API key from env); submission and confirmation polling run as Celery tasks; targets the ITA Preview environment in non-prod (configurable base URL).
- Endpoints: `POST /osha/300a/{site_id}/{year}/compute`, `POST .../certify`, `GET .../export.csv`, `GET .../export.pdf`, `POST .../submit-ita`.

**Testing**:
- `Unit: compute_300a sums entry classifications into correct totals.`
- `Unit: CSV output parses back per RFC 4180; column order matches ITA spec; quoting of fields with commas correct.`
- `Integration (respx-mocked ITA Preview): submit-ita → stores confirmation_id, ita_submitted=true; on poll, status updated.`
- `Integration: submit before certification → 409 problem+json.`

#### 5.3 — Chemical substances, SDS library, GHS classification

**What**: Substance registry, SDS documents with GHS hazard/precautionary mappings, and PDF SDS ingestion.

**Design**:
- `chemical_substance(cas_number unique, ec_number, name, is_svhc, reach_status, rohs_restricted)`.
- `sds_document(substance_id, product_name, manufacturer, revision_date, version, language, file_storage_key, status)` with many-to-many `sds_hazard_mapping`→`ghs_hazard_statement` and `sds_precaution_mapping`→`ghs_precautionary_statement` (seeded in Phase 2).
- SDS PDF upload → `pdfplumber` parser extracts product name, manufacturer, revision date, and detected H/P codes (regex on H###/P### plus section headings) to pre-fill mappings (human confirms).
- Optional ECHA enrichment (`integrations/echa.py`) flags SVHC by CAS against the REACH Candidate List (public API).
- Endpoints: `POST /substances`, `POST /sds` (with PDF), `GET /sds` (search by product/CAS/hazard), `PATCH /sds/{id}` (confirm mappings).

**Testing**:
- `Unit: SDS parser on committed sample PDF fixture → extracts ≥1 H-code and revision_date.`
- `Integration: upload SDS → mappings pre-filled, confirm → persisted; search by H-code returns it.`
- `Integration (respx-mocked ECHA): CAS on candidate list → is_svhc set true.`
- `Unit: duplicate CAS → unique-constraint 409.`

#### 5.4 — Chemical inventory with EPCRA thresholds

**What**: Site/location chemical inventory linked to SDS, with EPCRA Tier II threshold tracking.

**Design**:
- `chemical_inventory(site_id, location_id, sds_id, product_name, quantity, unit, max_quantity, epcra_tpq, last_verified, status)`.
- `services/chemical.py::epcra_exceedances(site)` returns items where `max_quantity >= epcra_tpq` for Tier II reporting.
- Endpoints: `POST /inventory`, `GET /inventory?site=`, `GET /inventory/epcra-exceedances?site=`.

**Testing**:
- `Unit: item with max_quantity above epcra_tpq → appears in exceedances; below → does not.`
- `Integration: inventory item references SDS → GET joins hazard summary from SDS mappings.`

---

## Phase 6: REST API Hardening, OpenAPI/JSON Schema Publication & Offline Field Sync

### Purpose
Turn the endpoints into a stable, published, integration-ready API (the interop artefact `standards.md` calls for) and deliver the offline-first field capture that is the platform's strongest UX differentiator. After this phase, third parties can integrate against a versioned OpenAPI 3.1 spec, and field workers can capture incidents/audits offline and sync reliably.

### Tasks

#### 6.1 — API conventions, pagination, errors, versioning, webhooks

**What**: Consistent pagination, filtering, RFC 9457 errors, `/v1` prefix, idempotency keys, and outbound webhooks.

**Design**:
- All list endpoints: cursor pagination `?limit&cursor` returning `{items, next_cursor}`.
- Errors: `application/problem+json` (RFC 9457) with `type`, `title`, `status`, `detail`, `errors[]`.
- `Idempotency-Key` header honoured on POST (stored, replayed) — critical for offline-retry safety.
- Webhooks: `webhook_subscription(tenant_id, url, event_types[], secret)`; deliveries signed (HMAC-SHA256) and retried with backoff via Celery. Events: `incident.created`, `capa.overdue`, `audit.completed`, `osha.submitted`, etc.

**Testing**:
- `Integration: same Idempotency-Key replayed → single resource created, identical response.`
- `Integration: validation failure → problem+json with errors[] field paths.`
- `Integration (mocked receiver): event fires → signed webhook delivered; bad signature rejected by verifier helper.`
- `Integration: webhook receiver 500 → retried per backoff, marked failed after max attempts.`

#### 6.2 — OpenAPI 3.1 spec + JSON Schema export + typed client

**What**: Generate and publish `schemas/openapi.json` and per-object JSON Schemas; generate the web typed client.

**Design**:
- `make spec` dumps FastAPI's OpenAPI 3.1 doc to `schemas/openapi.json` and exports each Pydantic model's JSON Schema (Draft 2020-12) to `schemas/objects/` (mirroring SafetyCulture's published-schema approach).
- CI check fails if committed spec drifts from generated.
- Web client generated from `openapi.json` into `web/lib/api-client.ts`.

**Testing**:
- `Unit: generated spec validates against OpenAPI 3.1 meta-schema.`
- `CI: committed schemas/openapi.json matches freshly generated (no drift).`
- `Unit: each exported object schema is valid JSON Schema 2020-12.`

#### 6.3 — Offline field sync engine

**What**: Bidirectional sync allowing offline creation/edit of incidents, audits, and responses, with conflict handling.

**Design**:
- Client (PWA) stores pending mutations in IndexedDB; each entity carries client-generated UUID PK and `client_updated_at`.
- `POST /sync/push`: batch of `{entity, op, id, payload, client_updated_at}`; server applies with last-writer-wins per field where safe, flags true conflicts (server `updated_at` > client base) into a `sync_conflict` record for manual resolution. Idempotent via the operation UUID.
- `GET /sync/pull?since=<cursor>`: returns changes since last sync for the user's accessible sites.
- Service worker caches the app shell and read data for the worker's assigned sites.
- Attachments queued and uploaded opportunistically when online.

**Testing**:
- `Unit: push of new offline incident (client UUID) → persisted with same id; replay of same op → no duplicate.`
- `Integration: concurrent edit (client base older than server) → sync_conflict recorded, server value retained, conflict surfaced in pull.`
- `E2E (Playwright, offline mode): create incident offline → go online → appears server-side after sync; attachment uploads.`
- `Integration: pull?since returns only changes for sites the field_worker is scoped to.`

---

## Phase 7: Training & Compliance, Predictive SIF, Regulatory Gap Analysis, Voice

### Purpose
Deliver the v1.1 "should-have" set that deepens compliance value and the AI-native advantage: training/certification with automated recertification reminders, predictive SIF risk scoring from leading indicators, AI regulatory gap analysis with auto-generated CAPAs, and voice-to-text incident capture for the field.

### Tasks

#### 7.1 — Training management with recertification reminders

**What**: Courses, training records with computed expiry, and automated recert reminders.

**Design**:
- `training_course(validity_months, regulatory_ref)`, `training_record(course_id, user_id, completed_date, expiry_date computed, status ∈ {valid,expiring_soon,expired,revoked}, certificate_key)`.
- Beat job nightly recomputes status: `expiring_soon` within configurable window (default 30 days), `expired` past expiry; emits notifications + optional webhook.
- Endpoints: `POST /courses`, `POST /training-records`, `GET /training/matrix?site=` (user × course compliance grid), `GET /training/expiring`.

**Testing**:
- `Unit: completed_date + validity_months → correct expiry_date; record 20 days from expiry → expiring_soon.`
- `Integration: beat reminder job creates notification for expiring records.`
- `Integration: training matrix shows gap (required course missing) for a user.`

#### 7.2 — Predictive SIF risk scoring

**What**: Per-site/location SIF risk score from leading indicators (near-misses, inspection failures, overdue CAPAs, training gaps).

**Design**:
- `ai/sif_model.py`: feature builder aggregates per site over a rolling window — near-miss count, PSIF-flagged count, audit non-conformance rate, overdue-CAPA count, training-gap rate, recent severity trend. Initial model = transparent weighted logistic regression (`scikit-learn`) trained on historical incidents (label = serious/SIF within N days); pluggable to swap models later.
- `sif_score(tenant_id, site_id, location_id, score 0-100, band ∈ {low,elevated,high,critical}, contributing_factors JSONB, computed_at)`.
- Beat job re-scores nightly; results power dashboard leading-indicator cards.
- Cold-start: when insufficient history, fall back to a documented rule-based heuristic and label the score as heuristic.
- Endpoints: `GET /sif/scores?site=`, `GET /sif/scores/{site_id}/factors`.

**Testing**:
- `Unit: feature builder on fixture incidents → expected feature vector.`
- `Unit: site with rising near-misses + overdue CAPAs scores higher than a clean site.`
- `Unit: insufficient-history site → heuristic band returned, flagged heuristic.`
- `Integration: beat scoring job writes sif_score rows for all active sites.`

#### 7.3 — AI regulatory gap analysis with auto-CAPA

**What**: Compare an operation's configured controls/obligations against a regulation's requirements; surface gaps and auto-generate CAPAs.

**Design**:
- Regulation text (e.g., OSHA 1910 subpart, ISO 45001 clauses) stored as `regulation_requirement(framework, ref, requirement_text, embedding vector)`.
- `ai/permit_extract.py` reuse: given an uploaded permit/regulation PDF, extract obligations; `ai/assistant.py` RAG matches obligations against the tenant's existing controls/audit evidence and the LLM classifies each as met/partial/gap with rationale (structured JSON).
- Gaps optionally auto-create CAPAs (`source_type=regulatory`), pending human approval.
- Endpoints: `POST /compliance/gap-analysis` (framework or uploaded doc), `GET /compliance/gaps`, `POST /compliance/gaps/{id}/create-capa`.

**Testing**:
- `Unit (NullProvider): gap analysis returns schema-valid {requirement, status∈{met,partial,gap}, rationale} list.`
- `Integration (mocked LLM): permit PDF → extracted obligations → permit_obligation rows + gaps.`
- `Integration: approve a gap → CAPA created with source_type=regulatory.`

#### 7.4 — Voice-to-text incident capture

**What**: Field workers dictate incident reports; audio is transcribed and pre-fills the incident form.

**Design**:
- `POST /incidents/voice`: upload audio → `ai/transcribe.py` (provider STT or local Whisper) → transcript → root-cause/classification pipeline pre-fills `title`, `description`, suggested `incident_type`. Runs as Celery task; offline-queued on the PWA.
- Returns a draft incident for human review before submission.

**Testing**:
- `Unit (stub STT): audio fixture → transcript → draft incident with populated description.`
- `Integration: voice upload enqueues job; GET draft returns transcript + suggested type.`
- `Integration: corrupt audio → job failed, 200 with status failed.`

---

## Phase 8: Environmental Compliance & ESG Reporting

### Purpose
Extend EHS into the environmental and sustainability domains demanded by CSRD/ESG mandates: environmental permits with obligation tracking, waste manifests, GHG emissions calculation (GHG Protocol), and ESG metric collection mapped to GRI/ESRS frameworks. This captures the fast-growing ESG-disclosure driver identified in research.

### Tasks

#### 8.1 — Environmental permits, obligations, waste manifests

**What**: Permit registry with extracted obligations (reusing AI permit extraction) and waste manifest tracking.

**Design**:
- `environmental_permit(permit_type ∈ {air,water,waste,stormwater,other}, permit_number, issuing_authority, issued/expiry_date, status, conditions_summary, file_storage_key)`.
- `permit_obligation(obligation_text, frequency, next_due_date, assigned_to, status)` — populated manually or via AI extraction from the permit PDF.
- `waste_manifest(manifest_number, waste_type, waste_code (EPA e.g. D001), quantity, unit, generator/transporter/disposal facility, shipped/received_date)`.
- Beat job emits reminders for obligations approaching `next_due_date` and permits nearing `expiry_date`.
- Endpoints: `POST /permits` (+AI extract), `GET /permits/obligations/due`, `POST /waste-manifests`, `GET /waste-manifests`.

**Testing**:
- `Integration: upload permit PDF → AI extraction creates obligations (mocked LLM); due job reminds before next_due_date.`
- `Unit: waste manifest requires waste_code when waste_type=hazardous → 422 if missing.`

#### 8.2 — GHG emissions calculation (GHG Protocol)

**What**: Scope 1/2/3 emissions records computed from activity data × versioned emission factors.

**Design**:
- `emission_source_type(scope 1/2/3, category)`, `emission_factor(fuel_or_activity, factor_value, unit, region, valid_from/to, data_source)` (versioned — calculations use the factor valid at the activity date, per data-model decision 7).
- `emission_record(source_type_id, emission_factor_id, reporting_period, activity_value, activity_unit, co2e_tonnes, data_quality)`; `services` computes `co2e_tonnes = activity_value × factor_value` with unit reconciliation.
- `reports/ghg.py`: Scope 1/2/3 rollups per site/year.
- Endpoints: `POST /emissions`, `GET /emissions/summary?site=&year=` (Scope 1/2/3 totals).

**Testing**:
- `Unit: activity 1000 kWh × factor 0.233 kgCO2e/kWh → 0.233 tCO2e; uses factor valid at activity date.`
- `Unit: activity date outside any factor validity → error naming missing factor.`
- `Integration: summary aggregates records into correct per-scope totals.`

#### 8.3 — ESG metric collection mapped to GRI/ESRS

**What**: Collect ESG metric values and export mapped to GRI and ESRS frameworks.

**Design**:
- `esg_metric_definition(code, framework ∈ {gri,esrs,issb,cdp,tcfd}, framework_ref e.g. "GRI 403-9"/"ESRS S1-14", unit, data_type)` (seeded Phase 2; same data point mappable to multiple frameworks via shared underlying metric).
- `esg_metric_value(metric_id, site_id NULL=org-wide, reporting_year, value_numeric/value_text, data_source, verified)`.
- Auto-population: safety KPIs (TRIR/LTIFR from OSHA data → GRI 403-9) and emissions (→ GRI 305, ESRS E1) derived from existing modules.
- `reports/esg_export.py`: export a framework-scoped report (CSV/JSON) of all mapped metrics for a year.
- Endpoints: `POST /esg/values`, `GET /esg/report?framework=gri&year=`, `POST /esg/auto-populate?year=`.

**Testing**:
- `Unit: TRIR computed from OSHA totals + hours worked → correct rate, stored against GRI 403-9.`
- `Integration: auto-populate pulls emissions into ESRS E1 metric values.`
- `Integration: GRI report export includes only gri-framework metrics for the year.`

---

## Phase 9: AI Differentiators — NL SDS Assistant (RAG), MCP Server, Dashboards

### Purpose
Ship the conversational and analytical layer that distinguishes the platform: a natural-language SDS/compliance assistant (RAG over SDS + regulatory text), the first-in-category MCP server enabling LLM agents to query EHS data, and the manager-facing dashboards that surface leading indicators and compliance status. After this phase the product's AI-native story is complete and externally accessible.

### Tasks

#### 9.1 — RAG ingestion & embeddings

**What**: Chunk and embed SDS documents and regulatory text into pgvector for retrieval.

**Design**:
- On SDS/permit/regulation ingest, a Celery task chunks text, embeds via `AIProvider.embed`, stores `doc_chunk(tenant_id, source_type, source_id, chunk_text, embedding vector, metadata JSONB)` with an `ivfflat`/`hnsw` index.
- Retrieval: cosine similarity top-k filtered by tenant + source scope.

**Testing**:
- `Integration (local embeddings): ingest SDS fixture → chunks embedded; nearest-neighbour query for a known phrase returns its chunk.`
- `Unit: chunker splits long text into bounded, overlapping chunks.`

#### 9.2 — Natural-language SDS & compliance assistant

**What**: Conversational endpoint answering chemical-safety and compliance questions with citations.

**Design**:
- `POST /assistant/query {question, scope}` → RAG retrieves relevant SDS/regulatory chunks → LLM answers with inline citations (chunk source ids), strictly grounded; refuses when no supporting chunk found. Streams response (SSE). Tenant-scoped retrieval only.
- Example: "Which chemicals at Site A are incompatible with bleach?" → retrieves SDS hazard/incompatibility sections, answers with cited products.

**Testing**:
- `Unit (NullProvider): query returns answer + citations referencing retrieved chunks.`
- `Integration: question with no relevant chunks → assistant returns "no supporting data" rather than hallucinating.`
- `Integration: retrieval never returns another tenant's chunks (isolation test).`

#### 9.3 — MCP server

**What**: An MCP server exposing read-only EHS tools for LLM agents (first in category).

**Design**:
- `mcp/server.py` (official Python MCP SDK) exposing tools: `list_incidents(site, status, date_range)`, `get_sif_scores(site)`, `search_sds(query)`, `get_risk_register(site)`, `get_compliance_status(framework)`, `get_capa_status(filters)`.
- Auth via a scoped API token mapped to a user/tenant; all tool calls run through the same services + RLS, so MCP cannot bypass tenant isolation or permissions.
- Read-only by default; no mutation tools in v1.

**Testing**:
- `Integration: MCP list_incidents with tenant token → returns only that tenant's incidents respecting permissions.`
- `Integration: MCP token lacking incident.read → tool returns authorization error.`
- `Unit: tool schemas advertised match underlying service signatures.`

#### 9.4 — Dashboards & analytics UI

**What**: Role-based dashboards: leading indicators (SIF bands, near-miss trend), CAPA aging, audit scores, OSHA TRIR/LTIFR, training compliance, ESG snapshot.

**Design**:
- Server-rendered Next.js dashboard pages consuming aggregation endpoints (`GET /analytics/*`) with KPI cards configurable per role.
- Endpoints: `/analytics/leading-indicators`, `/analytics/capa-aging`, `/analytics/incident-trends`, `/analytics/training-compliance`.

**Testing**:
- `Integration: analytics endpoints return correct aggregates over fixture data.`
- `E2E (Playwright): ehs_manager sees SIF + CAPA aging cards; field_worker sees a restricted view.`

---

## Phase 10: Backlog Differentiators — Computer Vision PPE, Contractor Management, Multi-Language

### Purpose
Implement the remaining "nice-to-have" advanced features that complete competitive parity with premium suites while remaining open: computer-vision PPE detection from images, contractor prequalification workflows, and multi-language UI/content support. These are additive and can be deferred without blocking earlier value.

### Tasks

#### 10.1 — Computer-vision PPE compliance detection

**What**: Detect missing PPE (hard hat, hi-vis, gloves, eye protection) in uploaded site images/frames.

**Design**:
- `ai/ppe_vision.py` wraps a YOLO model (`ultralytics`) with a configurable PPE-class model; inference runs as a Celery task (GPU optional, CPU fallback).
- `ppe_detection(tenant_id, site_id, image_key, detections JSONB [{class, bbox, confidence, compliant}], violation_count, reviewed)`.
- `POST /ppe/analyze` (image) → enqueues; `GET /ppe/detections?site=`. High-violation results can spawn CAPAs.
- Model weights not committed; documented download/config step. Feature is opt-in (heavy dependency).

**Testing**:
- `Unit (stub detector): image fixture → deterministic detections → violation_count computed.`
- `Integration: analyze enqueues job; GET returns detections; violation can create CAPA.`

#### 10.2 — Contractor management & prequalification

**What**: Contractor registry with prequalification workflow and document/insurance expiry tracking.

**Design**:
- `contractor(name, status ∈ {pending,prequalified,suspended,expired})`, `contractor_document(type, file_key, expiry_date)`, `prequalification(checklist responses, score, decision, decided_by)`.
- Beat job flags expiring insurance/certs; suspends contractors with expired mandatory docs.
- Endpoints: `POST /contractors`, `POST /contractors/{id}/prequalify`, `GET /contractors/expiring`.

**Testing**:
- `Integration: complete prequalification meeting threshold → status prequalified.`
- `Integration: expired mandatory insurance → beat job sets status expired.`

#### 10.3 — Multi-language support

**What**: Localised UI and translatable content (incident types, course names, templates).

**Design**:
- Web: i18n with locale resource bundles; server returns localised reference-data labels via a `translation(entity, record_id, locale, field, value)` table.
- `Accept-Language` honoured; user locale preference stored on `app_user`.
- Initial locales: en, es, fr, de, zh.

**Testing**:
- `Integration: request with Accept-Language: es → reference labels returned in Spanish where translations exist, fallback to en otherwise.`
- `E2E: switch UI locale → strings update.`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (skeleton, multi-tenancy, auth, RBAC)   ─── required by everything
    │
Phase 2: Core Domain (sites/locations, reference data,       ─── requires P1
         hybrid+JSONB, audit trail)
    │
Phase 3: Incidents + AI Root-Cause (core value)             ─── requires P2
    │
    ├── Phase 4: CAPA / Audits / Risk                        ─── requires P3
    │       │
    │       └── Phase 5: OSHA Recordkeeping + SDS/Chemical   ─── requires P4 (CAPA), P3 (incidents)
    │
    └── Phase 6: API hardening + OpenAPI + Offline Sync      ─── requires P3,P4 (entities to expose/sync)
            │
            ├── Phase 7: Training / SIF / Reg-Gap / Voice    ─── requires P4,P5 (indicators, OSHA data)
            │
            ├── Phase 8: Environmental + ESG                 ─── requires P2,P5 (reference data, OSHA→ESG)
            │
            └── Phase 9: NL Assistant (RAG) + MCP + Dashboards ── requires P5 (SDS), P7 (SIF) for full value
                    │
                    └── Phase 10: CV PPE / Contractors / i18n ── requires P4 (CAPA), additive
```

Parallelism after their dependencies are met:
- **Phase 4 and Phase 6** can proceed concurrently once Phase 3 is done (6.1/6.2 can begin immediately; 6.3 sync needs Phase 4 entities).
- **Phases 7, 8, and 9** can be developed concurrently after Phase 5 (they touch different modules; 9's full dashboards benefit from 7's SIF).
- **Phase 10's three tasks** (CV PPE, contractors, i18n) are mutually independent and can be split across developers.

Estimated scope: **large** (full-stack multi-tenant platform with AI, offline sync, regulatory exports, and integrations).

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass; new behaviour has both happy-path and edge-case tests.
3. `ruff` + `black --check` pass; `mypy --strict` passes (Python); `eslint` + `tsc` pass (web touched in the phase).
4. `docker compose build` succeeds and the affected services start cleanly.
5. The phase's headline capability works end-to-end (verified by an integration or E2E test).
6. New config options are documented in `.env.example` and README.
7. New/changed API endpoints appear in the regenerated `schemas/openapi.json` (and exported JSON Schemas), with the no-drift CI check passing.
8. Alembic migration(s) created, reversible, and applied cleanly on a fresh database; `make seed` remains idempotent.
9. New entities carry `tenant_id` + RLS (where tenant-scoped) and are covered by an isolation test.
10. New AI features run through the `AIProvider` gateway, record an `ai_job`, and pass with `NullProvider` in CI (no live API calls required to test).
```
