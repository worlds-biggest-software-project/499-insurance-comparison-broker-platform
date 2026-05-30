# Insurance Comparison & Broker Platform — Phased Development Plan

> Project: 499-insurance-comparison-broker-platform · Created: 2026-05-31
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and `data-model-suggestion-3.md` (the Hybrid Relational + JSONB model, selected below). It defines an open-source, AI-native platform that lets independent agents and brokers quote, compare, and bind policies across multiple carriers from a single interface, combined with modern agency management.

---

## Core Requirements (Synthesis)

- **What it does**: Single-entry comparative quoting across multiple carriers for personal and small commercial lines, with a full quote-to-bind digital workflow, agency management (clients, policies, documents, commissions, renewals), white-label consumer distribution, and AI automation (document extraction, appetite pre-screening, coverage gap detection).
- **Who uses it**: Independent agents/producers, CSRs, agency principals; MGAs building distribution; insurtechs embedding insurance; end consumers via white-label portals.
- **Key differentiators**: The only open-source alternative in a market of six-figure proprietary AMS/raters; AI as foundational infrastructure (not bolt-on); API-first/headless; transparent carrier appetite data; affordable entry point below the $300/mo incumbents.
- **MVP**: Multi-carrier single-entry quoting (personal + small commercial), ACORD-compliant data model, 10–15 carrier integrations, client/policy management with documents, side-by-side comparison, RBAC.
- **Post-MVP**: AI document extraction, appetite pre-screening, white-label portal, commission accounting, renewal management. Backlog: embedded insurance API, coverage gap analysis, predictive renewal retention, no-code workflows, cross-line bundling.
- **Deployment model**: Self-hosted and cloud; modular (quoting / AMS / distribution adoptable independently); API-first headless core with a separate web UI.
- **Integration surface**: Carrier REST/SOAP/AL3 APIs, LLM providers (document extraction, conversational quoting), payment gateway (PCI descoping via tokenisation), webhooks for CRM/AMS sync, optional IVANS connectivity.
- **Standards compliance**: ACORD (XML/AL3/NGDS data model + forms), SEMCI single-entry model, NAICS classification, OpenAPI 3.1, JSON Schema, AsyncAPI (webhooks), OAuth 2.0 / OIDC / JWT, PCI DSS v4.0, TLS 1.2/1.3, GLBA/CCPA/GDPR, NAIC Model Law #668, SOC 2 Type II posture.
- **Data model**: Hybrid Relational + JSONB on PostgreSQL (data-model-suggestion-3) — relational bones for agency/client/policy/commission integrity, JSONB for line-of-business-polymorphic risk details, carrier API config, quote coverages, and AI extraction output.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12 | The product is AI-heavy (document extraction, NLP coverage-gap analysis, appetite ML, conversational quoting). Python has the strongest LLM/ML ecosystem and excellent async support for fanning out to carriers. |
| API framework | FastAPI | Native async (critical for concurrent multi-carrier quoting), Pydantic v2 validation aligns with the JSONB-heavy data model, and it auto-generates the OpenAPI 3.1 spec required by `standards.md`. |
| ASGI server | Uvicorn (behind Gunicorn workers) | Standard production ASGI stack for FastAPI. |
| Database | PostgreSQL 16 | Required by the chosen hybrid model: relational integrity for accounting/audit plus JSONB + GIN indexes for polymorphic risk/quote/extraction data. Partitioning for audit/quote growth. |
| ORM / DB toolkit | SQLAlchemy 2.0 (async) + Alembic | Async ORM matching FastAPI; Alembic for versioned migrations as ACORD elements evolve. JSONB columns mapped via SQLAlchemy `JSONB` type with Pydantic validation at the boundary. |
| Schema validation | Pydantic v2 + JSON Schema | Validates the JSONB payloads (risk details, coverages, appetite criteria) that the DB treats as opaque. JSON Schema documents per-line risk shapes. |
| Task queue | Celery + Redis | Carrier quoting, AI extraction, renewal scans, and webhook delivery are async/long-running. Celery handles retries, timeouts, and scheduled (beat) jobs. |
| Cache / broker | Redis 7 | Celery broker, rate-limit counters, appetite-rule cache, idempotency keys. |
| LLM access | Provider-agnostic abstraction (LiteLLM-style gateway) over OpenAI/Anthropic | Document extraction and conversational quoting must not lock to one vendor; a gateway lets self-hosters supply their own keys/models. |
| Object storage | S3-compatible (MinIO for self-host, AWS S3 for cloud) via `boto3` | Document storage (applications, loss runs, dec pages) outside the DB. |
| Auth | OAuth 2.0 + OIDC, JWT access tokens (RFC 6749/7519, OIDC Core 1.0) | Required by `standards.md`; supports agent SSO, API clients, and consumer portal sessions. `authlib` + `python-jose`. |
| Payments | Stripe (tokenised) | PCI DSS v4.0 descoping — cardholder data never touches the platform; only tokens stored. |
| Frontend | Next.js 15 (React, TypeScript) + Tailwind + shadcn/ui | Agent console (quoting, comparison grid, AMS dashboards) and white-label consumer portal. Server components for fast dashboards; SPA-like quoting flows. Consumes the headless API. |
| Embeddable widget | Web Components (Lit) | Embedded-insurance distribution for non-insurance sites, mirroring Bold Penguin's framework-agnostic SDK approach. |
| Webhooks spec | AsyncAPI 2.x | Documents event-driven quote/bind/policy notifications per `standards.md`. |
| Containerisation | Docker + docker-compose | Self-hosted deployment is a core value prop; compose wires api, worker, beat, postgres, redis, minio, web. |
| Testing | pytest, pytest-asyncio, httpx, testcontainers, respx; Playwright for web E2E | Async-aware unit/integration testing; testcontainers spins real Postgres/Redis; respx mocks carrier/LLM HTTP. |
| Code quality | Ruff (lint+format), mypy (strict), pre-commit | Single fast linter/formatter; strict typing given the JSONB/Pydantic boundary risk. |
| Package manager | uv (with pyproject.toml) | Fast, reproducible Python dependency management. |
| Key libraries | `httpx` (async carrier calls), `tenacity` (retry/backoff), `pypdf`/`pdfplumber` (PDF parsing), `lxml` (ACORD XML), `cryptography` (credential vault), `structlog` (audit-grade logging) | Domain-specific needs: ACORD XML generation, resilient carrier calls, document parsing pre-LLM. |

