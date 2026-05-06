# Environmental Health & Safety (EHS)

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source EHS platform for incident reporting, chemical management, training compliance, and audits — accessible to organisations priced out of enterprise incumbents.

Environmental Health & Safety (EHS) is a candidate project for an open-source platform that consolidates incident management, chemical safety, audits, training, and regulatory reporting into a single AI-augmented system. It is aimed at safety managers, sustainability officers, and operations teams in industrial, manufacturing, and construction organisations who need enterprise-grade EHS capability without enterprise-grade pricing or implementation complexity.

---

## Why Environmental Health & Safety (EHS)?

- The EHS software market is dominated by enterprise SaaS platforms (VelocityEHS, Intelex, Cority, Enablon, Sphera) that use custom enterprise pricing in the tens to hundreds of thousands of dollars per year — leaving SMBs with a sharp gap between simple checklist tools like iAuditor and feature-complete EHS suites.
- Implementations of incumbent platforms typically take 3–12 months and require significant external consulting, blocking faster adoption.
- No open-source EHS platform with comparable feature breadth was identified in research; the category is essentially a closed-source monoculture.
- AI capability has shifted from differentiator to baseline expectation by 2026, but the most advanced AI features (predictive SIF modelling, computer vision PPE detection, permit analysis) are locked behind premium proprietary suites.
- There is no dominant open standard for EHS data exchange, so each platform is a data silo creating switching costs and integration burden.

---

## Key Features

### Incident, Risk & CAPA Management

- Incident and near-miss reporting with structured capture (type, location, involved parties, description)
- Root cause analysis with AI-assisted cause suggestion from incident text
- Corrective and preventive action (CAPA) management with assignments, due dates, and status tracking
- Risk assessment builder with configurable matrices and a risk register
- Predictive SIF (Serious Injury and Fatality) risk scoring using near-miss and inspection trend data

### Audits, Inspections & Training

- Audit and inspection management with mobile-friendly, offline-capable data capture
- OSHA 300/300A log generation and ITA-compatible export
- Training management with certification tracking and automated recertification reminders
- Document control and version management
- Compliance workflows aligned to ISO 45001, ISO 14001, and OSHA 29 CFR 1910 / 1926

### Chemical & Environmental Compliance

- SDS library management with GHS-aligned chemical inventory
- Environmental compliance tracking for waste, emissions, and permits
- AI-powered regulatory compliance monitoring and gap analysis
- Natural language chemical safety query interface (SDS assistant)
- REACH / RoHS substance compliance tracking

### ESG & Sustainability Reporting

- ESG metrics collection and reporting aligned to GRI and CSRD frameworks
- Multi-framework support including ISSB / IFRS S2, CDP, and TCFD
- GHG, energy, water, and waste metric capture embedded within EHS data

### Field & Integration Capability

- Voice-to-text incident reporting for hands-free field use
- Computer vision PPE compliance detection from site images or live camera feeds
- REST API with OpenAPI specification for third-party integrations
- Contractor management with prequalification workflows
- Multi-language support for global deployments

---

## AI-Native Advantage

AI is woven through the platform rather than added as a side module. Predictive incident modelling cross-references near-miss reports, inspection findings, corrective action completion rates, and environmental sensor data to surface leading indicators before incidents occur. Computer vision detects PPE non-compliance and hazards from existing CCTV or uploaded imagery. AI-driven regulatory monitoring interprets new rules against a specific operation and auto-generates required corrective tasks, while a natural language assistant lets workers query SDS data, exposure limits, and chemical incompatibilities without navigating document hierarchies.

---

## Tech Stack & Deployment

The project targets self-hosted and cloud deployment so organisations can choose where their safety and environmental data lives. A REST API with a published OpenAPI specification is a first-class deliverable for integration with HRIS, ERP (SAP, Oracle), and LMS systems, and for OSHA Injury Tracking Application (ITA) submission. The platform is designed to align with ISO 45001, ISO 14001, OSHA 29 CFR 1910 / 1926, the GHS chemical labelling standard, REACH / RoHS, and ESG disclosure frameworks (TCFD, GRI, CSRD). Mobile-first, offline-capable field capture is a core architectural requirement given typical connectivity in manufacturing and construction environments.

---

## Market Context

The EHS software market was valued at approximately USD 8.21 billion in 2025 and is projected to reach USD 8.9 billion in 2026 at a CAGR of roughly 8.4%, growing to USD 12.15 billion by 2030 (sources: MarketsandMarkets, The Business Research Company, Mordor Intelligence). Entry-level tools start around USD 23/month and SMB-focused mid-market platforms average around USD 188/month, while enterprise platforms (VelocityEHS, Cority, Enablon) use custom pricing in the tens to hundreds of thousands per year. Primary buyers are EHS managers and corporate safety directors at industrial, manufacturing, and construction organisations, sustainability officers managing ESG disclosure, and compliance teams handling chemical or hazardous materials operations.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
