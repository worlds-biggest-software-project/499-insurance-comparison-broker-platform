# Data Model Suggestion 4: Graph Database Model (Neo4j)

## Overview

A property graph model using Neo4j, where entities are **nodes** with labels and properties, and relationships are **first-class edges** with their own properties. This approach treats the insurance domain as what it fundamentally is: a densely connected network of parties, risks, products, coverages, and financial relationships where the connections between entities carry as much meaning as the entities themselves.

## Why a Graph Database Suits This Domain

Insurance comparison and brokerage is a relationship-intensive domain. Consider the core question an agent asks: "Which carriers will write this risk, at what price, with what coverages, given this client's loss history and this agency's appointment status?" Answering that question in a relational database requires joining 6-8 tables. In a graph, it is a single traversal.

Key graph-native advantages for this domain:

1. **Carrier appetite matching is a graph traversal problem.** A risk has properties (NAICS code, state, revenue, loss history). A carrier has appetite rules that form a subgraph of acceptable risk characteristics. Matching risks to carriers is pattern matching on connected subgraphs -- exactly what graph databases optimize for.

2. **Coverage comparison across carriers is relationship analysis.** When comparing quotes, agents need to understand not just premium differences but coverage equivalence -- which coverages map to which, what endorsements fill gaps, how sub-limits compare. These are edge-traversal queries.

3. **Commission structures form hierarchical networks.** Override commissions, volume bonuses, and split arrangements between agencies, MGAs, and carriers create multi-level relationship trees that graph databases handle naturally.

4. **Client-risk-policy-carrier networks support cross-selling and retention.** Graph queries like "find all clients with a homeowner policy but no umbrella" or "which clients share a carrier that just had a rate increase" are trivial traversals but complex SQL.

5. **ACORD's own information model is fundamentally an entity-relationship graph.** The ACORD reference architecture defines Party, Agreement, Product, Coverage, and Claim as interconnected concepts -- a graph model is the most faithful representation.

## Node Definitions

### Party Nodes

```cypher
// Agency
CREATE (a:Agency:Party {
  agency_id: randomUUID(),
  name: "Smith Insurance Group",
  legal_name: "Smith Insurance Group LLC",
  tax_id: "82-1234567",
  license_number: "AG-2024-12345",
  license_state: "TX",
  address_line1: "100 Main St",
  city: "Austin",
  state: "TX",
  zip_code: "78701",
  phone: "512-555-0100",
  email: "info@smithinsurance.com",
  website: "https://smithinsurance.com",
  created_at: datetime(),
  updated_at: datetime()
})

// Agency User
CREATE (u:User:Party {
  user_id: randomUUID(),
  email: "jsmith@smithinsurance.com",
  first_name: "John",
  last_name: "Smith",
  role: "producer",
  license_number: "PR-2024-67890",
  license_state: "TX",
  is_active: true,
  created_at: datetime()
})

// Client (Individual)
CREATE (c:Client:Party {
  client_id: randomUUID(),
  client_type: "individual",
  display_name: "Martinez, Carlos",
  first_name: "Carlos",
  last_name: "Martinez",
  date_of_birth: date("1985-03-15"),
  email: "carlos@email.com",
  phone: "512-555-0200",
  state: "TX",
  zip_code: "78704",
  source_channel: "portal",
  created_at: datetime()
})

// Client (Business)
CREATE (b:Client:Party {
  client_id: randomUUID(),
  client_type: "business",
  display_name: "Lone Star Restaurants LLC",
  business_name: "Lone Star Restaurants LLC",
  naics_code: "722511",
  entity_type: "llc",
  years_in_business: 8,
  annual_revenue: 2500000,
  employee_count: 45,
  state: "TX",
  zip_code: "78701",
  created_at: datetime()
})

// Carrier
CREATE (cr:Carrier:Party {
  carrier_id: randomUUID(),
  name: "Texas Mutual Insurance",
  am_best_rating: "A",
  naic_code: "22945",
  api_endpoint: "https://api.texasmutual.com/v2",
  api_type: "rest",
  is_active: true,
  created_at: datetime()
})
```

### Risk Nodes