### Project Structure

```
insurance-platform/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── .env.example
├── README.md
├── openapi/                         # generated OpenAPI 3.1 + AsyncAPI specs (CI artefact)
├── migrations/                      # Alembic migration scripts
│   └── versions/
├── src/
│   └── insurance_platform/
│       ├── __init__.py
│       ├── main.py                  # FastAPI app factory, router mounting
│       ├── config.py                # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── session.py           # async engine/session
│       │   ├── base.py              # declarative base, mixins (timestamps, agency_id)
│       │   └── models/              # SQLAlchemy models (agency, client, risk, quote, policy, …)
│       ├── schemas/                 # Pydantic request/response + JSONB payload schemas
│       │   ├── risk/                # per-line risk detail schemas (auto, homeowner, wc, …)
│       │   ├── coverage.py
│       │   └── ...
│       ├── api/
│       │   ├── deps.py              # auth, tenancy, RBAC dependencies
│       │   ├── v1/
│       │   │   ├── auth.py
│       │   │   ├── clients.py
│       │   │   ├── risks.py
│       │   │   ├── carriers.py
│       │   │   ├── quotes.py
│       │   │   ├── policies.py
│       │   │   ├── commissions.py
│       │   │   ├── documents.py
│       │   │   ├── renewals.py
│       │   │   ├── portal.py        # public white-label endpoints
│       │   │   └── webhooks.py
│       ├── services/                # business logic (no HTTP/DB framework leakage)
│       │   ├── quoting/             # orchestration of multi-carrier quoting
│       │   ├── appetite/            # pre-screening engine
│       │   ├── binding.py
│       │   ├── commission.py
│       │   ├── renewal.py
│       │   └── acord/               # ACORD form mapping + XML/AL3 export
│       ├── carriers/                # carrier integration layer
│       │   ├── base.py              # CarrierAdapter protocol
│       │   ├── registry.py
│       │   ├── mock_carrier.py
│       │   └── adapters/            # one module per real carrier
│       ├── ai/
│       │   ├── llm.py               # provider-agnostic LLM client
│       │   ├── extraction.py        # document → fields
│       │   ├── gap_analysis.py
│       │   └── prompts/             # versioned prompt templates
│       ├── auth/                    # oauth/jwt/rbac
│       ├── storage/                 # S3 abstraction
│       ├── payments/                # Stripe tokenisation
│       ├── webhooks/                # outbound event delivery
│       ├── workers/                 # Celery app, tasks, beat schedules
│       └── audit/                   # audit log writer + middleware
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   └── fixtures/                    # sample ACORD docs, carrier responses, JSON payloads
└── web/                             # Next.js agent console + portal (separate workspace)
    ├── package.json
    ├── app/
    ├── components/
    └── lib/api-client/              # generated from OpenAPI spec
```

---

## Phase 1: Foundation & Tenancy

### Purpose
Establish the project skeleton, configuration, database connectivity, multi-tenant data isolation, and the core agency/user entities. After this phase the application boots, connects to Postgres/Redis, runs migrations, and enforces per-agency data scoping — the bedrock every later feature relies on.

### Tasks

#### 1.1 — Project scaffolding & tooling
**What**: Initialise the repo with dependency management, linting, typing, Docker, and a bootable FastAPI app.

