# Data Model Suggestion 1: Normalized Relational Model (PostgreSQL)

## Overview

A traditional third-normal-form (3NF+) relational schema in PostgreSQL. Every entity gets its own table with proper foreign keys, check constraints, and indexes. This approach maximizes data integrity, enforces referential constraints at the database level, and aligns naturally with the ACORD information model's entity-relationship structure (Party, Policy, Product, Claim).

## Why This Suits an Insurance Comparison & Broker Platform

Insurance data is inherently relational: a **Party** (client or carrier) holds **Policies**, each policy contains **Coverages**, coverages attach to **Risks** (insurable items), and **Quotes** reference both carriers and risks. Commission accounting, renewal workflows, and ACORD-compliant data interchange all depend on well-defined relationships and referential integrity. A normalized model prevents data anomalies in premium accounting, ensures audit trails for regulatory compliance (PCI DSS, GDPR/CCPA), and maps directly to ACORD AL3/XML interchange schemas.

## Schema Definition

### Core Party & Agency Entities

```sql
-- Agencies using the platform
CREATE TABLE agency (
    agency_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    legal_name      VARCHAR(255),
    tax_id          VARCHAR(20),
    license_number  VARCHAR(50),
    license_state   CHAR(2),
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state           CHAR(2),
    zip_code        VARCHAR(10),
    phone           VARCHAR(20),
    email           VARCHAR(255),
    website         VARCHAR(255),
    white_label_config_id UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users within agencies (producers, CSRs, principals)
CREATE TABLE agency_user (
    user_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agency_id       UUID NOT NULL REFERENCES agency(agency_id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    role            VARCHAR(30) NOT NULL CHECK (role IN ('principal','producer','csr','admin')),
    license_number  VARCHAR(50),
    license_state   CHAR(2),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_agency_user_agency ON agency_user(agency_id);

-- Clients (insureds / prospects)
CREATE TABLE client (
    client_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agency_id       UUID NOT NULL REFERENCES agency(agency_id),
    client_type     VARCHAR(20) NOT NULL CHECK (client_type IN ('individual','business')),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    business_name   VARCHAR(255),
    tax_id          VARCHAR(20),
    naics_code      VARCHAR(10),
    date_of_birth   DATE,
    email           VARCHAR(255),
    phone           VARCHAR(20),
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state           CHAR(2),
    zip_code        VARCHAR(10),
    assigned_producer_id UUID REFERENCES agency_user(user_id),
    assigned_csr_id      UUID REFERENCES agency_user(user_id),
    source          VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_client_agency ON client(agency_id);
CREATE INDEX idx_client_name ON client(last_name, first_name);
CREATE INDEX idx_client_business ON client(business_name) WHERE business_name IS NOT NULL;
```

### Carrier & Product Catalog

```sql
-- Insurance carriers
CREATE TABLE carrier (
    carrier_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    am_best_rating  VARCHAR(10),
    naic_code       VARCHAR(10),
    api_endpoint    VARCHAR(500),
    api_type        VARCHAR(30) CHECK (api_type IN ('rest','soap','al3','manual')),
    api_credentials_vault_ref VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Carrier appetite rules (which risks a carrier will write)
CREATE TABLE carrier_appetite (
    appetite_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    carrier_id      UUID NOT NULL REFERENCES carrier(carrier_id),
    line_of_business VARCHAR(50) NOT NULL,
    state           CHAR(2) NOT NULL,
    naics_codes     VARCHAR(500),          -- comma-separated or range
    min_premium     NUMERIC(12,2),
    max_premium     NUMERIC(12,2),
    risk_criteria   TEXT,                   -- human-readable appetite notes
    is_active       BOOLEAN NOT NULL DEFAULT true,
    effective_date  DATE NOT NULL,
    expiration_date DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_appetite_carrier ON carrier_appetite(carrier_id);
CREATE INDEX idx_appetite_lob_state ON carrier_appetite(line_of_business, state);

-- Products offered by carriers
CREATE TABLE carrier_product (
    product_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    carrier_id      UUID NOT NULL REFERENCES carrier(carrier_id),
    product_name    VARCHAR(255) NOT NULL,
    line_of_business VARCHAR(50) NOT NULL,
    product_code    VARCHAR(50),
    is_bindable_online BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_product_carrier ON carrier_product(carrier_id);
CREATE INDEX idx_product_lob ON carrier_product(line_of_business);
```