```cypher
// Auto Risk
CREATE (r:Risk:AutoRisk {
  risk_id: randomUUID(),
  risk_type: "auto",
  year: 2024,
  make: "Toyota",
  model: "Camry",
  vin: "4T1B11HK5RU123456",
  usage: "commute",
  annual_mileage: 12000,
  garaging_state: "TX",
  garaging_zip: "78704",
  created_at: datetime()
})

// Property Risk
CREATE (r:Risk:PropertyRisk {
  risk_id: randomUUID(),
  risk_type: "homeowner",
  address_line1: "456 Oak Lane",
  city: "Austin",
  state: "TX",
  zip_code: "78704",
  year_built: 2010,
  construction_type: "masonry",
  square_footage: 2800,
  stories: 2,
  roof_type: "composition_shingle",
  roof_year: 2020,
  replacement_cost: 520000,
  protection_class: 3,
  created_at: datetime()
})

// Commercial Risk
CREATE (r:Risk:CommercialRisk {
  risk_id: randomUUID(),
  risk_type: "general_liability",
  description: "Full-service restaurant operations",
  naics_code: "722511",
  state: "TX",
  zip_code: "78701",
  annual_revenue: 2500000,
  employee_count: 45,
  years_in_business: 8,
  created_at: datetime()
})

// Workers Comp Class Code (reusable reference node)
CREATE (cc:ClassCode {
  code: "9082",
  description: "Restaurant NOC",
  state: "TX",
  base_rate: 2.45,
  effective_date: date("2026-01-01")
})
```

### Insurance Product & Coverage Nodes

```cypher
// Carrier Product
CREATE (p:Product {
  product_id: randomUUID(),
  product_name: "BusinessOwners Plus",
  line_of_business: "bop",
  product_code: "BOP-2026",
  is_bindable_online: true,
  created_at: datetime()
})

// Coverage Type (reference/catalog node)
CREATE (ct:CoverageType {
  coverage_code: "GL-PREM-OPS",
  coverage_name: "Premises & Operations Liability",
  line_of_business: "general_liability",
  is_standard: true,
  acord_coverage_code: "PREM"
})

// Appetite Rule
CREATE (ar:AppetiteRule {
  rule_id: randomUUID(),
  line_of_business: "general_liability",
  state: "TX",
  min_premium: 1000,
  max_premium: 250000,
  max_loss_ratio: 0.65,
  min_years_in_business: 2,
  is_active: true,
  effective_date: date("2026-01-01"),
  notes: "Package with property preferred"
})
```

### Quote & Policy Nodes

```cypher
// Quote Request
CREATE (qr:QuoteRequest {
  request_id: randomUUID(),
  line_of_business: "bop",
  effective_date: date("2026-07-01"),
  expiration_date: date("2027-07-01"),
  status: "completed",
  source_channel: "agent",
  created_at: datetime()
})

// Individual Carrier Quote
CREATE (q:Quote {
  quote_id: randomUUID(),
  quote_number: "TM-BOP-2026-78901",
  status: "returned",
  total_premium: 8750.00,
  annual_premium: 8750.00,
  term_months: 12,
  deductible: 1000.00,
  quoted_at: datetime(),
  expires_at: datetime("2026-07-15T23:59:59Z"),
  created_at: datetime()
})

// Coverage Instance (on a quote or policy)
CREATE (cov:Coverage {
  coverage_id: randomUUID(),
  coverage_code: "GL-PREM-OPS",
  coverage_name: "Premises & Operations Liability",
  coverage_limit: 1000000,
  aggregate_limit: 2000000,
  deductible: 0,
  premium: 4200.00,
  is_included: true
})

// Policy
CREATE (pol:Policy {
  policy_id: randomUUID(),
  policy_number: "TM-BOP-2026-78901-01",
  line_of_business: "bop",
  status: "active",
  effective_date: date("2026-07-01"),
  expiration_date: date("2027-07-01"),
  total_premium: 8750.00,
  bound_at: datetime(),
  created_at: datetime()
})
```

### Document & AI Nodes

```cypher
CREATE (d:Document {
  document_id: randomUUID(),
  document_type: "application",
  file_name: "acord125_lonestsar.pdf",
  file_path: "/docs/2026/07/acord125_lonestar.pdf",
  mime_type: "application/pdf",
  acord_form_number: "125",
  created_at: datetime()
})

CREATE (ex:Extraction {
  extraction_id: randomUUID(),
  model_version: "gpt-4o-2026-05",
  extraction_type: "acord_form",
  overall_confidence: 0.94,
  status: "completed",
  created_at: datetime()
})

CREATE (ef:ExtractedField {
  field_name: "BusinessName",
  field_value: "Lone Star Restaurants LLC",
  confidence: 0.98,
  acord_field_ref: "InsuredName",
  page_number: 1
})
```