**Design**:
- `pyproject.toml` with `uv`; deps: fastapi, uvicorn[standard], sqlalchemy[asyncio], asyncpg, alembic, pydantic, pydantic-settings, redis, celery, httpx, tenacity, structlog, authlib, python-jose[cryptography], boto3, stripe.
- `config.py` using `pydantic_settings.BaseSettings`:
  ```python
  class Settings(BaseSettings):
      database_url: str
      redis_url: str = "redis://localhost:6379/0"
      jwt_secret: str
      jwt_algorithm: str = "RS256"
      access_token_ttl_seconds: int = 900
      s3_endpoint: str | None = None
      s3_bucket: str = "documents"
      llm_provider: str = "openai"
      llm_api_key: str | None = None
      environment: Literal["dev", "test", "prod"] = "dev"
      model_config = SettingsConfigDict(env_file=".env")
  ```
- `main.py` app factory mounting `/api/v1`, `/health`, and exposing `/openapi.json` (OpenAPI 3.1).
- `docker-compose.yml`: services api, worker, beat, postgres:16, redis:7, minio.
- Ruff + mypy(strict) config; pre-commit hooks.

**Testing**:
- `Unit: Settings loads from env → fields populated with correct types and defaults`.
- `Unit: missing required DATABASE_URL → ValidationError naming the field`.
- `Integration: GET /health → 200 {"status":"ok","db":"up","redis":"up"}`.
- `E2E: docker compose up → all containers healthy, /health reachable`.

#### 1.2 — Database base, mixins & migrations
**What**: SQLAlchemy declarative base, common mixins, and Alembic wired for async.

**Design**:
- `TimestampMixin` (`created_at`, `updated_at` server-default `now()`), `UUIDPKMixin`.
- `AgencyScopedMixin` adding `agency_id: Mapped[UUID]` FK — the tenancy key on every tenant-owned row.
- Alembic configured against the async engine; `alembic revision --autogenerate` workflow documented.

**Testing**:
- `Integration (testcontainers Postgres): run migrations head → expected tables exist`.
- `Unit: model instance defaults → created_at/updated_at auto-populated on flush`.

#### 1.3 — Agency & user entities (from data-model-suggestion-3)
**What**: `agency` and `agency_user` tables with JSONB contact/settings/license fields.

**Design** (DDL from suggestion 3 §"Core Agency & User Entities"):
```sql
CREATE TABLE agency ( agency_id UUID PK, name, legal_name, tax_id, license_number,
  license_state CHAR(2), contact JSONB, settings JSONB, created_at, updated_at );
CREATE TABLE agency_user ( user_id UUID PK, agency_id FK, email UNIQUE, password_hash,
  first_name, last_name, role CHECK (principal|producer|csr|admin),
  license_info JSONB, preferences JSONB, is_active, last_login_at, created_at, updated_at );
```
- Pydantic schemas validate `contact` (`address{line1,line2,city,state,zip}`, phone, email, website) and `license_info` (`number,state,expiration,lines[]`).
- Password hashing via `argon2`.

**Testing**:
- `Unit: AgencyContact schema rejects malformed zip / invalid state code`.
- `Integration: create agency + user → rows persisted, password stored as argon2 hash (never plaintext)`.
- `Integration: duplicate user email → IntegrityError surfaced as 409`.

#### 1.4 — Tenancy enforcement dependency
**What**: A FastAPI dependency that scopes every query to the caller's `agency_id`.

**Design**:
- `get_current_agency()` extracts `agency_id` from the validated JWT claims.
- A query helper `agency_scoped(stmt, agency_id)` appends `WHERE agency_id = :id`; service layer never queries tenant tables without it.
- Cross-tenant access attempts return 404 (not 403) to avoid resource enumeration.

**Testing**:
- `Integration: user A requests client owned by agency B → 404`.
- `Unit: agency_scoped() adds correct WHERE clause to a select`.

---

## Phase 2: Auth, RBAC & Audit

### Purpose
Implement authentication, role-based authorisation for producers/CSRs/principals/admins, and the audit logging required by NAIC Model Law #668 and SOC 2. After this phase, endpoints are protected and every mutating action is recorded.

### Tasks

#### 2.1 — OAuth2/JWT authentication
**What**: Password and refresh-token flows issuing JWTs.

**Design** (RFC 6749/7519, OIDC Core 1.0):
- `POST /api/v1/auth/token` (password grant) → `{access_token, refresh_token, expires_in, token_type:"Bearer"}`.
- `POST /api/v1/auth/refresh`.
- Access tokens RS256-signed JWTs; claims `{sub:user_id, agency_id, role, scopes[], exp, iat}`.
- Refresh tokens stored hashed in Redis with rotation + revocation list.

**Testing**:
- `Integration: valid credentials → 200 with well-formed JWT (claims verified)`.
- `Integration: wrong password → 401, generic message, no user enumeration`.
- `Integration: expired access token → 401; refresh → new pair, old refresh revoked`.

#### 2.2 — RBAC
**What**: Role/scope enforcement per endpoint.