### Risk & Quoting Entities

```sql
-- Insurable risks (vehicles, properties, business operations, etc.)
CREATE TABLE risk (
    risk_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(client_id),
    risk_type       VARCHAR(50) NOT NULL CHECK (risk_type IN (
        'auto','homeowner','renters','commercial_property',
        'general_liability','bop','workers_comp','professional_liability',
        'commercial_auto','umbrella','other'
    )),
    description     TEXT,
    address_line1   VARCHAR(255),
    city            VARCHAR(100),
    state           CHAR(2),
    zip_code        VARCHAR(10),
    naics_code      VARCHAR(10),
    effective_date  DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_risk_client ON risk(client_id);

-- Risk detail key-value pairs (year/make/model for auto, sq ft for property, etc.)
CREATE TABLE risk_detail (
    detail_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    risk_id         UUID NOT NULL REFERENCES risk(risk_id) ON DELETE CASCADE,
    field_name      VARCHAR(100) NOT NULL,
    field_value     TEXT NOT NULL,
    acord_field_ref VARCHAR(50),
    UNIQUE (risk_id, field_name)
);
CREATE INDEX idx_risk_detail_risk ON risk_detail(risk_id);

-- Quote requests (a single submission that fans out to multiple carriers)
CREATE TABLE quote_request (
    request_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(client_id),
    agency_id       UUID NOT NULL REFERENCES agency(agency_id),
    requested_by    UUID NOT NULL REFERENCES agency_user(user_id),
    line_of_business VARCHAR(50) NOT NULL,
    effective_date  DATE NOT NULL,
    expiration_date DATE NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft','submitted','quoting','completed','expired','cancelled'
    )),
    source_channel  VARCHAR(30) CHECK (source_channel IN ('agent','portal','api','embedded')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_quote_request_client ON quote_request(client_id);
CREATE INDEX idx_quote_request_agency ON quote_request(agency_id);
CREATE INDEX idx_quote_request_status ON quote_request(status);

-- Junction: risks included in a quote request
CREATE TABLE quote_request_risk (
    request_id      UUID NOT NULL REFERENCES quote_request(request_id),
    risk_id         UUID NOT NULL REFERENCES risk(risk_id),
    PRIMARY KEY (request_id, risk_id)
);

-- Individual carrier quotes returned for a request
CREATE TABLE quote (
    quote_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id      UUID NOT NULL REFERENCES quote_request(request_id),
    carrier_id      UUID NOT NULL REFERENCES carrier(carrier_id),
    product_id      UUID REFERENCES carrier_product(product_id),
    quote_number    VARCHAR(100),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending','returned','declined','referred','bound','expired','rejected'
    )),
    total_premium   NUMERIC(12,2),
    annual_premium  NUMERIC(12,2),
    term_months     INTEGER DEFAULT 12,
    deductible      NUMERIC(12,2),
    effective_date  DATE,
    expiration_date DATE,
    declination_reason TEXT,
    carrier_response_raw TEXT,
    quoted_at       TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_quote_request ON quote(request_id);
CREATE INDEX idx_quote_carrier ON quote(carrier_id);
CREATE INDEX idx_quote_status ON quote(status);

-- Coverage-level detail within a quote
CREATE TABLE quote_coverage (
    coverage_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    quote_id        UUID NOT NULL REFERENCES quote(quote_id) ON DELETE CASCADE,
    coverage_code   VARCHAR(50) NOT NULL,
    coverage_name   VARCHAR(255) NOT NULL,
    coverage_limit  NUMERIC(14,2),
    deductible      NUMERIC(12,2),
    premium         NUMERIC(12,2),
    is_included     BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_quote_coverage_quote ON quote_coverage(quote_id);
```

### Policy & Binding