## Relationship Definitions

### Party Relationships

```cypher
// Agency structure
(user)-[:WORKS_FOR {role: "producer", since: date("2020-01-15")}]->(agency)
(client)-[:INSURED_BY]->(agency)
(client)-[:ASSIGNED_TO {role: "producer"}]->(user)
(client)-[:SERVICED_BY {role: "csr"}]->(user)
(agency)-[:APPOINTED_WITH {
  effective_date: date("2022-06-01"),
  lines: ["commercial_property","general_liability","bop","workers_comp"],
  commission_level: "preferred"
}]->(carrier)
```

### Risk Relationships

```cypher
// Client owns risks
(client)-[:HAS_RISK]->(risk)

// Risk locations
(risk)-[:LOCATED_IN]->(location:Location {state: "TX", zip: "78701"})

// Commercial risk classification
(risk:CommercialRisk)-[:CLASSIFIED_AS {payroll: 180000}]->(classCode:ClassCode)

// Drivers on auto risks
(driver:Driver {name: "Carlos Martinez", dob: date("1985-03-15"),
  license: "TX12345678"})-[:DRIVES]->(autoRisk)

// Additional insured relationships
(additionalInsured:Party)-[:ADDITIONAL_INSURED_ON {
  interest: "landlord", certificate_required: true
}]->(risk)
```

### Quoting Relationships

```cypher
// Quote request structure
(user)-[:REQUESTED {at: datetime()}]->(quoteRequest)
(quoteRequest)-[:FOR_CLIENT]->(client)
(quoteRequest)-[:INCLUDES_RISK]->(risk)
(quoteRequest)-[:SUBMITTED_TO]->(carrier)

// Carrier quotes
(quote)-[:RESPONDS_TO]->(quoteRequest)
(quote)-[:FROM_CARRIER]->(carrier)
(quote)-[:USES_PRODUCT]->(product)
(quote)-[:INCLUDES_COVERAGE]->(coverage)
(coverage)-[:COVERS]->(risk)

// Quote comparison edges
(quote)-[:COMPARED_WITH {premium_diff: -450.00, coverage_score: 0.92}]->(otherQuote)

// Appetite matching
(carrier)-[:HAS_APPETITE]->(appetiteRule)
(appetiteRule)-[:ACCEPTS_NAICS]->(naicsCode:NAICSCode {code: "722511"})
(appetiteRule)-[:COVERS_STATE]->(state:State {code: "TX"})
(risk)-[:MATCHES_APPETITE {score: 0.87, factors: ["naics","state","revenue"]}]->(appetiteRule)
```

### Policy Relationships

```cypher
// Policy binding
(policy)-[:BOUND_FROM]->(quote)
(policy)-[:INSURES]->(client)
(policy)-[:ISSUED_BY]->(carrier)
(policy)-[:MANAGED_BY]->(agency)
(policy)-[:BOUND_BY]->(user)
(policy)-[:COVERS_RISK]->(risk)
(policy)-[:HAS_COVERAGE]->(coverage)

// Policy lifecycle
(newPolicy)-[:RENEWS]->(priorPolicy)
(endorsement:Endorsement {
  type: "add_location", effective_date: date("2026-09-01"),
  premium_change: 1200.00
})-[:MODIFIES]->(policy)

// Cancellation
(cancellation:Cancellation {
  reason: "non_payment", effective_date: date("2026-10-15"),
  return_premium: 3500.00
})-[:TERMINATES]->(policy)
```

### Commission Relationships

```cypher
// Commission structure
(carrier)-[:PAYS_COMMISSION {
  type: "new_business", rate: 15.0,
  line: "bop", effective_date: date("2026-01-01")
}]->(agency)

(commissionTxn:CommissionTransaction {
  transaction_id: randomUUID(),
  gross_amount: 1312.50,
  status: "received"
})-[:EARNED_ON]->(policy)

(commissionTxn)-[:PAID_TO {split_percent: 60.0, amount: 787.50}]->(producer:User)
(commissionTxn)-[:RETAINED_BY {split_percent: 40.0, amount: 525.00}]->(agency)

// Override / hierarchical commissions
(mga:Agency)-[:OVERRIDE_FROM {rate: 2.0}]->(carrier)
(mga)-[:PAYS_OVERRIDE {rate: 2.0}]->(subAgency:Agency)
```

### Document & AI Relationships