**Design**:
- Scope matrix, e.g. `principal`: all; `producer`: quote/bind/own-clients; `csr`: service/no-bind; `admin`: agency config/users.
- `require_scopes("policy:bind")` dependency raising 403 on mismatch.
- Producers restricted to clients assigned to them (`assigned_producer_id`) unless principal/admin.

**Testing**:
- `Unit: scope matrix resolves expected scopes per role`.
- `Integration: csr calls bind endpoint → 403`.
- `Integration: producer accesses another producer's client → 404`.

#### 2.3 — Audit log
**What**: Append-only, partitioned audit trail.

**Design** (DDL from suggestion 3 §"Audit Log", `PARTITION BY RANGE (created_at)`):
- Columns: `log_id BIGSERIAL`, `agency_id`, `user_id`, `entity_type`, `entity_id`, `action CHECK(create|update|delete|view|export|bind)`, `changes JSONB`, `request_context JSONB`, `created_at`. GIN index on `changes`.
- Middleware/service hook captures `{field:{old,new}}` diffs on mutations and `{ip,user_agent,session_id,correlation_id}`.
- Monthly partition auto-creation via a Celery beat maintenance job.

**Testing**:
- `Integration: update client name → audit row with action=update and changed field diff`.
- `Integration: bind policy → audit row action=bind`.
- `Unit: partition-name helper returns audit_log_2026_06 for a June timestamp`.

---

## Phase 3: Domain Core — Clients, Risks & Carriers

### Purpose
Model the central insurance entities with the hybrid relational+JSONB approach, including the polymorphic risk model and per-line JSON Schemas. This is the data heart of the platform; quoting (Phase 4) depends entirely on it.

### Tasks

#### 3.1 — Client entity & CRUD
**What**: `client` table and CRUD API.

**Design** (DDL from suggestion 3 §"Client Entity"): relational `display_name,email,phone,state,zip_code,naics_code,assigned_producer_id,assigned_csr_id`; JSONB `profile` (individual vs business shapes), `source_info`; `tags TEXT[]`; GIN index on `profile`. Endpoints:
- `POST/GET/PATCH /api/v1/clients`, `GET /api/v1/clients/{id}`, `GET /api/v1/clients?q=&state=&tag=`.
- Pydantic discriminated union on `client_type` validates `profile`.

**Testing**:
- `Unit: business profile missing tax_id → ValidationError`.
- `Integration: create individual client → display_name computed "Last, First"`.
- `Integration: search by tag → GIN-indexed query returns matches`.

#### 3.2 — Polymorphic risk model & per-line schemas
**What**: `risk` table plus JSON Schemas for each `risk_type`.

**Design** (DDL from suggestion 3 §"Polymorphic Risk Model"): relational `client_id,risk_type,state,zip_code`; JSONB `details`, `acord_mappings`; GIN on `details`. Pydantic models per line:
```python
class AutoRiskDetails(BaseModel):
    year:int; make:str; model:str; vin:str
    usage:Literal["commute","pleasure","business"]; annual_mileage:int
    drivers:list[Driver]; garaging_address:Address
class HomeownerRiskDetails(BaseModel):
    year_built:int; construction_type:str; square_footage:int; stories:int
    roof_type:str; roof_year:int; protection_class:int; replacement_cost:Decimal
    pool:bool=False; trampoline:bool=False
class WorkersCompRiskDetails(BaseModel):
    class_codes:list[ClassCodePayroll]; experience_mod:Decimal
    governing_state:str; prior_carrier:str|None
RISK_SCHEMA_REGISTRY: dict[str, type[BaseModel]] = {"auto":AutoRiskDetails, ...}
```
- On write, `details` validated against the registered schema for `risk_type`; rejected if unknown line.

**Testing**:
- `Unit: auto details with non-numeric year → ValidationError`.
- `Unit: unknown risk_type → 422 listing supported lines`.
- `Integration: query commercial_property where details->>square_footage > 50000 in CA` (suggestion-3 query pattern).

#### 3.3 — Carrier & appetite catalog
**What**: `carrier` and `carrier_appetite` tables with JSONB config/criteria.

**Design** (DDL from suggestion 3 §"Carrier & Appetite"): `carrier.api_config` JSONB (`endpoint,type:rest|soap|al3|manual,auth{method,credentials_vault_ref},rate_limits,field_mappings,supported_lines[],response_format`); `carrier_appetite.criteria` JSONB (`naics_codes[],naics_exclusions[],min/max_premium,min_years_in_business,prohibited_operations[],territory_restrictions,class_codes,notes`); GIN on `criteria`.
- Credential refs point to the encrypted vault (Phase 6 storage), never plaintext keys.
- Admin endpoints `POST/PATCH /api/v1/carriers`, `POST /api/v1/carriers/{id}/appetite`.

**Testing**:
- `Unit: appetite criteria schema validates naics_codes as 6-digit strings`.
- `Integration: appetite query criteria @> '{"naics_codes":["722511"]}'` returns matching carriers (suggestion-3 query).
- `Integration: carrier api_config never returns raw credentials in responses`.

