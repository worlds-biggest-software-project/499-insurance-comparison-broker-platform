# Data Model Suggestion 3: Hybrid Relational + JSONB Model (PostgreSQL)

## Overview

A pragmatic hybrid approach using PostgreSQL with relational tables for core entities and JSONB columns for data that varies by line of business, carrier response format, or AI extraction output. This preserves referential integrity and SQL queryability for the stable, well-defined parts of the domain while embracing schema flexibility where insurance data is inherently polymorphic -- risk details differ wildly between auto, property, GL, and workers' comp; carrier API responses have no universal format; and AI-extracted fields are unpredictable by nature.

## Why This Suits an Insurance Comparison & Broker Platform

The central tension in insurance data modeling is that the *structure* of the domain is stable (clients have risks, risks get quoted, quotes become policies) but the *details* within each entity vary enormously. A homeowner risk has `construction_type`, `year_built`, and `roof_material`. A commercial auto risk has `fleet_size`, `radius_of_operation`, and `driver_schedules`. A normalized approach forces you into EAV tables or dozens of line-specific tables. A pure document store loses the relational integrity you need for commission accounting and policy lifecycle management.

JSONB columns give you the best of both worlds: stable foreign keys and indexed relational columns for the parts that every query needs, plus flexible nested structures for the parts that change by line, carrier, or context. PostgreSQL's GIN indexes on JSONB make these columns queryable at scale, and `jsonb_path_query` / `@>` operators allow precise filtering without sacrificing performance.

## Schema Definition

### Core Agency & User Entities

```sql
CREATE TABLE agency (
    agency_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    legal_name      VARCHAR(255),
    tax_id          VARCHAR(20),
    license_number  VARCHAR(50),
    license_state   CHAR(2),
    contact         JSONB NOT NULL DEFAULT '{}',
    -- contact: { address: { line1, line2, city, state, zip },
    --            phone, email, website }
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings: { timezone, date_format, default_lines, notification_prefs }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE agency_user (
    user_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agency_id       UUID NOT NULL REFERENCES agency(agency_id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    role            VARCHAR(30) NOT NULL CHECK (role IN ('principal','producer','csr','admin')),
    license_info    JSONB NOT NULL DEFAULT '{}',
    -- license_info: { number, state, expiration, lines: ["property","casualty"] }
    preferences     JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_agency_user_agency ON agency_user(agency_id);
```

### Client Entity with Flexible Attributes

```sql
CREATE TABLE client (
    client_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agency_id       UUID NOT NULL REFERENCES agency(agency_id),
    client_type     VARCHAR(20) NOT NULL CHECK (client_type IN ('individual','business')),
    -- Searchable relational columns
    display_name    VARCHAR(255) NOT NULL,  -- computed: "last, first" or business_name
    email           VARCHAR(255),
    phone           VARCHAR(20),
    state           CHAR(2),
    zip_code        VARCHAR(10),
    naics_code      VARCHAR(10),
    assigned_producer_id UUID REFERENCES agency_user(user_id),
    assigned_csr_id      UUID REFERENCES agency_user(user_id),
    -- Flexible attributes in JSONB
    profile         JSONB NOT NULL DEFAULT '{}',
    -- For individuals:
    --   { first_name, last_name, date_of_birth, ssn_last4, drivers_license,
    --     address: { line1, line2, city, state, zip },
    --     additional_insureds: [...] }
    -- For businesses:
    --   { business_name, dba, tax_id, entity_type, years_in_business,
    --     annual_revenue, employee_count, address: {...},
    --     officers: [{ name, title, ownership_pct }],
    --     locations: [{ address, operations, payroll }] }
    source_info     JSONB NOT NULL DEFAULT '{}',
    -- { source_channel, referral, marketing_campaign, portal_session_id }
    tags            TEXT[] DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_client_agency ON client(agency_id);
CREATE INDEX idx_client_name ON client(display_name);
CREATE INDEX idx_client_state ON client(state);
CREATE INDEX idx_client_naics ON client(naics_code) WHERE naics_code IS NOT NULL;
CREATE INDEX idx_client_profile ON client USING GIN (profile jsonb_path_ops);
CREATE INDEX idx_client_tags ON client USING GIN (tags);
```

