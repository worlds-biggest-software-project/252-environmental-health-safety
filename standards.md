# Standards & API Reference

> Project: Environmental Health & Safety (EHS) · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 45001:2018 — Occupational Health and Safety Management Systems**
- URL: https://www.iso.org/standard/63787.html
- The primary international standard for OHS management systems. EHS platforms must support its requirements for hazard identification, risk assessment, incident investigation, legal compliance, and corrective action management. Forms the compliance backbone of incident management, audit, and CAPA modules.

**ISO 14001:2015 (under revision for ISO 14001:2026) — Environmental Management Systems**
- URL: https://www.iso.org/standard/60857.html
- Governs environmental aspect identification, legal compliance, and environmental improvement programmes. A 2026 revision is in progress to simplify wording while preserving the Plan-Do-Check-Act (PDCA) cycle. EHS software must support environmental permit management, waste tracking, and emissions reporting consistent with this standard.

**ISO 50001:2018 — Energy Management Systems**
- URL: https://www.iso.org/standard/69426.html
- Relevant for EHS platforms extending into sustainability and carbon accounting. Requires systematic tracking of energy sources, consumption, and performance indicators.

**ISO 9001:2015 — Quality Management Systems**
- URL: https://www.iso.org/standard/62085.html
- Shares the same High Level Structure (HLS/Annex SL) as ISO 14001 and ISO 45001, enabling integrated management systems (IMS). EHS platforms serving QHSE programmes must align data models with ISO 9001 corrective action and document control requirements.

**ISO 31000:2018 — Risk Management**
- URL: https://www.iso.org/standard/65694.html
- Provides principles and guidelines for risk management applicable to EHS risk registers, bowtie analysis, and bow-tie diagrams. Risk assessment modules should align terminology and methodology with this standard.

**ISO 45003:2021 — Psychological Health and Safety at Work**
- URL: https://www.iso.org/standard/64283.html
- Growing area of EHS management covering psychosocial risk factors (workload, harassment, burnout). Relevant to incident reporting modules that capture psychological injury or near-miss events.

---

### W3C & IETF Standards

**RFC 9110 — HTTP Semantics**
- URL: https://www.rfc-editor.org/rfc/rfc9110
- The foundational HTTP specification underpinning all REST API communications. EHS platforms expose REST APIs that must conform to HTTP semantics for status codes, request methods, and content negotiation.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- The standard for delegated authorisation used by all major EHS APIs (SafetyCulture, Intelex, Cority) for third-party integrations and enterprise SSO.

**RFC 7519 — JSON Web Tokens (JWT)**
- URL: https://www.rfc-editor.org/rfc/rfc7519
- Token format used for API authentication across EHS platforms. Intelex v6 API uses OAuth 2.0 token authentication returning JWTs.

**RFC 4180 — Common Format for CSV Files**
- URL: https://www.rfc-editor.org/rfc/rfc4180
- Relevant for OSHA ITA CSV bulk submissions. OSHA's Injury Tracking Application accepts CSV uploads of 300A data for multi-establishment employers.

**W3C Linked Data / JSON-LD**
- URL: https://www.w3.org/TR/json-ld/
- Emerging relevance for EHS data interoperability. JSON-LD provides a mechanism for expressing structured EHS event data in a standards-linked format that could underpin a future open EHS data exchange standard.

---

### Data Model & API Specifications