---

## Phase 4: Carrier Integration & Comparative Quoting (Core Value)

### Purpose
The heart of the product: submit one risk and receive bindable quotes from many carriers, then compare them side by side. Implements the SEMCI single-entry model with a pluggable carrier adapter layer and async fan-out. After this phase the platform delivers its primary value proposition.

### Tasks

#### 4.1 — Carrier adapter abstraction
**What**: A uniform interface every carrier integration implements.

**Design**:
```python
class CarrierQuoteResult(BaseModel):
    status: Literal["returned","declined","referred","error"]
    quote_number: str | None
    total_premium: Decimal | None
    annual_premium: Decimal | None
    term_months: int = 12
    coverages: list[CoverageLine] = []
    rating_factors: dict[str, Any] = {}
    declination: dict | None = None
    raw_response: dict | None = None      # stored in quote.carrier_response_raw

class CarrierAdapter(Protocol):
    carrier_id: UUID
    supported_lines: set[str]
    async def quote(self, submission: CanonicalSubmission) -> CarrierQuoteResult: ...
    async def bind(self, quote_ref: BindRequest) -> BindResult: ...
```
- `CanonicalSubmission` is the SEMCI single-entry payload (client + risks + coverages-requested) mapped per-carrier via `api_config.field_mappings`.
- `registry.py` resolves adapters by `carrier_id`; `mock_carrier.py` returns deterministic fixtures for testing/demo.
- Calls wrapped in `tenacity` retry with per-carrier `rate_limits` and a hard timeout.

**Testing**:
- `Unit: field_mappings translate canonical → carrier-specific payload`.
- `Integration (respx mock): adapter handles HTTP 500 with retry then error result`.
- `Integration: mock carrier returns deterministic quote for sample submission`.

#### 4.2 — Quote request orchestration (single entry, multi-company)
**What**: `quote_request` + `quote` tables and the async fan-out service.

**Design** (DDL from suggestion 3 §"Quote Request & Carrier Quotes"): `quote_request` with `risk_ids UUID[]`, `status(draft|submitted|quoting|completed|expired|cancelled)`, `appetite_results JSONB`; `quote` with relational `status,total_premium,annual_premium,quote_number,quoted_at,expires_at` plus JSONB `coverages,rating_factors,declination,carrier_response_raw,coverage_gaps`.
- `POST /api/v1/quote-requests` → creates request, enqueues a Celery group: one `quote_carrier` task per eligible carrier.
- Each task calls the adapter, writes a `quote` row, updates status. Request transitions to `completed` when all tasks settle or a global timeout (default 90s) elapses.
- `GET /api/v1/quote-requests/{id}` streams partial results (carriers may respond at different times).

**State machine**: `draft → submitted → quoting → completed|expired|cancelled`; per-quote `pending → returned|declined|referred|error → bound|expired`.

**Testing**:
- `Integration: submit request with 3 mock carriers → 3 quote rows, request status=completed`.
- `Integration: one carrier times out → that quote=error, others returned, request still completes`.
- `Unit: state machine rejects illegal transition (completed → quoting)`.

#### 4.3 — Side-by-side comparison endpoint
**What**: Normalised comparison view across carriers.

**Design**: `GET /api/v1/quote-requests/{id}/comparison` returns carriers ordered by `annual_premium`, each with normalised coverage lines using standardised coverage codes (so "BI" maps consistently across carriers). Implements suggestion-3's comparison query joining `quote`+`carrier`.

**Testing**:
- `Integration: comparison sorted ascending by annual_premium; declined carriers grouped separately`.
- `Unit: coverage normaliser maps two carriers' differing codes to the same canonical code`.

#### 4.4 — Ship 10–15 carrier adapters (MVP target)
**What**: Concrete adapters meeting the MVP carrier-count goal.

**Design**: Implement adapters for available REST carrier sandboxes; for carriers without open APIs, provide AL3/manual stub adapters. Each adapter has its own `api_config` field mapping and a recorded fixture set. Document an "add a carrier" guide.

**Testing**:
- `Integration (respx, recorded fixtures): each adapter maps a sample carrier response to CarrierQuoteResult`.
- `Fixture-based: golden CanonicalSubmission → expected per-carrier payload snapshot`.

---

## Phase 5: Quote-to-Bind, Policies & Documents

### Purpose
Complete the digital workflow from selected quote to bound policy, with policy lifecycle management and document storage — the AMS core plus the bind step that differentiates digital-native platforms.

### Tasks

#### 5.1 — Binding workflow
**What**: Select a quote and bind it into a policy.

**Design**:
- `POST /api/v1/quotes/{id}/bind` (scope `policy:bind`): validates quote not expired, carrier confirms bindability, then calls `adapter.bind()`. On success creates a `policy` row, snapshots `coverages` from the quote, generates a binder document. Payment (Stripe token) collected pre-bind where required.
- Idempotency-Key header prevents double-bind.