### Carrier & Appetite with JSONB Flexibility

```sql
CREATE TABLE carrier (
    carrier_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    am_best_rating  VARCHAR(10),
    naic_code       VARCHAR(10),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- API integration config -- varies by carrier
    api_config      JSONB NOT NULL DEFAULT '{}',
    -- { endpoint, type: "rest"|"soap"|"al3"|"manual",
    --   auth: { method, credentials_vault_ref },
    --   rate_limits: { requests_per_minute, concurrent },
    --   field_mappings: { our_field: their_field, ... },
    --   supported_lines: [...],
    --   response_format: { quote_path, premium_path, ... } }
    contact         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE carrier_appetite (
    appetite_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    carrier_id      UUID NOT NULL REFERENCES carrier(carrier_id),
    line_of_business VARCHAR(50) NOT NULL,
    state           CHAR(2) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    effective_date  DATE NOT NULL,
    expiration_date DATE,
    -- Flexible appetite criteria -- varies hugely by line and carrier
    criteria        JSONB NOT NULL DEFAULT '{}',
    -- { naics_codes: ["722511","722513"], naics_exclusions: ["722410"],
    --   min_premium: 1000, max_premium: 500000,
    --   min_years_in_business: 3,
    --   prohibited_operations: ["demolition","blasting"],
    --   max_loss_ratio: 0.65,
    --   territory_restrictions: { excluded_zips: [...] },
    --   class_codes: { eligible: [...], preferred: [...] },
    --   notes: "No monoline GL; must package with property" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_appetite_carrier ON carrier_appetite(carrier_id);
CREATE INDEX idx_appetite_lob_state ON carrier_appetite(line_of_business, state);
CREATE INDEX idx_appetite_criteria ON carrier_appetite USING GIN (criteria jsonb_path_ops);
```

### Polymorphic Risk Model

```sql
-- Risk details are the most polymorphic data in the system.
-- The relational columns handle cross-line queries; JSONB handles line-specific data.
CREATE TABLE risk (
    risk_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(client_id),
    risk_type       VARCHAR(50) NOT NULL CHECK (risk_type IN (
        'auto','homeowner','renters','commercial_property',
        'general_liability','bop','workers_comp','professional_liability',
        'commercial_auto','umbrella','other'
    )),
    description     TEXT,
    -- Common searchable location fields
    state           CHAR(2),
    zip_code        VARCHAR(10),
    -- Line-specific details in JSONB
    details         JSONB NOT NULL DEFAULT '{}',
    -- Auto example:
    --   { year: 2024, make: "Toyota", model: "Camry", vin: "...",
    --     usage: "commute", annual_mileage: 12000,
    --     drivers: [{ name, dob, license_number, violations: [...] }],
    --     garaging_address: {...} }
    -- Homeowner example:
    --   { year_built: 1985, construction_type: "frame",
    --     square_footage: 2400, stories: 2, roof_type: "asphalt_shingle",
    --     roof_year: 2018, heating_type: "forced_air",
    --     protection_class: 4, fire_hydrant_distance_ft: 500,
    --     replacement_cost: 450000, pool: false, trampoline: false }
    -- Workers Comp example:
    --   { class_codes: [{ code: "8810", description: "Clerical", payroll: 250000 },
    --                    { code: "8742", description: "Sales", payroll: 180000 }],
    --     experience_mod: 0.95, governing_state: "CA",
    --     prior_carrier: "State Fund", years_with_carrier: 3 }
    acord_mappings  JSONB NOT NULL DEFAULT '{}',
    -- { field_name: acord_element_ref, ... } for ACORD form population
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_risk_client ON risk(client_id);
CREATE INDEX idx_risk_type ON risk(risk_type);
CREATE INDEX idx_risk_state ON risk(state);
CREATE INDEX idx_risk_details ON risk USING GIN (details jsonb_path_ops);
```

### Quote Request & Carrier Quotes