```sql
-- Bound policies
CREATE TABLE policy (
    policy_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    quote_id        UUID REFERENCES quote(quote_id),
    client_id       UUID NOT NULL REFERENCES client(client_id),
    agency_id       UUID NOT NULL REFERENCES agency(agency_id),
    carrier_id      UUID NOT NULL REFERENCES carrier(carrier_id),
    policy_number   VARCHAR(100) NOT NULL,
    line_of_business VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'active' CHECK (status IN (
        'active','cancelled','expired','non_renewed','reinstated','pending'
    )),
    effective_date  DATE NOT NULL,
    expiration_date DATE NOT NULL,
    total_premium   NUMERIC(12,2) NOT NULL,
    payment_plan    VARCHAR(30),
    bound_by        UUID REFERENCES agency_user(user_id),
    bound_at        TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    cancellation_reason TEXT,
    prior_policy_id UUID REFERENCES policy(policy_id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_policy_client ON policy(client_id);
CREATE INDEX idx_policy_agency ON policy(agency_id);
CREATE INDEX idx_policy_carrier ON policy(carrier_id);
CREATE INDEX idx_policy_number ON policy(policy_number);
CREATE INDEX idx_policy_expiration ON policy(expiration_date) WHERE status = 'active';

-- Coverages on a bound policy
CREATE TABLE policy_coverage (
    coverage_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES policy(policy_id) ON DELETE CASCADE,
    coverage_code   VARCHAR(50) NOT NULL,
    coverage_name   VARCHAR(255) NOT NULL,
    coverage_limit  NUMERIC(14,2),
    deductible      NUMERIC(12,2),
    premium         NUMERIC(12,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_policy_coverage_policy ON policy_coverage(policy_id);
```

### Commission & Accounting

```sql
-- Commission schedules by carrier/product
CREATE TABLE commission_schedule (
    schedule_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    carrier_id      UUID NOT NULL REFERENCES carrier(carrier_id),
    product_id      UUID REFERENCES carrier_product(product_id),
    line_of_business VARCHAR(50),
    commission_type VARCHAR(20) NOT NULL CHECK (commission_type IN ('new_business','renewal','bonus','override')),
    rate_percent    NUMERIC(5,2) NOT NULL,
    effective_date  DATE NOT NULL,
    expiration_date DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_commission_schedule_carrier ON commission_schedule(carrier_id);

-- Commission transactions
CREATE TABLE commission_transaction (
    transaction_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES policy(policy_id),
    schedule_id     UUID REFERENCES commission_schedule(schedule_id),
    producer_id     UUID REFERENCES agency_user(user_id),
    gross_commission NUMERIC(12,2) NOT NULL,
    agency_split    NUMERIC(12,2),
    producer_split  NUMERIC(12,2),
    split_percent   NUMERIC(5,2),
    statement_date  DATE,
    payment_date    DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'expected' CHECK (status IN (
        'expected','received','reconciled','disputed','written_off'
    )),
    carrier_statement_ref VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_commission_policy ON commission_transaction(policy_id);
CREATE INDEX idx_commission_producer ON commission_transaction(producer_id);
CREATE INDEX idx_commission_status ON commission_transaction(status);
```

### Documents & AI Processing

```sql
-- Document storage metadata
CREATE TABLE document (
    document_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agency_id       UUID NOT NULL REFERENCES agency(agency_id),
    client_id       UUID REFERENCES client(client_id),
    policy_id       UUID REFERENCES policy(policy_id),
    document_type   VARCHAR(50) NOT NULL CHECK (document_type IN (
        'application','loss_run','certificate','dec_page','endorsement',
        'audit','correspondence','acord_form','invoice','other'
    )),
    file_name       VARCHAR(255) NOT NULL,
    file_path       VARCHAR(500) NOT NULL,
    mime_type       VARCHAR(100),
    file_size_bytes BIGINT,
    acord_form_number VARCHAR(20),
    uploaded_by     UUID REFERENCES agency_user(user_id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_document_client ON document(client_id);
CREATE INDEX idx_document_policy ON document(policy_id);

-- AI extraction results from documents
CREATE TABLE ai_extraction (
    extraction_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(document_id),
    model_version   VARCHAR(50) NOT NULL,
    extraction_type VARCHAR(50) NOT NULL,
    confidence_score NUMERIC(4,3),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending','completed','reviewed','rejected'
    )),
    reviewed_by     UUID REFERENCES agency_user(user_id),
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_extraction_document ON ai_extraction(document_id);

-- Extracted field values
CREATE TABLE ai_extracted_field (
    field_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    extraction_id   UUID NOT NULL REFERENCES ai_extraction(extraction_id) ON DELETE CASCADE,
    field_name      VARCHAR(100) NOT NULL,
    field_value     TEXT,
    confidence      NUMERIC(4,3),
    acord_field_ref VARCHAR(50),
    page_number     INTEGER,
    bounding_box    VARCHAR(100)
);
CREATE INDEX idx_extracted_field_extraction ON ai_extracted_field(extraction_id);
```

