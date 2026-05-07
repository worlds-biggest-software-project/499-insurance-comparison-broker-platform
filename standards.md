# Standards & API Reference

> Project: Insurance Comparison & Broker Platform · Generated: 2026-05-07

## Industry Standards & Specifications

### ACORD Standards

- **ACORD XML Standards** — The primary data exchange format for the P&C insurance industry. Defines XML schemas for policy, claims, billing, and carrier communication. Used by virtually all insurance software for carrier connectivity.
  - URL: https://acord.org/standards-architecture/acord-data-standards

- **ACORD AL3 (Agency/Company Linked 3)** — Legacy EDI-based standard for data exchange between agencies and carriers. Still in widespread use for IVANS Download transactions and batch policy data exchange.
  - URL: https://acord.org/standards-architecture/acord-data-standards

- **ACORD Next-Generation Digital Standards (NGDS)** — Modern JSON and YAML data-interchange formats built on microservices architecture and RESTful API specifications. Designed to replace XML-based standards for new digital-first implementations. Includes standardised resource definitions for Life & Annuities with P&C expansion underway.
  - URL: https://www.acord.org/docs/default-source/events/virtual-gift-bag/acord-next-gen-digital-standards-introduction-training-manual.pdf

- **ACORD GRLC Generation 2.0** — Updated data standards for the Global Reinsurance and Large Commercial segment. Introduces digital-first messaging supporting straight-through, end-to-end processing across the reinsurance policy lifecycle. Available in both XML and JSON formats.
  - URL: https://acord.org/standards-architecture/acord-data-standards/Global_Reinsurance_Data_Standards

- **ACORD Reference Architecture** — Comprehensive framework including business processes, product models, development frameworks, information models, data models, and capability models for insurance industry applications.
  - URL: https://www.acord.org/standards-architecture/reference-architecture

- **ACORD Forms** — Standardised paper and electronic forms (applications, certificates, endorsements) used across the insurance industry. Over 800 forms covering P&C, Life, and Surety lines.
  - URL: https://acord.org/standards-architecture/acord-data-standards

### SEMCI Standard

- **SEMCI (Single Entry Multiple Company Interface)** — Industry standard based on service-oriented architecture (SOA) enabling agents to submit risk data once and receive quotes from multiple carriers. Uses ACORD Web Transaction Interface (WTI) and XML Insurance and Surety Service Specification. Communication protocol is HTTP with ACORD XML payloads.
  - URL: https://en.wikipedia.org/wiki/SEMCI
  - Patent: US 7,900,138 B2 (Real-Time SEMCI) — https://patents.google.com/patent/US7900138B2/en

### IVANS Standards

- **IVANS Download** — Industry standard for automated carrier-to-agency data exchange. Supports policy declarations, claims data, commission statements, and document delivery. Uses AL3 format over the IVANS Exchange network.
  - URL: https://www.ivans.com/for-agents/products/ivans-download/

- **IVANS File Transfer API** — RESTful API for sending and receiving download data through IVANS Exchange. Carriers send data mapped to IVANS standard translations; AMS platforms retrieve data via API or Transfer Manager Client.
  - URL: https://api.ivans.com/docs/ivans-exchange

### W3C & IETF Standards

- **RFC 7231 — HTTP/1.1 Semantics and Content** — Defines HTTP methods, status codes, and content negotiation used by all RESTful insurance APIs.
  - URL: https://datatracker.ietf.org/doc/html/rfc7231

- **RFC 8259 — JSON (JavaScript Object Notation)** — The primary data interchange format for modern insurance APIs and ACORD NGDS.
  - URL: https://datatracker.ietf.org/doc/html/rfc8259

- **RFC 6749 — OAuth 2.0 Authorization Framework** — Standard for delegated authorization used by insurance platform APIs (Applied, Vertafore, CoverForce, etc.).
  - URL: https://datatracker.ietf.org/doc/html/rfc6749

- **RFC 7519 — JSON Web Token (JWT)** — Token format commonly used for API authentication in insurance platforms.
  - URL: https://datatracker.ietf.org/doc/html/rfc7519

- **OpenID Connect Core 1.0** — Identity layer on top of OAuth 2.0 for user authentication. Relevant for SSO across agent portals, carrier systems, and consumer-facing applications.
  - URL: https://openid.net/specs/openid-connect-core-1_0.html

- **W3C XML Schema** — Foundation for ACORD XML data standards. Defines the structure and validation rules for ACORD XML messages.
  - URL: https://www.w3.org/XML/Schema

### Data Model & API Specifications

- **OpenAPI Specification 3.1** — Standard for describing RESTful APIs. Used by modern insurance platforms (Applied Dev Center, Vertafore Adapt API, CoverForce) for API documentation and client generation.
  - URL: https://www.openapis.org/

- **JSON Schema** — Vocabulary for annotating and validating JSON documents. Foundation for ACORD NGDS data definitions and request/response validation in insurance APIs.
  - URL: https://json-schema.org/