```sql
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
    source_channel  VARCHAR(30),
    risk_ids        UUID[] NOT NULL DEFAULT '{}',  -- array of risk IDs included
    -- Pre-screening results stored as JSONB
    appetite_results JSONB NOT NULL DEFAULT '{}',
    -- { eligible: [{ carrier_id, score, reasons }],
    --   ineligible: [{ carrier_id, reasons }],
    --   model_version, screened_at }
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_qr_client ON quote_request(client_id);
CREATE INDEX idx_qr_agency ON quote_request(agency_id);
CREATE INDEX idx_qr_status ON quote_request(status);
CREATE INDEX idx_qr_risk_ids ON quote_request USING GIN (risk_ids);

CREATE TABLE quote (
    quote_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id      UUID NOT NULL REFERENCES quote_request(request_id),
    carrier_id      UUID NOT NULL REFERENCES carrier(carrier_id),
    -- Relational columns for comparison queries
    status          VARCHAR(30) NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending','returned','declined','referred','bound','expired','rejected'
    )),
    total_premium   NUMERIC(12,2),
    annual_premium  NUMERIC(12,2),
    term_months     INTEGER DEFAULT 12,
    quote_number    VARCHAR(100),
    quoted_at       TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    -- Full quote details and coverages in JSONB
    coverages       JSONB NOT NULL DEFAULT '[]',
    -- [{ code, name, limit, deductible, premium, is_included,
    --    sub_limits: {...}, endorsements: [...] }]
    rating_factors  JSONB NOT NULL DEFAULT '{}',
    -- { base_rate, territory_factor, experience_mod, schedule_credit,
    --   loss_history_factor, package_discount, ... }
    declination     JSONB,
    -- { reason, underwriter_notes, alternative_markets }
    carrier_response_raw JSONB,
    -- Raw API response for debugging and audit
    coverage_gaps   JSONB NOT NULL DEFAULT '[]',
    -- [{ gap_type, description, severity, benchmark, model_version }]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_quote_request ON quote(request_id);
CREATE INDEX idx_quote_carrier ON quote(carrier_id);
CREATE INDEX idx_quote_status ON quote(status);
CREATE INDEX idx_quote_premium ON quote(annual_premium) WHERE status = 'returned';
CREATE INDEX idx_quote_coverages ON quote USING GIN (coverages jsonb_path_ops);
```

### Policy with JSONB Endorsement History

```sql
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
    bound_by        UUID REFERENCES agency_user(user_id),
    bound_at        TIMESTAMPTZ,
    prior_policy_id UUID REFERENCES policy(policy_id),
    -- Coverages snapshot at bind time
    coverages       JSONB NOT NULL DEFAULT '[]',
    -- Same structure as quote.coverages
    -- Endorsement history -- append-only array
    endorsements    JSONB NOT NULL DEFAULT '[]',
    -- [{ endorsement_id, type, effective_date, description,
    --    changes: { added: [...], removed: [...], modified: [...] },
    --    premium_change, applied_at }]
    -- Cancellation/non-renewal details
    termination     JSONB,
    -- { type: "cancelled"|"non_renewed", reason, effective_date,
    --   return_premium, notified_at, requested_by }
    -- Payment and billing info
    billing         JSONB NOT NULL DEFAULT '{}',
    -- { payment_plan, billing_type: "direct"|"agency", installments: [...] }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_policy_client ON policy(client_id);
CREATE INDEX idx_policy_agency ON policy(agency_id);
CREATE INDEX idx_policy_carrier ON policy(carrier_id);
CREATE INDEX idx_policy_number ON policy(policy_number);
CREATE INDEX idx_policy_expiration ON policy(expiration_date) WHERE status = 'active';
CREATE INDEX idx_policy_coverages ON policy USING GIN (coverages jsonb_path_ops);
```

### Commission Tracking