### Workflow, Alerts & Audit

```sql
-- Renewal and task alerts
CREATE TABLE alert (
    alert_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agency_id       UUID NOT NULL REFERENCES agency(agency_id),
    policy_id       UUID REFERENCES policy(policy_id),
    client_id       UUID REFERENCES client(client_id),
    assigned_to     UUID REFERENCES agency_user(user_id),
    alert_type      VARCHAR(50) NOT NULL CHECK (alert_type IN (
        'renewal','expiration','payment_due','coverage_gap',
        'retention_risk','commission_discrepancy','task','other'
    )),
    severity        VARCHAR(10) NOT NULL DEFAULT 'info' CHECK (severity IN ('info','warning','critical')),
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    due_date        DATE,
    is_resolved     BOOLEAN NOT NULL DEFAULT false,
    resolved_at     TIMESTAMPTZ,
    resolved_by     UUID REFERENCES agency_user(user_id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alert_agency ON alert(agency_id);
CREATE INDEX idx_alert_assigned ON alert(assigned_to) WHERE NOT is_resolved;
CREATE INDEX idx_alert_due ON alert(due_date) WHERE NOT is_resolved;

-- Comprehensive audit log
CREATE TABLE audit_log (
    log_id          BIGSERIAL PRIMARY KEY,
    agency_id       UUID NOT NULL REFERENCES agency(agency_id),
    user_id         UUID REFERENCES agency_user(user_id),
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL CHECK (action IN ('create','update','delete','view','export','bind')),
    changes         TEXT,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);

-- White-label portal configuration
CREATE TABLE white_label_config (
    config_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agency_id       UUID NOT NULL REFERENCES agency(agency_id),
    subdomain       VARCHAR(100) UNIQUE,
    custom_domain   VARCHAR(255) UNIQUE,
    logo_url        VARCHAR(500),
    primary_color   VARCHAR(7),
    secondary_color VARCHAR(7),
    custom_css      TEXT,
    enabled_lines   VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_white_label_agency ON white_label_config(agency_id);
```

## Trade-offs

**Strengths:**
- Maximum data integrity through foreign keys, check constraints, and unique constraints
- Clean mapping to ACORD information model entities (Party, Policy, Product)
- Straightforward commission reconciliation and premium accounting queries
- Well-understood by insurance domain developers; broad ORM and tooling support
- Natural fit for regulatory audit and compliance queries
- Standard PostgreSQL -- no extensions or exotic features required

**Weaknesses:**
- Risk detail variability (auto vs. property vs. GL) is handled via an EAV-style `risk_detail` table, which complicates queries for specific risk types
- Adding new coverage lines requires DDL changes or reliance on the `risk_detail` pattern
- Quote comparison queries joining across `quote` + `quote_coverage` for multiple carriers can be verbose
- High write volume from real-time carrier API responses may stress normalized insert patterns
- Schema migrations needed whenever ACORD adds new data elements

## Scalability Considerations

- Table partitioning on `audit_log` by `created_at` (monthly or quarterly) to manage growth
- Partitioning on `quote` by `created_at` for high-volume agencies
- Read replicas for reporting and commission reconciliation workloads
- Connection pooling (PgBouncer) essential for multi-agency SaaS deployment
- Consider materialized views for quote comparison dashboards and commission summaries

## Migration Path

This normalized schema serves well as the canonical data model even if the application evolves toward CQRS or event sourcing. Tables map cleanly to aggregate roots (Client, QuoteRequest, Policy), making it straightforward to layer event-sourced write models on top while keeping these tables as the read projection. The schema is also a natural starting point for adding JSONB columns to specific tables (see Suggestion 3) as flexibility requirements emerge.