**Testing**:
- `Integration: bind returned quote → policy created, coverages snapshotted, quote status=bound`.
- `Integration: bind expired quote → 409`.
- `Integration: duplicate Idempotency-Key → single policy`.

#### 5.2 — Policy entity & lifecycle
**What**: `policy` table with JSONB endorsements/termination/billing.

**Design** (DDL from suggestion 3 §"Policy"): relational `policy_number,line_of_business,status,effective_date,expiration_date,total_premium,prior_policy_id`; JSONB `coverages` (append-only `endorsements`, `termination`, `billing`). Endpoints for endorse, cancel, reinstate, non-renew; each appends to `endorsements`/sets `termination` and writes audit + outbound webhook event.

**Testing**:
- `Integration: apply endorsement → appended to endorsements[] with premium_change; total_premium updated`.
- `Integration: cancel policy → status=cancelled, termination populated with return_premium`.

#### 5.3 — Document storage & ACORD export
**What**: `document` table, S3 upload, and ACORD form/XML generation.

**Design** (DDL from suggestion 3 §"Documents", `extraction` JSONB reserved for Phase 7): pre-signed S3 upload URLs; virus-scan hook; `document_type(application|loss_run|certificate|dec_page|endorsement|audit|correspondence|acord_form|invoice|other)`. `services/acord/` maps client+risk+policy data to ACORD form fields and emits ACORD XML (lxml) for carrier interchange and PDF ACORD forms (e.g., ACORD 125/126/140).

**Testing**:
- `Integration: request upload URL, PUT file, register document → metadata row, file in MinIO`.
- `Unit: ACORD XML output validates against the ACORD XSD for the target form`.
- `Fixture-based: policy → ACORD 125 PDF with correct field placement`.

---

## Phase 6: Credential Vault, Payments & Webhooks

### Purpose
Production-readiness infrastructure: securely store carrier/API credentials, process premium payments PCI-compliantly, and emit event-driven webhooks for CRM/AMS integration. Enables real carrier connectivity and embedded distribution.

### Tasks

#### 6.1 — Encrypted credential vault
**What**: Encrypted-at-rest store for carrier API credentials referenced by `api_config.credentials_vault_ref`.

**Design**: Envelope encryption (`cryptography` Fernet/AES-GCM) with a master key from env/KMS; optional pluggable HashiCorp Vault backend. Decryption only in the carrier worker process; credentials never logged or returned via API.

**Testing**:
- `Unit: encrypt→decrypt round-trips; ciphertext differs each call (nonce)`.
- `Integration: vault_ref resolves to credentials only inside worker context; API serialiser redacts`.

#### 6.2 — Payment processing (PCI descoped)
**What**: Stripe tokenised premium collection.

**Design** (PCI DSS v4.0): frontend tokenises card via Stripe Elements; backend receives only the token, creates a PaymentIntent, stores `payment_method_token` + status — never PAN/CVV. TLS 1.2/1.3 enforced.

**Testing**:
- `Integration (Stripe test mode): token → successful PaymentIntent recorded against policy`.
- `Unit: assert no card-number-shaped field ever persisted (regex guard test)`.

#### 6.3 — Outbound webhooks (AsyncAPI)
**What**: Event delivery for quote/bind/policy/commission events.

**Design**: `webhook_subscription` table (`agency_id,url,events[],secret`). Events `quote.completed`, `policy.bound`, `policy.cancelled`, `commission.reconciled`. Celery delivery with HMAC-SHA256 signature header, exponential backoff, dead-letter after N attempts. AsyncAPI spec generated for the catalog.

**Testing**:
- `Integration: policy.bound fires → subscriber receives signed payload; bad endpoint → retried then dead-lettered`.
- `Unit: HMAC signature verifies with the subscription secret`.

---

## Phase 7: AI — Document Extraction, Appetite Pre-Screening & Coverage Gaps

### Purpose
Deliver the AI-native differentiators: extract structured data from documents into ACORD fields, pre-screen risks against carrier appetite before submission, and flag coverage gaps. This is the project's strategic moat.

### Tasks

#### 7.1 — Provider-agnostic LLM client
**What**: Single interface over multiple LLM providers.

**Design**: `ai/llm.py` exposes `complete(messages, schema)` and `extract(document_text, schema)` returning validated JSON (structured output / function-calling). Provider chosen by `settings.llm_provider`; self-hosters supply keys. Token usage logged for cost tracking. Versioned prompts in `ai/prompts/`.

**Testing**:
- `Unit (mocked provider): extract returns schema-valid JSON; invalid JSON → repaired or raises`.
- `Unit: provider switch (openai↔anthropic) preserves the same interface`.

#### 7.2 — Document extraction → ACORD field population
**What**: Turn uploaded applications/loss runs/dec pages into structured fields.

