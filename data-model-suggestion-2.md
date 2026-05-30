# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Overview

An event-sourced architecture with Command Query Responsibility Segregation (CQRS). Instead of storing current state in mutable rows, the system persists an immutable append-only log of **domain events** as the source of truth. Current state is derived by replaying events. Read-optimized **projections** are maintained separately for queries, dashboards, and reporting. The write side processes **commands** that validate business rules and emit events; the read side subscribes to those events and updates denormalized views.

## Why This Suits an Insurance Comparison & Broker Platform

Insurance is one of the most audit-intensive domains in software. Regulators, E&O insurers, and carrier auditors routinely ask "what did the agent know, and when?" Event sourcing provides a complete, immutable timeline of every state change -- every quote requested, every coverage adjusted, every bind decision. This is not a nice-to-have; it directly addresses regulatory requirements (state DOI audit trails, GDPR right-to-explanation, PCI DSS change logging).

The quote-to-bind workflow is naturally event-driven: a quote request is submitted, carriers respond asynchronously, an agent selects a quote, binding is requested, and confirmation arrives. CQRS cleanly separates the complex write-side validation (appetite matching, coverage rules, commission calculations) from the read-side denormalized views (quote comparison grids, dashboard widgets, commission reports).

## Event Store Schema (PostgreSQL)

The event store is the single source of truth. All domain state changes are captured here.

```sql
-- Core event store: immutable append-only log
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(50) NOT NULL,    -- 'QuoteRequest', 'Policy', 'Client', etc.
    aggregate_id    UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL,   -- 'QuoteRequested', 'CarrierQuoteReceived', etc.
    event_version   INTEGER NOT NULL,        -- per-aggregate sequence number
    payload         JSONB NOT NULL,          -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}', -- correlation_id, causation_id, user_id, ip, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_id, event_version)
);
CREATE INDEX idx_event_aggregate ON event_store(aggregate_type, aggregate_id);
CREATE INDEX idx_event_type ON event_store(event_type);
CREATE INDEX idx_event_created ON event_store(created_at);

-- Snapshot store for aggregates with long event histories
CREATE TABLE aggregate_snapshot (
    snapshot_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(50) NOT NULL,
    aggregate_id    UUID NOT NULL,
    version         INTEGER NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_snapshot_aggregate ON aggregate_snapshot(aggregate_id, version);

-- Projection checkpoints: tracks which event each projection has processed
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL REFERENCES event_store(event_id),
    last_position   BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Domain Events

### Client Aggregate Events

```
ClientRegistered
  { client_id, agency_id, client_type, name, contact_info, naics_code }

ClientProfileUpdated
  { client_id, changed_fields: { field_name: { old, new } } }

ClientAssignmentChanged
  { client_id, producer_id, csr_id, changed_by }

RiskAdded
  { client_id, risk_id, risk_type, description, location, details: {...} }

RiskUpdated
  { client_id, risk_id, changed_fields: { field_name: { old, new } } }

RiskRemoved
  { client_id, risk_id, reason }
```

### QuoteRequest Aggregate Events

```
QuoteRequestCreated
  { request_id, client_id, agency_id, requested_by, line_of_business,
    effective_date, risk_ids: [...], source_channel }

CarrierSubmissionSent
  { request_id, carrier_id, product_id, submission_payload_ref,
    submitted_at }

CarrierQuoteReceived
  { request_id, quote_id, carrier_id, total_premium, annual_premium,
    term_months, deductible, coverages: [{ code, name, limit, deductible, premium }],
    quote_number, expires_at }

CarrierQuoteDeclined
  { request_id, carrier_id, declination_reason, declined_at }

CarrierQuoteReferred
  { request_id, carrier_id, referral_reason, underwriter_contact }

QuoteSelected
  { request_id, quote_id, selected_by }

QuoteRequestCompleted
  { request_id, selected_quote_id, completed_at }

QuoteRequestExpired
  { request_id, expired_at }