```sql
CREATE TABLE commission_schedule (
    schedule_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    carrier_id      UUID NOT NULL REFERENCES carrier(carrier_id),
    line_of_business VARCHAR(50),
    commission_type VARCHAR(20) NOT NULL CHECK (commission_type IN (
        'new_business','renewal','bonus','override','contingency'
    )),
    rate_percent    NUMERIC(5,2) NOT NULL,
    effective_date  DATE NOT NULL,
    expiration_date DATE,
    -- Tiered/conditional commission rules
    rules           JSONB NOT NULL DEFAULT '{}',
    -- { min_premium: 5000, volume_tiers: [{ threshold, rate }],
    --   loss_ratio_cap: 0.60, eligible_products: [...] }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_comm_sched_carrier ON commission_schedule(carrier_id);

CREATE TABLE commission_transaction (
    transaction_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES policy(policy_id),
    schedule_id     UUID REFERENCES commission_schedule(schedule_id),
    producer_id     UUID REFERENCES agency_user(user_id),
    gross_commission NUMERIC(12,2) NOT NULL,
    agency_split    NUMERIC(12,2),
    producer_split  NUMERIC(12,2),
    split_percent   NUMERIC(5,2),
    status          VARCHAR(20) NOT NULL DEFAULT 'expected' CHECK (status IN (
        'expected','received','reconciled','disputed','written_off'
    )),
    statement_date  DATE,
    payment_date    DATE,
    -- Reconciliation details
    reconciliation  JSONB NOT NULL DEFAULT '{}',
    -- { carrier_statement_ref, expected_amount, actual_amount, variance,
    --   reconciled_at, notes, dispute_reason }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_comm_txn_policy ON commission_transaction(policy_id);
CREATE INDEX idx_comm_txn_producer ON commission_transaction(producer_id);
CREATE INDEX idx_comm_txn_status ON commission_transaction(status);
```

### Documents & AI Processing

```sql
CREATE TABLE document (
    document_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agency_id       UUID NOT NULL REFERENCES agency(agency_id),
    client_id       UUID REFERENCES client(client_id),
    policy_id       UUID REFERENCES policy(policy_id),
    document_type   VARCHAR(50) NOT NULL,
    file_name       VARCHAR(255) NOT NULL,
    file_path       VARCHAR(500) NOT NULL,
    mime_type       VARCHAR(100),
    file_size_bytes BIGINT,
    -- AI extraction results stored alongside the document
    extraction      JSONB NOT NULL DEFAULT '{}',
    -- { status: "pending"|"completed"|"reviewed"|"rejected",
    --   model_version, overall_confidence,
    --   fields: [{ name, value, confidence, acord_ref, page, bbox }],
    --   reviewed_by, reviewed_at,
    --   corrections: [{ field, old_value, new_value }] }
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- { acord_form_number, uploaded_by, tags, description }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_doc_client ON document(client_id);
CREATE INDEX idx_doc_policy ON document(policy_id);
CREATE INDEX idx_doc_type ON document(document_type);
CREATE INDEX idx_doc_extraction ON document USING GIN (extraction jsonb_path_ops);
```

### White-Label & Workflow Configuration

```sql
CREATE TABLE white_label_config (
    config_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agency_id       UUID NOT NULL UNIQUE REFERENCES agency(agency_id),
    subdomain       VARCHAR(100) UNIQUE,
    custom_domain   VARCHAR(255) UNIQUE,
    -- Flexible branding and flow config
    branding        JSONB NOT NULL DEFAULT '{}',
    -- { logo_url, favicon_url, primary_color, secondary_color,
    --   font_family, custom_css, header_html, footer_html }
    portal_config   JSONB NOT NULL DEFAULT '{}',
    -- { enabled_lines: ["auto","homeowner","renters"],
    --   question_flows: { auto: { steps: [...] }, homeowner: { steps: [...] } },
    --   disclaimer_text, privacy_policy_url,
    --   lead_capture_fields: [...] }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- No-code workflow definitions
CREATE TABLE workflow_definition (
    workflow_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agency_id       UUID NOT NULL REFERENCES agency(agency_id),
    name            VARCHAR(255) NOT NULL,
    trigger_type    VARCHAR(50) NOT NULL,  -- 'quote_request','renewal','new_client', etc.
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Complete workflow definition in JSONB
    definition      JSONB NOT NULL,
    -- { trigger: { type, conditions: {...} },
    --   steps: [{ id, type: "action"|"condition"|"wait"|"notification",
    --             config: {...}, next_step_id, on_failure }],
    --   variables: { ... } }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_workflow_agency ON workflow_definition(agency_id);
CREATE INDEX idx_workflow_trigger ON workflow_definition(trigger_type);
```