**Design**: Pipeline: S3 fetch → text/layout extract (`pdfplumber`) → LLM `extract` with target schema (risk/ACORD fields) → write to `document.extraction` JSONB (`status,model_version,overall_confidence,fields[{name,value,confidence,acord_ref,page,bbox}]`). Low-confidence fields flagged for human review (`status:reviewed` after `corrections[]`). Runs as a Celery task triggered on upload.

**Prompt template (structure)**: system role = "insurance document extraction specialist"; user supplies document text + JSON Schema of target ACORD fields; model returns field/value/confidence/acord_ref.

**Testing**:
- `Fixture-based: sample ACORD 125 PDF → extracted business_name/NAICS/address with confidence`.
- `Integration: low-confidence field → status awaiting review; correction recorded in corrections[]`.
- `Unit: extraction output validates against the risk schema before persisting`.

#### 7.3 — Appetite pre-screening engine
**What**: Rank/filter carriers before submission to cut declinations.

**Design**: `services/appetite/` evaluates each `carrier_appetite.criteria` against a risk (NAICS in/exclusions, premium bounds, years-in-business, prohibited ops, territory). Produces `appetite_results` JSONB (`eligible[{carrier_id,score,reasons}], ineligible[{carrier_id,reasons}], model_version`). Phase 4 fan-out submits only to eligible carriers. Deterministic rules first; optional ML scoring layer.

**Testing**:
- `Unit: restaurant NAICS 722511 in TX matches carrier whose criteria include it; excluded NAICS filtered out`.
- `Integration: quote request runs pre-screen → only eligible carriers receive submissions`.

#### 7.4 — Coverage gap detection
**What**: NLP analysis flagging gaps/exclusions vs. risk profile and benchmarks.

**Design**: For returned quotes, LLM compares coverages present against a benchmark set for the line/NAICS; writes `quote.coverage_gaps` JSONB (`[{gap_type,description,severity,benchmark,model_version}]`). Surfaced in the comparison view (Phase 4.3) and as `coverage_gap` alerts.

**Testing**:
- `Fixture-based: restaurant BOP without liquor liability → high-severity gap flagged`.
- `Unit: gap result conforms to coverage_gaps schema; benchmark lookup by NAICS works`.

---

## Phase 8: Commissions, Renewals & Alerts

### Purpose
Complete the agency-management value: track commissions and reconcile carrier statements, and run the renewal/expiration lifecycle with automated alerts and re-marketing — addressing the underserved "intelligent renewal" opportunity.

### Tasks

#### 8.1 — Commission tracking & reconciliation
**What**: `commission_schedule` and `commission_transaction` tables plus reconciliation.

**Design** (DDL from suggestion 3 §"Commission"): schedule with JSONB `rules` (tiers, loss-ratio caps); transaction with relational splits and JSONB `reconciliation` (`carrier_statement_ref,expected_amount,actual_amount,variance,reconciled_at,dispute_reason`). On bind, `CommissionExpected` is computed from the applicable schedule. A statement-import endpoint matches line items to expected transactions and flags variances/disputes.

**Testing**:
- `Unit: tiered schedule selects correct rate at a given premium volume`.
- `Integration: import statement → matched transactions reconciled, mismatches → disputed`.

#### 8.2 — Renewal management & alerts
**What**: `alert` records and a scheduled renewal scanner.

**Design**: Celery beat job scans active policies daily; at expiration−90d creates a renewal alert and (configurable) auto-creates a re-marketing `quote_request` seeded from prior policy data. `alert` types (`renewal,expiration,payment_due,coverage_gap,retention_risk,commission_discrepancy,task`) with severity and assignment. Renewal pipeline endpoint lists upcoming expirations with days-to-expiration.

**Testing**:
- `Integration: policy expiring in 89 days → renewal alert created once (idempotent on re-scan)`.
- `Integration: auto-remarket enabled → new quote_request created with prior risks`.
- `Unit: days_to_expiration computed correctly across month boundaries`.

---

## Phase 9: Agent Console & White-Label Consumer Portal

### Purpose
Deliver the human-facing surfaces: the producer/CSR/principal console for quoting and AMS, and the brandable self-service consumer portal — turning the API into a usable product and enabling the consumer-distribution value prop.

### Tasks

#### 9.1 — Agent console (Next.js)
**What**: Web UI for quoting, comparison, clients, policies, renewals, commissions.

**Design**: Next.js 15 app consuming a typed client generated from the OpenAPI spec. Key views: quote intake (progressive single-entry form), live comparison grid (streams partial carrier results, shows gap warnings), client/policy records, renewal pipeline, commission dashboard. Role-based navigation from JWT claims.

**Testing**:
- `E2E (Playwright): producer logs in, creates auto quote, sees ≥2 carrier quotes appear, binds one → policy visible`.
- `E2E: CSR has no Bind button (RBAC reflected in UI)`.

#### 9.2 — White-label consumer portal & config
**What**: Brandable self-service quoting via `white_label_config`.