```cypher
(document)-[:BELONGS_TO]->(client)
(document)-[:ATTACHED_TO]->(policy)
(document)-[:UPLOADED_BY]->(user)

(extraction)-[:EXTRACTED_FROM]->(document)
(extractedField)-[:PART_OF]->(extraction)
(extractedField)-[:MAPS_TO_FIELD {acord_ref: "InsuredName"}]->(risk)

// AI coverage gap detection
(gapAlert:CoverageGap {
  gap_type: "missing_coverage", severity: "high",
  description: "No liquor liability for restaurant with bar revenue"
})-[:DETECTED_IN]->(quote)
(gapAlert)-[:RECOMMENDS]->(coverageType:CoverageType)
```

## Key Graph Queries

### Appetite Matching: "Which carriers will write this risk?"

```cypher
MATCH (client:Client {client_id: $clientId})-[:HAS_RISK]->(risk:CommercialRisk)
MATCH (carrier:Carrier)-[:HAS_APPETITE]->(rule:AppetiteRule)
WHERE rule.line_of_business = risk.risk_type
  AND rule.state = risk.state
  AND rule.is_active = true
  AND (rule.min_years_in_business IS NULL
       OR client.years_in_business >= rule.min_years_in_business)
  AND (rule.max_premium IS NULL
       OR risk.estimated_premium <= rule.max_premium)
RETURN carrier.name, carrier.am_best_rating, rule,
       CASE WHEN client.years_in_business >= 5 THEN 'preferred' ELSE 'standard' END AS tier
ORDER BY carrier.am_best_rating ASC
```

### Cross-Sell: "Clients with homeowner but no umbrella"

```cypher
MATCH (client:Client)-[:INSURED_BY]->(agency:Agency {agency_id: $agencyId})
MATCH (client)-[:HAS_RISK]->(:Risk {risk_type: "homeowner"})
WHERE NOT EXISTS {
  MATCH (client)-[:HAS_RISK]->(:Risk {risk_type: "umbrella"})
}
RETURN client.display_name, client.email, client.phone
ORDER BY client.display_name
```

### Renewal Pipeline with Loss History

```cypher
MATCH (policy:Policy {status: "active"})-[:MANAGED_BY]->(agency:Agency {agency_id: $agencyId})
WHERE policy.expiration_date <= date() + duration({days: 90})
MATCH (policy)-[:INSURES]->(client:Client)
MATCH (policy)-[:ISSUED_BY]->(carrier:Carrier)
OPTIONAL MATCH (policy)<-[:RENEWS]-(priorPolicy:Policy)
OPTIONAL MATCH (policy)-[:COVERS_RISK]->(risk:Risk)
RETURN policy.policy_number, client.display_name, carrier.name,
       policy.expiration_date, policy.total_premium,
       duration.between(date(), policy.expiration_date).days AS days_remaining,
       COUNT(priorPolicy) AS renewal_count
ORDER BY policy.expiration_date ASC
```

### Commission Network: "Show me the money flow"

```cypher
MATCH path = (carrier:Carrier)-[c1:PAYS_COMMISSION]->(agency:Agency)
              <-[:WORKS_FOR]-(producer:User)
MATCH (txn:CommissionTransaction)-[:EARNED_ON]->(policy:Policy)-[:ISSUED_BY]->(carrier)
MATCH (txn)-[split:PAID_TO]->(producer)
WHERE policy.effective_date >= date("2026-01-01")
RETURN carrier.name, policy.policy_number, txn.gross_amount,
       split.amount AS producer_payout, split.split_percent
ORDER BY txn.gross_amount DESC
```

### Coverage Gap Analysis Across Portfolio

```cypher
MATCH (agency:Agency {agency_id: $agencyId})<-[:INSURED_BY]-(client:Client)
MATCH (client)-[:HAS_RISK]->(risk:Risk)
MATCH (gap:CoverageGap)-[:DETECTED_IN]->(quote:Quote)-[:RESPONDS_TO]->
      (qr:QuoteRequest)-[:FOR_CLIENT]->(client)
WHERE gap.severity = "high"
RETURN client.display_name, risk.risk_type, gap.description,
       gap.gap_type, quote.carrier_name
ORDER BY client.display_name
```

## Indexes and Constraints