### Audit Log

```sql
CREATE TABLE audit_log (
    log_id          BIGSERIAL PRIMARY KEY,
    agency_id       UUID NOT NULL,
    user_id         UUID,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,
    changes         JSONB,
    -- { field_name: { old: ..., new: ... }, ... }
    request_context JSONB,
    -- { ip_address, user_agent, session_id, correlation_id }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE audit_log_2026_01 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE audit_log_2026_02 PARTITION OF audit_log
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... additional partitions created by maintenance job

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_changes ON audit_log USING GIN (changes jsonb_path_ops);
```

## Key Query Patterns

### Side-by-Side Quote Comparison

```sql
SELECT q.quote_id, c.name AS carrier_name, c.am_best_rating,
       q.total_premium, q.annual_premium, q.status,
       q.coverages, q.coverage_gaps, q.rating_factors
FROM quote q
JOIN carrier c ON c.carrier_id = q.carrier_id
WHERE q.request_id = $1 AND q.status IN ('returned','referred')
ORDER BY q.annual_premium ASC;
```

### Find Risks with Specific Attributes

```sql
-- Find all commercial properties over 50,000 sq ft in California
SELECT r.risk_id, r.details->>'building_name' AS building,
       (r.details->>'square_footage')::int AS sqft
FROM risk r
WHERE r.risk_type = 'commercial_property'
  AND r.state = 'CA'
  AND (r.details->>'square_footage')::int > 50000;
```

### Appetite Matching with JSONB Criteria

```sql
-- Find carriers whose appetite matches a risk
SELECT ca.carrier_id, c.name, ca.criteria
FROM carrier_appetite ca
JOIN carrier c ON c.carrier_id = ca.carrier_id
WHERE ca.line_of_business = 'general_liability'
  AND ca.state = 'TX'
  AND ca.is_active = true
  AND ca.criteria @> '{"naics_codes": ["722511"]}'::jsonb;
```

## Trade-offs

**Strengths:**
- Eliminates the EAV anti-pattern for risk details; each risk type stores its natural structure
- Carrier API config, response formats, and appetite rules can differ per carrier without schema changes
- Coverage details, rating factors, and endorsement histories are naturally nested data -- JSONB represents them cleanly
- GIN indexes on JSONB columns enable efficient queries into flexible data
- No schema migration needed when adding a new line of business or carrier
- Single database engine (PostgreSQL) -- no polyglot persistence complexity
- Retains full relational integrity for the stable core: agency-client-policy-commission relationships

**Weaknesses:**
- JSONB columns are opaque to the ORM type system; application code must validate structure
- Need JSON Schema validation at the application layer or via PostgreSQL CHECK constraints
- GIN indexes use more disk space than B-tree indexes on scalar columns
- JSONB updates require rewriting the entire JSONB value (no partial update in standard PostgreSQL; pg 16+ has `jsonb_set` improvements)
- Reporting tools and BI integrations may struggle with JSONB columns -- may need to expose flattened views
- Risk of "schema drift" if JSONB structures are not documented and validated consistently

## Scalability Considerations

- Audit log table partitioned by month from day one
- Quote and policy tables partitioned by `created_at` when volume warrants it
- GIN indexes should be monitored for bloat; periodic `REINDEX CONCURRENTLY` in maintenance windows
- Consider `jsonb_to_record` / `jsonb_to_recordset` for materializing flattened reporting views
- Connection pooling and read replicas as in the normalized model
- JSONB columns compress well with PostgreSQL TOAST

## Migration Path

This hybrid model is the most pragmatic starting point for a new project. It avoids premature normalization of volatile data while keeping strong relational bones. If specific JSONB structures stabilize over time (e.g., auto risk details become well-defined), those can be extracted into dedicated relational columns or tables with a straightforward migration. If event sourcing is adopted later, the JSONB columns already contain the kind of structured payloads that event stores expect -- the migration to event payloads is largely a copy operation.