QuoteRequestCancelled
  { request_id, cancelled_by, reason }

AppetitePreScreenCompleted
  { request_id, eligible_carriers: [...], ineligible_carriers: [{ id, reason }],
    model_version }

CoverageGapDetected
  { request_id, quote_id, gap_type, description, severity,
    industry_benchmark, model_version }
```

### Policy Aggregate Events

```
PolicyBound
  { policy_id, quote_id, request_id, client_id, agency_id, carrier_id,
    policy_number, line_of_business, effective_date, expiration_date,
    total_premium, coverages: [...], bound_by, bound_at }

PolicyEndorsementApplied
  { policy_id, endorsement_id, endorsement_type, changes: {...},
    premium_change, effective_date }

PolicyCancelled
  { policy_id, cancelled_by, cancellation_type, reason,
    effective_date, return_premium }

PolicyRenewed
  { policy_id, new_policy_id, new_carrier_id, new_premium,
    premium_change_pct, renewed_at }

PolicyNonRenewed
  { policy_id, reason, notified_at }

PolicyReinstated
  { policy_id, reinstated_by, effective_date }

RenewalMarketingInitiated
  { policy_id, new_request_id, days_before_expiration, initiated_by }
```

### Commission Aggregate Events

```
CommissionExpected
  { transaction_id, policy_id, carrier_id, producer_id, schedule_id,
    gross_amount, agency_split, producer_split, split_percent }

CommissionStatementReceived
  { statement_id, carrier_id, statement_date, line_items: [
    { policy_number, premium, commission_amount, transaction_type }
  ]}

CommissionReconciled
  { transaction_id, statement_id, expected_amount, actual_amount,
    variance, reconciled_at }

CommissionDisputed
  { transaction_id, dispute_reason, disputed_by }

CommissionPaid
  { transaction_id, payment_date, amount, payment_method }
```

### Document & AI Events

```
DocumentUploaded
  { document_id, agency_id, client_id, policy_id, document_type,
    file_name, file_path, acord_form_number, uploaded_by }

AIExtractionStarted
  { extraction_id, document_id, model_version, extraction_type }

AIExtractionCompleted
  { extraction_id, document_id, fields: [{ name, value, confidence, acord_ref }],
    overall_confidence }

AIExtractionReviewed
  { extraction_id, reviewed_by, corrections: [{ field, old_value, new_value }],
    approved }
```

## Command Handlers

Commands represent intentions. Each handler validates preconditions, enforces invariants, and emits events.

```
SubmitQuoteRequest
  Pre: client exists, risks valid, effective_date in future
  Emits: QuoteRequestCreated, then CarrierSubmissionSent for each eligible carrier
  Side-effect: triggers appetite pre-screening, sends API requests to carriers

SelectQuote
  Pre: quote_request is in 'quoting' or 'completed' state, quote is 'returned'
  Emits: QuoteSelected

BindPolicy
  Pre: quote selected, quote not expired, carrier confirms bindability
  Emits: PolicyBound, CommissionExpected
  Side-effect: calls carrier bind API, generates binder document

CancelPolicy
  Pre: policy is active, cancellation date valid per carrier rules
  Emits: PolicyCancelled
  Side-effect: notifies carrier via API

InitiateRenewal
  Pre: policy active, expiration within renewal window
  Emits: RenewalMarketingInitiated
  Side-effect: creates new QuoteRequest with prior policy data

ReconcileCommission
  Pre: statement received, matching policy found
  Emits: CommissionReconciled or CommissionDisputed

UploadDocument
  Pre: file validated, virus scanned
  Emits: DocumentUploaded, AIExtractionStarted
  Side-effect: queues document for AI extraction pipeline

ReviewExtraction
  Pre: extraction completed, user has review permission
  Emits: AIExtractionReviewed