**Design** (DDL from suggestion 3 §"White-Label & Workflow"): `white_label_config` with JSONB `branding` (logo, colors, css) and `portal_config` (`enabled_lines,question_flows,disclaimer_text,lead_capture_fields`). Public endpoints under `/api/v1/portal/*` resolve agency by `subdomain`/`custom_domain`, present progressive question flows, and create portal-sourced quote requests (`source_channel:portal`). No agent auth; rate-limited and CAPTCHA-gated.

**Testing**:
- `E2E: visit agency subdomain → branded portal, complete homeowner flow → comparison shown`.
- `Integration: portal quote request tagged source_channel=portal and scoped to correct agency`.
- `Unit: question-flow renderer respects enabled_lines config`.

---

## Phase 10: Embedded Insurance API, No-Code Workflows & Hardening

### Purpose
Capture the fastest-growing distribution channel (embedded insurance) and the no-code workflow differentiator, then harden the platform for production (security, performance, compliance posture). Backlog features land here.

### Tasks

#### 10.1 — Embedded insurance API & web component SDK
**What**: Headless embedding for non-insurance platforms.

**Design**: Scoped API keys for partner platforms; embeddable Lit web component (`<insurance-quote-widget agency-key=…>`) that drives the portal quote flow inside any site, posting to `/api/v1/portal/*`. Webhooks (Phase 6.3) notify partners of quote/bind events.

**Testing**:
- `E2E: widget embedded in a test host page → completes a quote, fires partner webhook`.
- `Integration: partner API key scoped to its agency only`.

#### 10.2 — No-code workflow builder
**What**: `workflow_definition`-driven automation.

**Design** (DDL from suggestion 3 §"Workflow"): JSONB `definition` (`trigger{type,conditions}`, `steps[{id,type:action|condition|wait|notification,config,next_step_id,on_failure}]`). A workflow engine evaluates triggers (quote_request, renewal, new_client) and executes steps via Celery.

**Testing**:
- `Unit: engine executes a linear 3-step workflow; condition step branches correctly`.
- `Integration: new_client trigger → notification step fires`.

#### 10.3 — Security & compliance hardening
**What**: Production posture for SOC 2 / NAIC #668 / GLBA / CCPA/GDPR.

**Design**: rate limiting, security headers, dependency scanning in CI, secret rotation, data-subject export/delete endpoints (CCPA/GDPR right to erasure), encryption-in-transit (TLS 1.3) and at-rest, breach-notification logging hooks (NAIC 3-day rule), least-privilege DB roles, audit-log immutability checks.

**Testing**:
- `Integration: GDPR delete request → client PII erased, audit retains non-PII action record`.
- `Integration: rate limit exceeded → 429`.
- `Security: automated dependency + SAST scan passes in CI; no high-severity findings`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Tenancy            ─── required by everything
    │
Phase 2: Auth, RBAC & Audit              ─── requires 1
    │
Phase 3: Domain Core (Client/Risk/Carrier) ─ requires 1,2
    │
Phase 4: Carrier Integration & Quoting   ─── requires 3   [CORE VALUE]
    │
    ├── Phase 5: Quote-to-Bind/Policies/Docs ── requires 4
    │       │
    │       └── Phase 8: Commissions/Renewals ── requires 5
    │
    ├── Phase 6: Vault/Payments/Webhooks  ─── requires 4 (can parallel with 5)
    │
    └── Phase 7: AI (Extraction/Appetite/Gaps) ── 7.3 used by 4; 7.2 needs 5.3; can parallel with 5/6
            │
Phase 9: Agent Console & Portal          ─── requires 4,5 (portal needs 3); UI parallels backend 6/7/8
    │
Phase 10: Embedded API / No-Code / Hardening ── requires 6,9
```

**Parallelism opportunities**:
- After Phase 4: Phases 5, 6, and 7 can be developed concurrently (different teams/agents).
- Phase 9 (frontend) can begin against the OpenAPI spec as soon as Phases 4–5 endpoints stabilise, in parallel with 6/7/8.
- Within Phase 4, the 10–15 carrier adapters (4.4) can be built in parallel once 4.1 lands.

---

## Definition of Done (per phase)

Every phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests pass (`pytest`), including the named scenarios above.
3. Linting and formatting pass (`ruff check`, `ruff format --check`).
4. Type checking passes (`mypy --strict`).
5. Docker build succeeds and `docker compose up` brings the stack healthy.
6. The phase's feature works end-to-end (demonstrable via API call, CLI, or UI).
7. New configuration options are documented in `.env.example` and README.
8. New/changed API endpoints appear correctly in the generated OpenAPI 3.1 spec; new events appear in the AsyncAPI catalog.
9. Database changes have an Alembic migration that applies and rolls back cleanly.
10. Mutating actions write audit-log entries; any PII handling is covered by the compliance hooks (from Phase 2 onward).
11. JSONB payloads have a corresponding Pydantic/JSON Schema validating them at the application boundary.
```