**OpenAPI Specification 3.x**
- URL: https://spec.openapis.org/oas/latest.html
- The de-facto standard for describing REST APIs. SafetyCulture publishes its Public API as an OpenAPI spec (https://github.com/SafetyCulture/api-json-schemas). Intelex and Cority also expose REST APIs. A new AI-native EHS platform should publish an OpenAPI 3.1 spec from day one.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- Used for validating EHS data payloads. SafetyCulture's GitHub repository (`SafetyCulture/api-json-schemas`) publishes JSON Schemas for all API request and response objects.

**GHS — UN Globally Harmonised System of Classification and Labelling of Chemicals**
- URL: https://unece.org/transport/standards/transport/dangerous-goods/ghs-rev10-2023
- Defines the 16-section SDS format, hazard classification categories, and signal words. EHS chemical management modules must model SDS data and hazard labels according to GHS Rev.10 (2023). OSHA HazCom 2012 (29 CFR 1910.1200) incorporates GHS for the US market.

**GHG Protocol Corporate Standard**
- URL: https://ghgprotocol.org/corporate-standard
- The de-facto standard for corporate greenhouse gas accounting (Scope 1, 2, 3 emissions). ESG modules in EHS platforms (Cority, Benchmark Gensuite, VelocityEHS) use GHG Protocol methodology for emissions calculation and reporting.

**ESRS (European Sustainability Reporting Standards) / CSRD**
- URL: https://www.efrag.org/en/projects/esrs-mandatory-application
- Mandatory reporting framework for companies in scope of the EU Corporate Sustainability Reporting Directive (CSRD). Requires "double materiality" reporting: both financial risk from ESG factors and external environmental/social impacts. EHS platforms targeting European enterprise customers must support ESRS data collection and mapping.

**IFRS S1 / S2 (ISSB — International Sustainability Standards Board)**
- URL: https://www.ifrs.org/issued-standards/ifrs-sustainability-disclosure-standards/
- Absorbed the TCFD and SASB frameworks; now the global baseline for investor-focused sustainability disclosure. Climate-related risk data (IFRS S2) originates in EHS environmental modules. EHS platforms should map collected environmental data to ISSB reporting requirements.

**GRI Standards (Global Reporting Initiative)**
- URL: https://www.globalreporting.org/standards/
- The most widely used voluntary sustainability reporting framework. Safety and environmental KPIs from EHS platforms (e.g., GRI 403 — Occupational Health and Safety; GRI 305 — Emissions) are direct data sources for GRI-aligned sustainability reports.

---

### Regulatory Frameworks

**OSHA 29 CFR 1904 — Recordkeeping and Reporting Occupational Injuries and Illnesses**
- URL: https://www.osha.gov/recordkeeping/
- Requires US employers to maintain OSHA 300 Logs, 301 Incident Reports, and 300A Annual Summaries. Establishments in designated high-hazard industries with 100+ employees must electronically submit 300A data via the OSHA Injury Tracking Application (ITA). OSHA provides a REST API for programmatic ITA submission; EHS platforms must support automated submission to this API.

**OSHA ITA API — Injury Tracking Application**
- URL: https://www.osha.gov/injuryreporting/
- OSHA provides three submission methods: web form, CSV upload, and REST API. EHS platforms can automate annual submission directly via the ITA API, eliminating manual data entry. ITA Preview Environment available for integration testing.

**EPA EPCRA / CERCLA — Emergency Planning and Community Right-to-Know Act**
- URL: https://www.epa.gov/epcra
- Requires Tier II chemical inventory reporting for hazardous chemicals above threshold quantities. EHS chemical management modules must track chemical quantities against EPCRA thresholds and support Tier II report generation.

**EU REACH (Registration, Evaluation, Authorisation and Restriction of Chemicals)**
- URL: https://echa.europa.eu/regulations/reach/understanding-reach
- European chemical regulation requiring substance registration, SVHC (Substances of Very High Concern) tracking, and SDS updates. Chemical management modules must track REACH compliance for substances in use.

**EU RoHS (Restriction of Hazardous Substances)**
- URL: https://ec.europa.eu/environment/topics/waste-and-recycling/rohs-directive_en
- Restricts hazardous substances in electrical and electronic equipment. Relevant to product stewardship modules in EHS platforms serving electronics manufacturers.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) / OpenID Connect (OIDC)**
- URL: https://openid.net/connect/
- OAuth 2.0 is the authentication mechanism used by SafetyCulture, Intelex, and Cority APIs for third-party integrations. OIDC extends OAuth 2.0 with identity verification. Enterprise EHS deployments require SSO via OIDC, typically integrating with Microsoft Entra ID (Azure AD) or Okta.

**OWASP Top 10 (2021)**
- URL: https://owasp.org/www-project-top-ten/
- Security baseline for EHS web applications. Critical given that EHS platforms store sensitive incident data, occupational health records, and chemical inventory that may be subject to privacy regulations.

**NIST SP 800-53 — Security and Privacy Controls**
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- Relevant for EHS platforms deployed in US Federal Government or regulated industry (defence, nuclear, utilities) environments.

**GDPR (EU General Data Protection Regulation) / UK GDPR**
- URL: https://gdpr.eu/
- Occupational health records (medical surveillance data, injury records) constitute sensitive personal data under GDPR Article 9. EHS platforms operating in the EU must implement appropriate data subject rights, consent management, data residency controls, and retention policies.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Anthropic's open protocol for connecting AI assistants to external data sources and tools. An AI-native EHS platform should expose an MCP server to allow LLM-based agents to query incident records, risk registers, SDS libraries, and compliance status in natural language, enabling use cases such as: "What are the top three SIF precursors at Site A this month?" or "Which chemicals in our inventory require updated SDS under the new GHS revision?"

---

## Similar Products — Developer Documentation & APIs

### SafetyCulture (iAuditor)
- **Description:** Mobile-first inspection and safety management platform used by 85,000+ organisations. Covers audits, incidents, training, and assets.
- **API Documentation:** https://developer.safetyculture.com/
- **OpenAPI / JSON Schemas:** https://github.com/SafetyCulture/api-json-schemas
- **SDKs/Libraries:** No official SDK; community integrations via Zapier, n8n, Pipedream, and Relevance AI
- **Developer Guide:** https://developer.safetyculture.com/reference/introduction
- **Standards:** REST/JSON, OpenAPI 3.x, Webhooks
- **Authentication:** OAuth 2.0, API Key; API access requires Premium or Enterprise plan