```cypher
// Uniqueness constraints (also create indexes)
CREATE CONSTRAINT agency_id_unique FOR (a:Agency) REQUIRE a.agency_id IS UNIQUE;
CREATE CONSTRAINT user_id_unique FOR (u:User) REQUIRE u.user_id IS UNIQUE;
CREATE CONSTRAINT user_email_unique FOR (u:User) REQUIRE u.email IS UNIQUE;
CREATE CONSTRAINT client_id_unique FOR (c:Client) REQUIRE c.client_id IS UNIQUE;
CREATE CONSTRAINT carrier_id_unique FOR (cr:Carrier) REQUIRE cr.carrier_id IS UNIQUE;
CREATE CONSTRAINT risk_id_unique FOR (r:Risk) REQUIRE r.risk_id IS UNIQUE;
CREATE CONSTRAINT quote_request_id_unique FOR (qr:QuoteRequest) REQUIRE qr.request_id IS UNIQUE;
CREATE CONSTRAINT quote_id_unique FOR (q:Quote) REQUIRE q.quote_id IS UNIQUE;
CREATE CONSTRAINT policy_id_unique FOR (p:Policy) REQUIRE p.policy_id IS UNIQUE;
CREATE CONSTRAINT policy_number_unique FOR (p:Policy) REQUIRE p.policy_number IS UNIQUE;
CREATE CONSTRAINT document_id_unique FOR (d:Document) REQUIRE d.document_id IS UNIQUE;

// Performance indexes
CREATE INDEX client_state FOR (c:Client) ON (c.state);
CREATE INDEX client_naics FOR (c:Client) ON (c.naics_code);
CREATE INDEX risk_type FOR (r:Risk) ON (r.risk_type);
CREATE INDEX risk_state FOR (r:Risk) ON (r.state);
CREATE INDEX policy_status FOR (p:Policy) ON (p.status);
CREATE INDEX policy_expiration FOR (p:Policy) ON (p.expiration_date);
CREATE INDEX quote_status FOR (q:Quote) ON (q.status);
CREATE INDEX appetite_lob FOR (ar:AppetiteRule) ON (ar.line_of_business, ar.state);

// Full-text index for search
CREATE FULLTEXT INDEX client_search FOR (c:Client) ON EACH [c.display_name, c.business_name, c.email];
```

## Trade-offs

**Strengths:**
- Carrier appetite matching, cross-sell identification, and coverage gap analysis are natural graph traversals -- orders of magnitude faster than equivalent SQL joins on large datasets
- The ACORD information model (Party-Agreement-Product-Coverage) maps directly to a property graph
- Commission hierarchies with overrides, splits, and volume tiers are elegantly represented as edge properties on relationship chains
- Schema-flexible nodes handle the polymorphism of risk types (auto vs. property vs. commercial) without EAV or JSONB
- Relationship-first queries ("find all clients connected to this carrier through any policy") are trivial
- Visual graph exploration tools (Neo4j Bloom) provide powerful ad-hoc investigation for agents and principals

**Weaknesses:**
- Aggregate calculations (monthly premium totals, commission summaries) are not graph-native; require either Cypher aggregation or a complementary analytical store
- Transaction handling is less mature than PostgreSQL; complex multi-node ACID transactions need care
- Smaller ecosystem of ORMs, migration tools, and reporting integrations compared to SQL
- Team must learn Cypher query language and graph modeling patterns
- Operational complexity: Neo4j clustering, backup, and monitoring tooling is less standardized than PostgreSQL
- Premium accounting and financial reconciliation queries are naturally tabular, not graph-shaped -- a relational sidecar may be needed
- Licensing: Neo4j Enterprise (needed for clustering) has commercial licensing; the Community edition has limitations

## Scalability Considerations

- Neo4j scales reads via read replicas in a causal cluster
- Write scaling requires sharding by agency or geographic region (Neo4j Fabric)
- For financial reporting, consider a PostgreSQL sidecar for commission accounting and premium summaries -- sync via change data capture from Neo4j
- Graph queries scale with the size of the traversed subgraph, not the total database size, making per-agency queries fast even at platform scale
- Consider Neo4j Aura (managed cloud) to reduce operational burden

## Migration Path

A pragmatic deployment pairs Neo4j for the relationship-intensive operations (appetite matching, quote comparison, cross-sell, coverage gap detection) with PostgreSQL for transactional accounting (commissions, premium payments, audit logs). Start with PostgreSQL as the system of record (Suggestion 1 or 3) and introduce Neo4j as a specialized query engine, synchronized via change data capture. This polyglot approach captures graph advantages where they matter most while keeping financial data in the relational model where it belongs. Over time, if the graph model proves dominant in query patterns, more of the read workload can shift to Neo4j while PostgreSQL remains the transactional backbone.