- **AsyncAPI Specification** — Standard for describing event-driven APIs. Relevant for webhook-based notifications (quote events, bind confirmations, policy changes) used by Bold Penguin, CoverForce, and others.
  - URL: https://www.asyncapi.com/

- **NAICS (North American Industry Classification System)** — Standard 6-digit codes used for risk classification in commercial insurance quoting. Used by Tarmika, Ask Kodiak, and Bold Penguin for carrier appetite matching.
  - URL: https://www.census.gov/naics/

- **FIBO / FIB-DM (Financial Industry Business Data Model)** — Open-source industry standard for financial concepts and relationships. The most extensive data model blueprint for financial institutions, with 3,212 normative entities. Potentially applicable for insurance financial reporting and regulatory data.
  - URL: https://spec.edmcouncil.org/fibo/

### Security & Authentication Standards

- **PCI DSS v4.0 (Payment Card Industry Data Security Standard)** — Mandatory for any insurance platform collecting premium payments via credit/debit card. Defines security requirements for storing, processing, and transmitting cardholder data. Insurance platforms typically achieve compliance through payment gateway partnerships (tokenisation, descoping).
  - URL: https://www.pcisecuritystandards.org/

- **NIST Cybersecurity Framework 2.0** — Comprehensive framework for managing cybersecurity risk. Recommended baseline for insurance platforms handling sensitive personal and financial data.
  - URL: https://www.nist.gov/cyberframework