```

## Read Projections (PostgreSQL)

Projections subscribe to the event stream and maintain denormalized read tables optimized for specific query patterns.

### Quote Comparison Projection

```sql
-- Optimized for the side-by-side comparison view
CREATE TABLE proj_quote_comparison (
    request_id      UUID NOT NULL,
    client_id       UUID NOT NULL,
    client_name     VARCHAR(255),
    line_of_business VARCHAR(50),
    effective_date  DATE,
    carrier_id      UUID NOT NULL,
    carrier_name    VARCHAR(255),
    am_best_rating  VARCHAR(10),
    quote_id        UUID,
    quote_status    VARCHAR(30),
    total_premium   NUMERIC(12,2),
    annual_premium  NUMERIC(12,2),
    deductible      NUMERIC(12,2),
    coverages_json  JSONB,          -- array of coverage objects for easy rendering
    quote_number    VARCHAR(100),
    expires_at      TIMESTAMPTZ,
    gap_warnings    JSONB,          -- coverage gap alerts
    PRIMARY KEY (request_id, carrier_id)
);
CREATE INDEX idx_proj_quote_client ON proj_quote_comparison(client_id);
```

### Active Policies Projection

```sql
CREATE TABLE proj_active_policy (
    policy_id       UUID PRIMARY KEY,
    policy_number   VARCHAR(100) NOT NULL,
    client_id       UUID NOT NULL,
    client_name     VARCHAR(255),
    agency_id       UUID NOT NULL,
    carrier_id      UUID NOT NULL,
    carrier_name    VARCHAR(255),
    line_of_business VARCHAR(50),
    status          VARCHAR(30),
    effective_date  DATE,
    expiration_date DATE,
    total_premium   NUMERIC(12,2),
    days_to_expiration INTEGER,
    is_renewal_initiated BOOLEAN DEFAULT false,
    coverages_json  JSONB,
    prior_policy_id UUID,
    bound_by_name   VARCHAR(255),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_policy_agency ON proj_active_policy(agency_id);
CREATE INDEX idx_proj_policy_expiration ON proj_active_policy(expiration_date);
CREATE INDEX idx_proj_policy_client ON proj_active_policy(client_id);
```

### Commission Dashboard Projection

```sql
CREATE TABLE proj_commission_summary (
    agency_id       UUID NOT NULL,
    producer_id     UUID,
    carrier_id      UUID NOT NULL,
    month           DATE NOT NULL,      -- first of month
    expected_total  NUMERIC(14,2) DEFAULT 0,
    received_total  NUMERIC(14,2) DEFAULT 0,
    variance_total  NUMERIC(14,2) DEFAULT 0,
    dispute_count   INTEGER DEFAULT 0,
    policy_count    INTEGER DEFAULT 0,
    PRIMARY KEY (agency_id, carrier_id, month, producer_id)
);
CREATE INDEX idx_proj_commission_producer ON proj_commission_summary(producer_id);

CREATE TABLE proj_commission_detail (
    transaction_id  UUID PRIMARY KEY,
    policy_id       UUID NOT NULL,
    policy_number   VARCHAR(100),
    client_name     VARCHAR(255),
    carrier_name    VARCHAR(255),
    producer_name   VARCHAR(255),
    gross_commission NUMERIC(12,2),
    producer_split  NUMERIC(12,2),
    status          VARCHAR(20),
    statement_date  DATE,
    payment_date    DATE,
    variance        NUMERIC(12,2)
);
```

### Renewal Pipeline Projection

```sql
CREATE TABLE proj_renewal_pipeline (
    policy_id       UUID PRIMARY KEY,
    policy_number   VARCHAR(100),
    client_id       UUID NOT NULL,
    client_name     VARCHAR(255),
    carrier_name    VARCHAR(255),
    line_of_business VARCHAR(50),
    expiration_date DATE NOT NULL,
    days_to_expiration INTEGER,
    current_premium NUMERIC(12,2),
    retention_risk_score NUMERIC(4,3),
    renewal_status  VARCHAR(30) DEFAULT 'upcoming',  -- upcoming, in_progress, quoted, bound, lost
    assigned_to     VARCHAR(255),
    new_request_id  UUID,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_renewal_expiration ON proj_renewal_pipeline(expiration_date);
CREATE INDEX idx_proj_renewal_status ON proj_renewal_pipeline(renewal_status);
```

### Agency Dashboard Projection

```sql
CREATE TABLE proj_agency_dashboard (
    agency_id       UUID PRIMARY KEY,
    total_clients   INTEGER DEFAULT 0,
    active_policies INTEGER DEFAULT 0,
    total_premium   NUMERIC(14,2) DEFAULT 0,
    open_quotes     INTEGER DEFAULT 0,
    pending_renewals INTEGER DEFAULT 0,
    mtd_new_business NUMERIC(14,2) DEFAULT 0,
    mtd_commissions NUMERIC(14,2) DEFAULT 0,
    coverage_gaps_detected INTEGER DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Process Managers (Sagas)

Long-running processes that coordinate across aggregates by listening to events and issuing commands.

### Quote-to-Bind Saga

```
Triggered by: QuoteRequestCreated
1. Run AppetitePreScreen -> emit AppetitePreScreenCompleted
2. For each eligible carrier: send CarrierSubmissionSent
3. Wait for CarrierQuoteReceived / CarrierQuoteDeclined (with timeout)
4. When all responses in or timeout reached: emit QuoteRequestCompleted
5. On QuoteSelected -> validate and wait for BindPolicy command
6. On PolicyBound -> calculate and emit CommissionExpected
7. Timeout: if no response from carrier in 48h, mark as expired
```

### Renewal Management Saga

```
Triggered by: policy expiration_date - 90 days (scheduled event)
1. Emit RenewalMarketingInitiated
2. Create new QuoteRequest with prior policy data
3. Monitor QuoteRequestCompleted
4. If no bind within 30 days of expiration: escalate alert
5. On PolicyRenewed or PolicyNonRenewed: complete saga
```

## Trade-offs

**Strengths:**
- Complete audit trail is inherent -- every state change is an immutable event, satisfying DOI and E&O audit requirements without separate audit tables
- Natural fit for the asynchronous, multi-carrier quote/response workflow
- Temporal queries ("what was the policy state on March 15th?") are trivial
- Read projections can be rebuilt from scratch if requirements change
- Supports real-time event-driven integrations (webhooks, CRM sync) natively
- Each projection is independently scalable and optimizable

**Weaknesses:**
- Significantly higher implementation complexity vs. CRUD; requires event store infrastructure, projection managers, and saga orchestration
- Eventual consistency between write and read sides can confuse users ("I just bound the policy, why doesn't it show?")
- Event schema evolution requires careful versioning (upcasting old events to new schemas)
- Debugging production issues requires replaying event streams rather than querying current state
- Team must understand DDD, event sourcing, and CQRS patterns -- steep learning curve
- Snapshot management needed for aggregates with long histories (policies with many endorsements)

## Scalability Considerations

- Event store partitioned by `aggregate_type` and `created_at` for write throughput
- Projections can run on separate database instances or even different database engines (e.g., Elasticsearch for full-text search)
- Event bus (Kafka, RabbitMQ, or PostgreSQL LISTEN/NOTIFY for smaller deployments) decouples write and read scaling
- Snapshots reduce replay time for frequently-modified aggregates
- Projections can be parallelized by aggregate type

## Migration Path

The event-sourced model can be adopted incrementally. Start with a traditional CRUD model (Suggestion 1) and introduce event sourcing for the highest-value aggregates first -- QuoteRequest and Policy -- where the audit trail and async workflow benefits are most pronounced. Other aggregates (Client, Commission) can remain CRUD-based initially and migrate when the team is comfortable with the pattern. The normalized tables from Suggestion 1 can serve as read projections during the transition.