### Intelex EHSQ
- **Description:** Enterprise EHSQ platform with 15+ modules covering safety, environment, quality, and training.
- **API Documentation:** https://developers.intelex.com/
- **SDKs/Libraries:** Python SDK on PyPI (`pip install intelex`); Postman collection with v6 API examples
- **Developer Guide:** https://www.intelex.com/products/applications/api
- **Standards:** REST/JSON, API Key authentication and OAuth 2.0 token auth (v6.6.7+)
- **Authentication:** API Key; OAuth 2.0 Bearer Token (v6 API)

### Cority (CorityOne)
- **Description:** Enterprise EHS+ platform with 13 Cortex AI agents across 30+ use cases. Covers occupational health, safety, environment, and sustainability.
- **API Documentation:** Available to customers via Cority support portal (not publicly indexed)
- **SDKs/Libraries:** Not publicly documented
- **Developer Guide:** https://www.cority.com/ (customer-facing only)
- **Standards:** REST/JSON; described as processing tens of thousands of transactions per day
- **Authentication:** Not publicly specified; enterprise SSO and OAuth supported

### Benchmark Gensuite
- **Description:** Enterprise EHS and sustainability platform with 18-language support and CSRD-ready ESG modules.
- **API Documentation:** Available to enterprise customers; not publicly indexed
- **SDKs/Libraries:** Not publicly documented
- **Developer Guide:** https://benchmarkgensuite.com/
- **Standards:** REST/JSON; ERP integration (SAP)
- **Authentication:** Enterprise SSO; API Key

### Enablon (Wolters Kluwer)
- **Description:** Enterprise GRC and EHS platform operating in 160+ countries, with strong risk management and sustainability reporting.
- **API Documentation:** Available via Microsoft Azure Marketplace listing and Enablon support portal
- **SDKs/Libraries:** Not publicly documented
- **Developer Guide:** https://www.wolterskluwer.com/en/solutions/enablon
- **Standards:** REST/JSON; Microsoft Azure-native; ERP integration (SAP, Oracle, Microsoft Dynamics)
- **Authentication:** OAuth 2.0; Microsoft Entra ID (Azure AD) SSO

### VelocityEHS Accelerate®
- **Description:** Comprehensive EHS platform covering safety, ergonomics, chemical management, environmental compliance, and ESG.
- **API Documentation:** Available to enterprise customers; not publicly indexed
- **SDKs/Libraries:** Not publicly documented
- **Developer Guide:** https://www.ehs.com/
- **Standards:** REST/JSON; OSHA ITA direct submission
- **Authentication:** Enterprise SSO; API Key

### OSHA Injury Tracking Application (ITA)
- **Description:** OSHA's official portal for electronic submission of injury and illness data (300A and 300 log data).
- **API Documentation:** https://www.osha.gov/injuryreporting/
- **SDKs/Libraries:** Not officially provided; EHS vendors implement native integration
- **Developer Guide:** https://www.osha.gov/injuryreporting/faqs
- **Standards:** REST/JSON; CSV bulk upload; web form entry
- **Authentication:** Employer account registration; API key for automated submission

### ECHA (European Chemicals Agency) — REACH / C&L Inventory
- **Description:** EU regulatory authority for chemical substance registration and classification. Provides API access to the REACH Candidate List, C&L Inventory, and substance data.
- **API Documentation:** https://echa.europa.eu/information-on-chemicals
- **SDKs/Libraries:** ECHA CHEM data download; structured substance data in XML and JSON formats
- **Standards:** REST/JSON; XML; Open Data
- **Authentication:** Public API; no authentication required for read access

---

## Notes

- **No open EHS data exchange standard exists.** Unlike healthcare (HL7 FHIR) or finance (XBRL), the EHS domain lacks a universally adopted open data format for transferring incident records, risk assessments, or audit findings between systems. Each vendor maintains a proprietary data model. This represents both a technical gap and an opportunity for an open-source EHS platform to establish a community standard.

- **ISO 14001:2026 revision is in progress.** The revision aims to simplify requirements without changing the PDCA core structure. EHS platforms and open-source tools built in 2026 should design environmental management modules to accommodate the updated requirements when the revised standard is published.

- **CSRD data demand is expanding.** The EU CSRD came into force for large companies in 2025 and extends to SMBs in subsequent phases. EHS platforms will increasingly be required to serve as primary data sources for mandatory ESRS sustainability reports, meaning EHS data models must support ESG metric collection aligned to ESRS E1-E5 (environmental) and S1 (social) disclosure requirements.

- **OSHA ITA API maturity.** OSHA's Injury Tracking Application REST API is available and used by major EHS platforms for automated annual submission. The API is well-documented and stable, making it a priority integration target for any new EHS platform targeting the US market.

- **MCP ecosystem is emerging.** As of May 2026, no major EHS platform has published an MCP server specification. An open-source EHS platform with a first-class MCP server would be the first in the category to do so, enabling immediate integration with Claude, GPT-4o, and other LLM assistants for natural language EHS queries.