- **NAIC Insurance Data Security Model Law (#668)** — Model regulation requiring insurers and licensed entities to implement information security programmes, conduct risk assessments, and notify commissioners of cybersecurity events within 3 days. Enacted in 22+ states as of 2026.
  - URL: https://content.naic.org/sites/default/files/model-law-668.pdf

- **SOC 2 Type II** — Service Organisation Control audit standard. Expected by insurance carriers and agencies from SaaS platform vendors. Covers security, availability, processing integrity, confidentiality, and privacy.
  - URL: https://us.aicpa.org/interestareas/frc/assuranceadvisoryservices/socforserviceorganizations

- **GLBA (Gramm-Leach-Bliley Act)** — US federal law requiring financial institutions (including insurance entities) to protect consumer financial information. Mandates privacy notices and safeguards for non-public personal information.
  - URL: https://www.ftc.gov/legal-library/browse/statutes/gramm-leach-bliley-act

- **HIPAA (Health Insurance Portability and Accountability Act)** — Applicable when the platform handles health insurance data. Requires safeguards for protected health information (PHI).
  - URL: https://www.hhs.gov/hipaa/

- **TLS 1.2/1.3** — Transport Layer Security for encrypting all API communications. Required by PCI DSS and expected by all carrier API integrations.
  - URL: https://datatracker.ietf.org/doc/html/rfc8446

### Regulatory Standards

- **NAIC Model Laws** — Suite of model regulations governing broker licensing, surplus lines, market conduct, and rate filing. Compliance requirements vary by state and line of business.
  - URL: https://content.naic.org/

- **CCPA / CPRA (California Consumer Privacy Act / California Privacy Rights Act)** — Data privacy obligations covering policyholder personal information. Requires disclosure, deletion, and opt-out rights for California residents.
  - URL: https://oag.ca.gov/privacy/ccpa

- **GDPR (General Data Protection Regulation)** — EU data protection regulation applicable when serving European markets or handling EU citizen data. Requires lawful basis for processing, data minimisation, and right to erasure.
  - URL: https://gdpr.eu/

## Similar Products — Developer Documentation & APIs

### Applied Epic / Applied Systems
- **Description:** Enterprise agency management system with integrated comparative quoting, carrier connectivity, and full policy lifecycle management. The largest AMS platform in the independent agency channel.
- **API Documentation:** https://devcenter.myappliedproducts.com/docs/overview
- **Developer Portal:** https://developer.myappliedproducts.com/documentation
- **SDKs/Libraries:** RESTful APIs (migrating from legacy SDK); JavaScript examples available
- **Developer Guide:** https://devcenter.myappliedproducts.com/docs/overview
- **Standards:** REST/JSON, ACORD XML, IVANS AL3
- **Authentication:** OAuth 2.0

### EZLynx (Applied Systems)
- **Description:** All-in-one comparative rater and agency management system for personal lines agencies, with 330+ carrier integrations and native AI capabilities.
- **API Documentation:** Available through EZLynx platform (partner access required)
- **SDKs/Libraries:** API tools for push/pull operations and high-volume automated quoting
- **Developer Guide:** Via EZLynx Connect Marketplace
- **Standards:** REST/JSON, ACORD XML
- **Authentication:** API key-based

### Vertafore AMS360
- **Description:** Comprehensive agency management system with personal lines comparative rater, document management, and carrier connectivity through the IVANS network.
- **API Documentation:** https://help.vertafore.com/devportal/content/getstarted/gettingstarted.htm
- **Developer Portal:** Vertafore Developer Portal (requires VSSO account)
- **SDKs/Libraries:** WSAPI (WCF-based), Adapt API (RESTful)
- **Developer Guide:** https://help.vertafore.com/devportal/content/howto/tutorial1_headerconnection.htm
- **Standards:** REST/JSON (Adapt API), WCF/SOAP (legacy WSAPI), ACORD XML
- **Authentication:** OAuth 2.0 (Adapt API), API credentials (WSAPI)

### Bold Penguin
- **Description:** API-first small business commercial insurance quoting platform with embeddable web components and webhook-based event notifications.
- **API Documentation:** https://developers.boldpenguin.com/docs/carrier_integrations/
- **SDKs/Libraries:** https://developers.boldpenguin.com/docs/sdk/sdk_overview/ (Web Components via Stencil.js)
- **Developer Guide:** https://developers.boldpenguin.com/
- **Standards:** REST/JSON, Web Components, ACORD forms
- **Authentication:** API key-based, OAuth for carrier integrations

### CoverForce
- **Description:** API infrastructure platform for commercial insurance quote-and-bind, supporting embedded insurance and multi-carrier quoting across Workers' Comp, GL, BOP, and Cyber.
- **API Documentation:** Available through CoverForce partnership (not publicly documented)
- **SDKs/Libraries:** RESTful APIs for quoting, binding, and policy issuance
- **Developer Guide:** Via partner onboarding
- **Standards:** REST/JSON
- **Authentication:** API key-based

### Tarmika (Applied Systems)
- **Description:** Single-entry commercial lines quoting platform with API-native carrier integrations returning bindable quotes in under 60 seconds.
- **API Documentation:** Available through Applied Systems partnership
- **SDKs/Libraries:** API-based carrier integrations
- **Developer Guide:** Via Applied Systems partner programme
- **Standards:** REST/JSON, ACORD forms, NAICS codes
- **Authentication:** Via Applied Systems credentials

### IVANS (Applied Systems)
- **Description:** Industry network for carrier-to-agency data exchange supporting policy download, claims data, commission statements, and document delivery.
- **API Documentation:** https://api.ivans.com/docs/ivans-exchange
- **SDKs/Libraries:** Transfer Manager Client, File Transfer API
- **Developer Guide:** Via IVANS partner programme
- **Standards:** AL3, ACORD XML, REST API for file transfer
- **Authentication:** API credentials via IVANS Exchange

### Ask Kodiak (IVANS / Applied Systems)
- **Description:** Commercial insurance appetite and eligibility platform matching risks against carrier product data using NAICS codes and over 20,000 data points.
- **API Documentation:** Available through Ask Kodiak partnership
- **SDKs/Libraries:** Open API for appetite queries and product matching
- **Developer Guide:** Via partner onboarding
- **Standards:** REST/JSON, NAICS codes
- **Authentication:** API key-based

### Majesco
- **Description:** Enterprise core insurance platform covering policy administration, billing, claims, and distribution management with a comprehensive API catalogue.
- **API Documentation:** Via Majesco Product Portal (partner/customer access)
- **SDKs/Libraries:** API Management Platform (APIM) with orchestration tools
- **Developer Guide:** API Store with browsable specifications; no-code API framework
- **Standards:** REST/JSON, ACORD-compliant data models
- **Authentication:** OAuth 2.0, API gateway

### Cogitate DigitalEdge
- **Description:** Cloud-native, multi-tenant insurance platform for MGAs and carriers, covering policy, billing, and claims with 60+ pre-integrated ecosystem partners.
- **API Documentation:** Available through Cogitate partnership
- **SDKs/Libraries:** Adaptive API technology, webhooks
- **Developer Guide:** Via partner onboarding
- **Standards:** REST/JSON, microservices architecture on Azure
- **Authentication:** API key-based, OAuth 2.0

## Notes

- **ACORD Membership**: Access to full ACORD standard specifications typically requires ACORD membership. The standards are not freely available in their entirety, which is a significant consideration for open-source implementations.

- **IVANS Network Lock-In**: The IVANS network, now owned by Applied Systems, is the de facto standard for carrier-agency data exchange. Any new platform must either integrate with IVANS or build direct carrier API connections, which is significantly more effort but avoids dependency on a competitor-owned network.

- **SEMCI Patent**: US Patent 7,900,138 covers the Real-Time SEMCI process. While the patent may be nearing expiration (filed 2001, granted 2011), any implementation of comparative rating should review this patent for potential infringement concerns.

- **Emerging API Standards**: The insurance industry is transitioning from XML/SOAP-based integrations to RESTful JSON APIs. ACORD NGDS represents this shift but adoption is still early, particularly in P&C lines. Building on modern standards (OpenAPI 3.1, JSON Schema) positions a new platform well for the industry's direction.

- **State-by-State Regulatory Variation**: Insurance regulation in the US is state-based, creating compliance complexity. Rate filing requirements, surplus lines rules, broker licensing, and data privacy laws vary by jurisdiction. Any platform must account for 50+ regulatory regimes.

- **Open Insurance Initiative**: There is growing interest in "open insurance" modelled after open banking, but no equivalent of PSD2 or Open Banking standards exists yet for insurance. This represents an opportunity for an open-source platform to help define these standards.
