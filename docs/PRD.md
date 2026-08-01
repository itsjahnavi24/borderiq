# BorderIQ Product Requirements Document

| Field | Value |
|---|---|
| Title | BorderIQ Product Requirements Document (PRD) |
| Version | 3.0 |
| Status | Draft for review |
| Owner | Jahnavi Ralhan |
| Last updated | 2026-08-01 |
| Scope | Product definition for the BorderIQ CBAM compliance intelligence platform: current MVP and target platform. Defines WHAT and WHY. Implementation detail lives in the LLD. |
| Related documents | [LLD](LLD.md), [ARCHITECTURE](ARCHITECTURE.md), [API_SPEC](API_SPEC.md), [DATA_MODEL](DATA_MODEL.md), [DECISION_ENGINE](DECISION_ENGINE.md), [TOOL_CALLING](TOOL_CALLING.md), [KNOWLEDGE_GRAPH](KNOWLEDGE_GRAPH.md), [SECURITY](SECURITY.md), [DEPLOYMENT](DEPLOYMENT.md), [ROADMAP](ROADMAP.md), [AGENTS](AGENTS.md) |

Labelling convention used throughout this document and the whole documentation suite:

- **Current MVP (verified)**: behaviour confirmed by direct inspection of source code available to the documentation effort (the frontend client).
- **Current MVP (described)**: behaviour of the hackathon backend as described by the project owner. The backend source has not yet been committed to this repository and has not been independently verified. Claims under this label must be re-verified when the code lands.
- **Representative demo data**: synthetic or hardcoded values used to make the MVP demonstrable. Never real customer or verified regulatory data.
- **Target architecture / Proposed**: designed in this documentation suite, not built.
- **Out of scope**: explicitly excluded.
- **Open design question**: a decision deliberately not yet made.

## 1. Document Control

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0 | 2026 (hackathon) | Jahnavi Ralhan | Initial hackathon concept notes |
| 2.0 | 2026-08-01 | Jahnavi Ralhan | Draft PRD skeleton committed to repository |
| 3.0 | 2026-08-01 | Jahnavi Ralhan | Full rewrite to the approved documentation contract: requirement IDs, acceptance criteria, current versus target labelling, personas, journeys, risks, phasing, glossary |

Review and approval: this document is a Draft until the owner marks it Approved. Requirements marked Proposed are design intent, not commitments, until scheduled in [ROADMAP.md](ROADMAP.md).

## 2. Executive Summary

BorderIQ is a compliance intelligence platform for the EU Carbon Border Adjustment Mechanism (CBAM). It helps manufacturers, exporters, and their advisors answer one operational question per shipment:

> Can this shipment be reported using the proposed emissions methodology, what evidence supports that decision, what is the estimated financial impact, and what must happen next?

Answering that question today requires reconciling fragmented shipment data, product and CN-code mappings, plant and production-route information, emissions evidence, verifier documentation, and lengthy, changing regulations. The work is manual, spreadsheet-heavy, repetitive, and difficult to audit.

A general-purpose language model can explain a regulation. It does not own enterprise shipment data, validated emission factors, document-verification status, deterministic calculation logic, regulatory versioning, organisational policies, approval workflows, or audit evidence. BorderIQ's value is the governed orchestration of all of these: structured operational data, unstructured regulatory evidence, deterministic calculation engines, policy evaluation, evidence validation, risk classification, human review, and audit-ready decision packages.

The current MVP demonstrates the two foundational subsystems: citation-backed regulatory question answering over official CBAM documents (retrieval-augmented generation), and a deterministic shipment calculation that compares actual against default embedded emissions and quantifies the cost difference. The target platform, defined in this document and detailed in the LLD, evolves the MVP into a structured compliance investigation workflow with a typed tool registry, versioned policy evaluation, entity resolution, evidence validation, scenario analysis, human review, and decision replay.

Architectural principle, applied everywhere: the language model may interpret, plan, classify, extract, and narrate. It must not invent or independently calculate authoritative compliance figures. Deterministic services calculate. Policy engines evaluate. Retrieval systems provide evidence. Humans approve high-impact decisions.

## 3. Product Vision

BorderIQ aims to become a governed decision layer between enterprise operational systems and environmental-trade compliance obligations, starting with CBAM.

Its purpose is not to replace ERP, carbon-accounting, or legal systems. Its purpose is to reconcile their data, apply the correct currently effective rule, execute validated calculations, and expose a defensible decision workflow with a replayable trace.

Success looks like this: a compliance officer opens BorderIQ with a shipment, and within minutes has a structured decision stating the eligible methodology, the calculated figures with lineage, the evidence gaps, the financial impact of the available reporting choices, the supporting regulation passages, and the review actions required before filing. Six months later, an auditor can replay exactly how that decision was produced.

Positioning relative to foundation models: BorderIQ does not compete with foundation models. It turns them into governed participants inside a specialised, deterministic, and auditable compliance workflow. General-purpose models explain regulations; BorderIQ reconciles enterprise data with the applicable rule, executes validated calculations, and produces a decision package that can be reviewed and replayed. RAG is one subsystem, not the product.

## 4. Industry and Regulatory Context

The EU Carbon Border Adjustment Mechanism applies carbon pricing to imports of covered carbon-intensive goods (initially iron and steel, cement, aluminium, fertilisers, hydrogen, and electricity) so that imported goods carry environmental costs comparable to EU-manufactured goods.

Operationally, CBAM obliges declarants to report the embedded emissions of covered goods per reporting period, using methodologies and default values published by the European Commission, with rules that have evolved through the transitional period and continue to evolve into the definitive regime. Three properties of the regime shape this product:

1. **The rules are versioned and effective-dated.** The correct answer for a shipment depends on which regulation version applies to its reporting period. Retrieving "a relevant passage" is insufficient; the platform must retrieve the currently effective rule for a specific shipment, product, production route, and period.
2. **Default values are intentionally conservative.** Where actual emissions cannot be sufficiently demonstrated, declarants fall back on published default values, which are generally set high. Producers whose real emissions intensity is lower than the default overpay when they cannot evidence actuals. The financially material decision is therefore evidential: can actuals be demonstrated for this shipment?
3. **Every figure must survive audit.** Reported values require supporting evidence: installation data, supplier declarations, verifier reports, and traceability to the methodology applied.

Regulatory disclaimer: this document describes product behaviour, not legal interpretation. Statements about CBAM are context for design and must not be relied on as regulatory guidance. See Section 26.

## 5. Problem Statement

For each shipment of covered goods, a compliance team must currently:

1. Identify the applicable regulation version for the reporting period.
2. Classify the product and map it to the correct CN code.
3. Identify the producing installation and production route.
4. Determine which emissions methodology is permitted given the available evidence.
5. Retrieve the applicable default values or validated actual emissions data.
6. Calculate embedded emissions (direct, indirect, and precursor where applicable).
7. Estimate the financial exposure under the available reporting choices.
8. Assemble supporting evidence and prepare audit documentation.

This information is fragmented across ERP systems, plant systems, engineering reports, supplier declarations, verifier documents, and hundreds of pages of legal text. The same analysis is reconstructed by hand in spreadsheets every reporting period.

Large language models improve step 1 in isolation (explaining what a regulation says) but do not solve steps 2 through 8, because they lack the enterprise data, deterministic logic, policy versioning, and audit controls the workflow requires.

## 6. Current Workflow (As-Is)

| Step | Actor | Tooling | Failure modes |
|---|---|---|---|
| Collect shipment and product data | Trade compliance analyst | ERP exports, email, spreadsheets | Inconsistent product naming, missing CN codes |
| Collect plant and process data | Plant engineer | Plant systems, engineering reports | Data not in reportable units, stale factors |
| Interpret the regulation | Analyst, sometimes external counsel | PDF reading, prior filings | Wrong version applied, clauses missed |
| Choose methodology | Analyst | Judgement | Unsupported use of actuals, or unnecessary use of defaults |
| Calculate | Analyst | Spreadsheet | Formula drift, unit errors, no lineage |
| Assemble evidence | Analyst | Shared drives | Missing or expired verifier documents |
| File and archive | Analyst | Manual | No replayable record of how figures were produced |

Elapsed time is measured in days to weeks per reporting cycle, repeated every cycle, with no institutional learning captured between cycles.

## 7. Root Causes

1. **Data fragmentation.** The inputs live in systems owned by different departments with no shared identifiers.
2. **Entity ambiguity.** The same product appears under ERP names, supplier names, CN codes, and regulatory categories with no maintained mapping.
3. **Regulatory volatility.** Rules, guidance, and default values change; spreadsheets and habits do not.
4. **No separation of interpretation from calculation.** Legal reading and arithmetic are performed by the same person in the same artefact, so neither is independently checkable.
5. **No decision record.** The output is a filed number, not a reconstructable decision.

## 8. Cost of Inaction

- **Direct overpayment**: defaulting when actuals were demonstrable converts conservative default values into real cost on every affected tonne, recurring each period.
- **Compliance risk**: using actuals without sufficient evidence, or applying a superseded rule, creates audit findings and potential penalties.
- **Operational cost**: analyst weeks per cycle, scaling linearly with shipment volume.
- **Audit exposure**: without decision lineage, every audit becomes archaeology.
- **Strategic blindness**: without scenario analysis, decarbonisation investments cannot be connected to compliance cost.

This PRD deliberately quantifies none of these with invented figures. Baseline measurement is itself a requirement (see Section 30).

## 9. Product Thesis

The strongest enterprise AI pattern for this domain is not autonomous free-form generation. It is governed orchestration:

1. Resolve the correct entities (product, CN code, installation, route).
2. Resolve the applicable regulation version and policy for the reporting period.
3. Validate evidence sufficiency before choosing a methodology.
4. Call deterministic services for every authoritative figure.
5. Evaluate versioned policies for eligibility.
6. Retrieve and cite the supporting regulatory passages.
7. Detect uncertainty and route high-risk or ambiguous cases to humans.
8. Narrate the result, and preserve a replayable decision trace.

The language model interprets, plans, classifies, extracts, and narrates. Deterministic services calculate. Policy engines decide eligibility. Human reviewers approve material or uncertain cases.

## 10. Target Users

| User | Primary need |
|---|---|
| Trade compliance managers and analysts | Faster, defensible shipment-level reporting decisions |
| Sustainability and ESG teams | Credible embedded-emissions figures with lineage |
| Carbon accounting specialists | Methodology eligibility and factor traceability |
| Export finance teams | Financial exposure of reporting choices |
| Internal auditors | Replayable decision records and evidence packages |
| Regulatory and sustainability consultants | A platform to deliver CBAM services across clients |
| Plant and process engineers | A consumer for validated installation data, without becoming reporting analysts themselves |

## 11. Buyers and Stakeholders

| Role | Interest | Buying influence |
|---|---|---|
| CFO / Finance | Tariff exposure and avoidable overpayment | Economic buyer |
| Head of Trade / Supply Chain | EU market access, filing reliability | Sponsor |
| Chief Sustainability Officer | Carbon data credibility, ESG alignment | Sponsor |
| Head of Compliance / Legal | Audit defensibility, controlled AI use | Approver, can veto |
| CIO / IT Security | Integration effort, data protection, model governance | Approver, can veto |
| Consulting partner | Deployable platform for client engagements | Channel buyer |

Open design question OQ-06 (Section 36): whether the primary buyer is the non-EU exporter, the EU importing declarant, or the consulting provider. The target data model supports all three; go-to-market sequencing depends on the answer.

## 12. Personas

**P1: Priya, Trade Compliance Manager (primary).** Owns CBAM declarations for a steel exporter. Non-lawyer, spreadsheet-fluent, audit-scarred. Needs: per-shipment decisions she can defend, evidence gap lists before filing, and less time inside PDFs. Fears: an AI that invents a clause. Trust condition: every claim carries a citation she can open; every number recomputes identically.

**P2: Daniel, Carbon Accounting Specialist.** Sits between engineering and compliance. Needs: methodology eligibility per installation and route, factor lineage, and unit-safe calculations. Trust condition: the calculation trace shows formula, inputs, units, and factor sources.

**P3: Meera, Sustainability Consultant.** Runs CBAM engagements for multiple clients. Needs: repeatable investigations, client-separable data, exportable decision packages. Trust condition: tenant isolation and consistent outputs across analysts.

**P4: Tomas, Internal Auditor (secondary).** Samples filed declarations quarterly. Needs: decision replay, the exact document versions and policies applied, and reviewer actions. Trust condition: the audit package is complete without asking anyone.

## 13. Jobs To Be Done

- When a shipment of covered goods is scheduled, determine the permitted reporting methodology and its evidence requirements, so filing is neither overpaid nor unsupported.
- When emissions data arrives from a plant or supplier, validate whether it is sufficient and current for use, so actuals can be claimed with confidence.
- When a regulation changes, identify which products, routes, policies, and open decisions are affected, so nothing is filed against a superseded rule.
- When an auditor asks about a past declaration, reproduce the complete decision with evidence, so the answer takes minutes, not weeks.
- When management asks what decarbonisation is worth, simulate compliance cost under alternative scenarios, so investment cases use defensible numbers.

## 14. Goals

| ID | Goal |
|---|---|
| G-1 | Reduce elapsed time per shipment compliance investigation from days to minutes for standard cases |
| G-2 | Reduce unsupported or unnecessary use of default values by surfacing evidence-backed actuals eligibility |
| G-3 | Detect missing, expired, or conflicting evidence before filing, not after |
| G-4 | Make every calculated figure reproducible: same inputs, same versions, same outputs |
| G-5 | Make every regulatory claim traceable to a cited passage of an authoritative document version |
| G-6 | Support bulk enterprise workflows (batch shipments, recurring periods) |
| G-7 | Version rules, evidence, models, and prompts so decisions are replayable |
| G-8 | Produce audit-ready decision packages as a first-class output |

## 15. Non-Goals

- Legal advice, or any output presented as a legal conclusion.
- Autonomous regulatory filing or submission to authorities.
- Replacement of ERP, MES, or carbon-accounting systems.
- Automatic activation of policy rules extracted from regulation text without qualified human approval.
- LLM-generated numeric liability figures under any circumstance.
- Universal coverage of all industrial production routes in the first releases.
- Real-time IoT ingestion as a source of final emissions figures (raw meter data must pass through validated processing before BorderIQ consumes the verified emissions record).
- Certification of the calculation engine as legally authoritative (Out of scope until a formal validation process exists; see Section 26).

## 16. Product Principles

| # | Principle | Practical consequence |
|---|---|---|
| PP-1 | Deterministic decisions | Authoritative figures come only from versioned deterministic engines. The LLM never performs arithmetic that reaches a decision or filing. |
| PP-2 | Evidence before eligibility | Methodology selection is a function of validated evidence state, not user assertion. |
| PP-3 | Effective-date correctness is a feature | Retrieval and policy evaluation are scoped to the rule version effective for the shipment's reporting period. |
| PP-4 | Structured outputs | Machine-validated decision objects, not unvalidated prose. Narration is a rendering of the structure. |
| PP-5 | Abstain rather than guess | When evidence or retrieval is insufficient, the system says so and routes to review. |
| PP-6 | Fail closed | Optional and high-risk capabilities degrade to safe, shape-preserving responses. |
| PP-7 | Humans approve what matters | Material exposure, ambiguity, and policy changes require human approval. |
| PP-8 | Everything replayable | Inputs, document versions, policies, model versions, prompts, tool calls, and reviewer actions are recorded per decision. |
| PP-9 | Simplest reliable component | Deterministic workflow over agent, SQL over graph for CRUD, exact lookup over vector search for exact identifiers, modular monolith before microservices. |
| PP-10 | Honest data labelling | Synthetic and demo data is always identified as such, in the UI and in the docs. |

## 17. Current MVP Scope

The MVP was built for a hackathon to prove the two hardest-to-trust subsystems: grounded regulatory answers and deterministic shipment math. Verification status is split per the labelling convention.

### 17.1 Current MVP (verified): frontend

Source-inspected single-page client (Flask-template-ready HTML, Bootstrap 5, vanilla JavaScript).

- Surfaces: Executive Dashboard, Compliance Assistant, Shipment Analyzer, Knowledge Centre, Portfolio Analytics (labelled Coming Soon, demo data), Regulatory Updates (labelled demo content), Audit Trail (browser-session only), Settings (read-only).
- Live API integration: `POST /api/ask`, `GET /api/status`, `GET /api/knowledge-base`, with a 20-second timeout and a labelled demo fallback when the backend is unreachable ("Demo Mode, Backend Offline").
- Compliance Assistant: question input, retrieval pipeline progress indicator, answer card, evidence cards showing source file, page number, document type, and chunk ID, with expand and copy citation actions.
- Shipment Analyzer: product selection auto-resolves a default emissions value, CN code, category, and supporting document page from a hardcoded 8-product catalogue (Representative demo data); a client-side demo rules engine computes default total, actual total, avoided emissions, savings, a ratio-based risk band, and a templated recommendation. A code comment marks the intended swap point for a backend analysis endpoint.
- Session audit trail: each question and analysis is logged in-memory with inputs, evidence references, and chunk IDs; lost on page refresh.
- Known MVP artefacts to be corrected in the target design: a hardcoded retrieval-confidence figure, floating-point arithmetic, unsourced risk thresholds, and dashboard KPIs that are Representative demo data (labelled as such in the UI).

### 17.2 Current MVP (described): backend

As described by the project owner; source not yet committed to this repository and not independently verified:

- Python and Flask, JSON REST endpoints including `POST /api/ask`, `GET /api/status`, `GET /api/knowledge-base`.
- Retrieval over a small corpus of official CBAM PDFs (regulation, guidance, default values documents) loaded via LangChain-compatible PDF tooling, chunked with metadata (source file, page number, document type, chunk ID), embedded with an OpenAI embedding model, stored in a locally persisted ChromaDB, retrieved top-k, and narrated by an OpenAI language model with citations.
- A Python product catalogue and rules-engine calculations mirroring the shipment analysis above.

Open verification items: exact chunking strategy (page-wise versus recursive character splitting has been described inconsistently), exact model identifiers, whether a shipment-analysis endpoint exists server-side, presence of tests, and environment configuration. These are tracked as OQ-01 in Section 36 and must be resolved when the code is committed.

### 17.3 Current MVP limitations (both layers)

No durable shipment or decision storage; no authentication or multi-tenancy; local single-node vector store; no ERP, verifier, or carbon-price integrations; no policy governance; no effective-date scoping in retrieval; no evaluation harness; no calibrated confidence; browser-only audit trail; demo product catalogue whose default values are unverified against the source document.

## 18. Target Product Scope

The target platform (Proposed; phased in [ROADMAP.md](ROADMAP.md)) is a compliance investigation system:

1. **Shipment investigation**: intake (form, CSV or Excel, batch API, later ERP connectors and events), validation, canonicalisation, entity resolution, regulatory-version resolution, evidence sufficiency, methodology selection, deterministic calculation, policy evaluation, risk classification, regulatory grounding, structured decision assembly, review routing, and audit packaging. Lifecycle detail in [DECISION_ENGINE.md](DECISION_ENGINE.md).
2. **Tool orchestration**: a versioned, typed tool registry; the LLM may select tools and may not fabricate their outputs. Contract in [TOOL_CALLING.md](TOOL_CALLING.md).
3. **Deterministic calculation and policy layer**: separate calculation, policy, risk, scenario engines and a decision assembler; decimal-safe, unit-explicit, versioned, test-fixtured. AI may suggest policy changes; a qualified reviewer must approve activation.
4. **Regulatory intelligence**: document registry with provenance, versioning and supersession, structural parsing, hybrid retrieval (dense plus lexical) with metadata scoping and reranking, citation validation, and abstention on insufficient evidence.
5. **Entity linking and reconciliation**: canonical IDs, alias tables, deterministic mappings first, fuzzy matching as fallback with confidence and review queues; explicit distinction between missing, conflicting, stale, unverified, unsupported, and suspicious data.
6. **Knowledge graph (optional, fail-closed)**: relationship traversal where it earns its keep; see [KNOWLEDGE_GRAPH.md](KNOWLEDGE_GRAPH.md).
7. **Decision twin and scenarios**: versioned computational representation of a decision's inputs enabling controlled what-if simulation that never overwrites the official decision.
8. **Regulatory change intelligence**: ingest, diff, classify, map impact, create review tasks; policy updates only after human approval.
9. **Human review**: routing rules, review tasks, decision statuses including auto-approved, review required, blocked, insufficient evidence, unsupported case.
10. **Auditability and replay**: complete decision traces reconstructing any past decision.
11. **Enterprise integrations and events**: adapters (CSV, Excel, SAP, Oracle, Dynamics, SQL, object storage, document management, sustainability platforms, verifier systems, carbon-price services) and an event backbone with idempotency, retries, dead-lettering, and replay.
12. **Multi-tenancy and production security**: per [SECURITY.md](SECURITY.md) and [DEPLOYMENT.md](DEPLOYMENT.md).

## 19. Product Surfaces

| Surface | Current MVP | Target |
|---|---|---|
| Executive Dashboard | Verified; KPIs are Representative demo data | Live portfolio rollups from the decision store |
| Compliance Assistant | Verified frontend; backend described | Retains Q&A; answers scoped by shipment context and effective dates; abstention states |
| Shipment Analyzer | Verified frontend with client-side demo engine | Full investigation workflow backed by server-side engines and the structured decision object |
| Knowledge Centre | Verified; lists indexed documents | Document registry view: versions, effective dates, supersession, provenance |
| Portfolio Analytics | Demo charts, labelled | Live risk, savings, evidence-completeness analytics |
| Regulatory Updates | Demo timeline, labelled | Change-intelligence feed with impact mapping and review tasks |
| Audit Trail | Session-only, verified | Durable decision log with replay and audit-package export |
| Review Workspace | Not present | Proposed: queue of review tasks with evidence and approve, reject, request-info actions |
| Administration | Settings read-only | Proposed: tenant configuration, policy management, mapping review |

## 20. End-to-End User Journeys

### J1: Regulatory question (Current MVP, end-to-end)

Priya asks "What are default values for pig iron?" The system retrieves passages from the indexed official documents, answers only from that context, and displays evidence cards with document, page, and chunk identifiers. Priya expands a citation, copies it into her working notes, and the interaction is captured in the session audit trail. If the backend is down, the UI shows a clearly labelled demo response rather than failing silently.

### J2: Shipment investigation (Target; MVP demonstrates a reduced form)

Priya uploads a CSV of 40 consignments. Each is validated and canonicalised; products resolve to CN codes (two ambiguous mappings go to a review queue); the regulation version effective for the reporting period is selected; evidence checks find one installation with an expired verifier report; eligible shipments get deterministic actual-versus-default calculations with exposure and savings; policies classify each as auto-approved, review required, or insufficient evidence; each decision carries cited passages, a calculation trace, and required actions. Priya reviews the flagged cases, approves, and exports audit packages. The MVP form-based analyzer is the single-shipment, demo-data ancestor of this journey.

### J3: Regulation change (Target)

A new implementing regulation is ingested. The diff engine classifies changed articles and default-value tables, maps affected CN codes and routes, lists impacted active policies and open decisions, and creates a review task. Daniel reviews the proposed policy updates, approves them with an effective date, and affected in-flight investigations are re-run. Nothing changed without human approval, and the change itself is now part of the audit record.

## 21. Functional Requirements

Priority: P0 = required for the phase where it first appears, P1 = important, P2 = desirable. Status values: Implemented (verified), Implemented (described), Demo only, Proposed (Phase N per ROADMAP).

### 21.1 Current MVP requirements

| ID | Requirement | Rationale | Priority | Acceptance criteria | Status |
|---|---|---|---|---|---|
| FR-001 | Answer natural-language CBAM questions with citations. Given a question, return an answer plus a source list where each source has source_file, page_number, document_type, chunk_id. | Grounded answers are the trust foundation | P0 | Response schema matches; every answer displays at least one openable citation; answers derive only from retrieved context | Implemented (frontend verified; backend described) |
| FR-002 | Report knowledge-base and service status: documents indexed, chunks created, vector DB state, LLM state, embedding model. | Operators and demos need an honest health view | P0 | `GET /api/status` returns the five fields; UI renders them and degrades to labelled demo values on failure | Implemented (frontend verified; backend described) |
| FR-003 | List indexed documents with type, pages processed, status, last indexed. | Users must see exactly what the system knows | P0 | `GET /api/knowledge-base` renders as document cards; client-side filter works; empty-filter state recoverable | Implemented (frontend verified; backend described) |
| FR-004 | Single-shipment analysis: user supplies product, quantity, actual emissions intensity, carbon price; system auto-resolves default value, CN code, category, and source reference; outputs status, risk, savings, avoided emissions, summary, recommendation, and supporting reference. | Demonstrates decision support beyond Q&A; users must never hand-enter defaults | P0 | Selecting a product populates the reference panel without user input; results render for valid inputs; invalid inputs produce inline errors | Demo only (client-side engine, 8-product Representative demo catalogue) |
| FR-005 | Deterministic separation: no authoritative figure in any surface is produced by the LLM. | Core trust principle PP-1 | P0 | Code inspection finds no LLM call in any calculation path; identical inputs yield identical outputs | Implemented (verified for frontend demo engine; described for backend) |
| FR-006 | Session audit trail: every question and analysis logged with timestamp, inputs, evidence references, and outcome summary, with an expandable explainability view. | Seed of decision replay | P1 | Actions appear in the Audit Trail surface within the session; entries show evidence used and chunk IDs | Implemented (verified; volatile by design in MVP) |
| FR-007 | Honest degradation: when the backend is unreachable, surfaces fall back to labelled demo content and the global status badge switches to demo mode. | Never silently fake liveness | P0 | Killing the backend produces labelled fallbacks and the demo badge; response shapes are preserved | Implemented (verified) |

### 21.2 Target platform requirements (all Proposed)

| ID | Requirement | Rationale | Priority | Acceptance criteria | Status |
|---|---|---|---|---|---|
| FR-010 | Shipment intake via form, CSV or Excel import, and batch API, with schema validation and per-row error reporting. | Enterprise volume enters in bulk | P0 | A 1,000-row CSV imports with a per-row accepted or rejected report; rejected rows carry actionable reasons | Proposed (Phase 3) |
| FR-011 | Entity resolution: products and CN codes resolve via canonical IDs and alias tables; deterministic mappings preferred; fuzzy match only as fallback with a confidence score; ambiguous mappings create review-queue items with provenance. | Wrong product mapping corrupts everything downstream | P0 | Known aliases resolve deterministically; ambiguous cases never auto-resolve; every mapping records method and source | Proposed (Phase 1 basic, Phase 3 full) |
| FR-012 | Regulatory-version resolution: every investigation binds to the regulation and policy versions effective for its reporting period. | Effective-date correctness is a first-class requirement | P0 | Golden tests: shipments in different periods bind to different versions where applicable; superseded versions are never applied to current periods | Proposed (Phase 4 with policy engine, Phase 5 full) |
| FR-013 | Evidence sufficiency validation: required evidence per methodology is checked (presence, validity window, verification status) before methodology selection; outcomes distinguish missing, expired, unverified, conflicting. | Eligibility follows evidence, PP-2 | P0 | An expired verifier report blocks actuals eligibility with a stated reason; the decision lists exact evidence gaps | Proposed (Phase 4) |
| FR-014 | Policy evaluation: policy-as-code with versions, effective dates, jurisdiction, product and route applicability, evidence requirements, and approval thresholds; AI-suggested policy changes require qualified human approval before activation. | Governed eligibility, PP-7 | P0 | Policies are versioned data, not code branches; activation requires a recorded approval; decisions cite applied policy versions | Proposed (Phase 4) |
| FR-015 | Deterministic calculation engine: direct, indirect, and precursor embedded-emissions calculations plus default and actual scenarios and exposure estimation; pure functions, explicit units, decimal-safe arithmetic, versioned formulas, golden-test fixtures, full input and factor lineage. | Authoritative figures, PP-1, G-4 | P0 | Golden tests pass; unit mismatches are rejected, not coerced; every CalculationRun records formula version and inputs | Proposed (Phase 1 core, Phase 2 hardened) |
| FR-016 | Scenario comparison: default versus actual, carbon-price change, route change, energy-mix change, evidence-state change, regulation-version change; scenarios record assumptions and never overwrite official decisions. | Decision support, not just reporting | P1 | A scenario run produces a comparison object referencing the base decision; official decision unchanged | Proposed (Phase 7) |
| FR-017 | Risk classification: rule-based scoring over exposure magnitude, evidence state, mapping confidence, and data anomalies, with documented thresholds. | Routing and prioritisation | P1 | Same inputs, same class; thresholds documented and versioned; class changes are explainable by the contributing factors | Proposed (Phase 2 basic, Phase 4 policy-linked) |
| FR-018 | Structured compliance decision object: status, methodology, eligibility, calculated figures, assumptions, missing evidence, risk, supporting sources, applied policies, required actions, reviewer state, multi-dimensional confidence, provenance. | Machine-checkable output, PP-4 | P0 | Decisions validate against a published schema; invalid decisions are rejected, never rendered | Proposed (Phase 2) |
| FR-019 | Human review workflow: routing rules for ambiguity, evidence gaps, material exposure, and novel cases; review tasks with approve, reject, request-information; decision statuses auto-approved, review required, blocked, insufficient evidence, unsupported case. | PP-7 | P0 | Cases matching routing rules cannot finalise without a reviewer action; reviewer identity and action recorded | Proposed (Phase 4) |
| FR-020 | Audit package and replay: per decision, store input snapshot, document versions, retrieved passages and scores, tool calls with inputs and outputs, policy and engine versions, model and prompt versions, structured decision, reviewer actions; support reconstruction of any past decision. | G-7, G-8 | P0 | Replaying a stored decision reproduces identical figures and evidence references; audit package exports complete | Proposed (Phase 2 trace, Phase 4 full package) |
| FR-021 | Versioned tool registry: every tool has stable name, owner, version, typed input and output schemas, validation, error semantics, timeout, idempotency, permissions, audit fields, approval requirement, and deterministic-or-probabilistic classification; the LLM may select tools and cannot fabricate outputs. | Governed orchestration | P0 | Tool calls with schema-invalid inputs are rejected; every execution is traced; high-impact tools require approval context | Proposed (Phase 2) |
| FR-022 | Regulatory ingestion: authoritative-source allowlist, immutable raw storage, checksums and provenance, document versioning, effective dates, supersession links, structural parsing by article, annex, table, section, and metadata enrichment. | Trustworthy corpus, G-5 | P0 | Every fragment traces to a checksummed source version; superseded documents remain queryable but never retrieved for current periods | Proposed (Phase 5) |
| FR-023 | Hybrid scoped retrieval: metadata filtering (jurisdiction, regulation family, effective date, reporting period, CN code, category, route, methodology, emissions type, version) before dense-plus-lexical search, reranking, citation validation, sufficiency checks, and abstention when evidence is inadequate. | The hard problem is the correct currently effective rule | P0 | Retrieval eval suite meets targets in Section 29; abstention triggers on out-of-corpus questions instead of confident wrong answers | Proposed (Phase 5) |
| FR-024 | Regulatory change intelligence: ingest, structural diff, change classification, impact mapping to CN codes, routes, methodologies, active policies, and open decisions; human-review task creation; approved-only policy updates; re-run of affected decisions. | J3; no silent legal drift | P1 | A changed default-value table produces an impact report and review task; no policy activates without approval | Proposed (Phase 5) |
| FR-025 | Knowledge graph (optional): entities and relationships per KNOWLEDGE_GRAPH.md, used for multi-hop impact and similarity queries; fully fail-closed with shape-preserving empty responses when absent. | Relationship intelligence where traversal earns it | P2 | With the graph disabled, all endpoints return identical shapes with empty graph sections; graph answers cite traversal paths | Proposed (Phase 6) |
| FR-026 | Decision twin: versioned computational representation of shipment, installation, route, emissions inputs, regulation and policy versions, price assumptions, and evidence state, powering FR-016 simulations. | Precise scenario semantics | P2 | Twin versions are immutable; simulations reference a twin version and record assumptions | Proposed (Phase 7) |
| FR-027 | Durable enterprise data store: shipments, decisions, evidence, mappings, and audit records persisted with retention rules. | Nothing enterprise is browser-session | P0 | Data survives restart; retention policies enforceable per record class | Proposed (Phase 3) |
| FR-028 | Integration adapters: CSV and Excel first; SAP, Oracle, Dynamics, SQL, object storage, document management, sustainability platforms, verifier systems, and carbon-price services behind a stable adapter interface. | Meet data where it lives | P1 | Adding an adapter requires no core-schema change; each adapter documents field mappings and failure behaviour | Proposed (Phase 8) |
| FR-029 | Multi-tenancy: tenant_id on transactional entities, tenant-scoped retrieval namespaces, storage prefixes, quotas, per-tenant policies, aliases, and mappings; cross-tenant access prevented at every layer. | Consulting and SaaS deployment | P0 for SaaS | Cross-tenant retrieval tests fail closed; tenant deletion removes tenant data per retention policy | Proposed (Phase 9) |
| FR-030 | Declaration draft export: assemble a draft declaration package from finalised decisions for human filing; never auto-submit. | Last mile without autonomy | P1 | Export requires finalised decisions and a human trigger; package contents match the audit record | Proposed (Phase 8) |

## 22. Non-Functional Requirements

| ID | Requirement | Rationale | Priority | Acceptance criteria | Status |
|---|---|---|---|---|---|
| NFR-001 | Determinism and reproducibility: identical inputs and versions produce byte-identical decision figures. | Audit survival | P0 | Replay test suite passes on every release | Proposed (Phase 2) |
| NFR-002 | Interactive latency: single-shipment investigation completes in under 30 seconds at p95 for standard cases; regulatory Q&A under 15 seconds at p95. Targets, to be validated. | Usable in daily work | P1 | Load-test report against targets; documented if revised | Proposed (Phase 3) |
| NFR-003 | Batch throughput: 1,000-shipment import processed asynchronously with progress reporting; no interactive blocking. | Enterprise cycles | P1 | Batch of 1,000 completes with per-row outcomes and a completion event | Proposed (Phase 3) |
| NFR-004 | Availability: 99.5 percent monthly for the application tier in the first enterprise phase. Target, to be validated against buyer requirements. | Enterprise expectation | P1 | Uptime measured and reported | Proposed (Phase 9) |
| NFR-005 | Auditability retention: decision traces and audit packages retained for a configurable period (default 7 years, aligned to typical trade-document retention; confirm per jurisdiction as OQ-09). | Regulatory audit windows | P0 | Retention configurable per tenant; deletion honoured after expiry | Proposed (Phase 4) |
| NFR-006 | Security baseline: authentication, RBAC, encryption in transit and at rest, secrets management, audit logging per SECURITY.md. | Table stakes | P0 | Security review checklist passes before any non-demo deployment | Proposed (Phase 3 onward) |
| NFR-007 | Tenant isolation: no query path can return another tenant's data; verified by automated tests at API, retrieval, storage, and graph layers. | SaaS integrity | P0 for SaaS | Isolation test suite in CI; zero cross-tenant results | Proposed (Phase 9) |
| NFR-008 | Observability: structured logs with correlation IDs, metrics for latency, tool failure rate, queue depth, cost per investigation, and retrieval quality signals. | Operability | P1 | Dashboards exist; every investigation traceable end to end by request ID | Proposed (Phase 3) |
| NFR-009 | Graceful degradation: optional subsystems (graph, reranker, change feed) fail closed with preserved response shapes, mirroring the MVP's demo-mode behaviour. | PP-6 | P0 | Chaos tests disabling each optional subsystem keep core investigation functional | Proposed (per subsystem) |
| NFR-010 | Cost controls: per-investigation model-token and search budgets; caching of stable retrievals; cost per investigation reported. | Unit economics | P1 | Budget breaches are logged and capped, not silently exceeded | Proposed (Phase 3) |
| NFR-011 | Accessibility: WCAG 2.1 AA for the web application. MVP has partial support (focus states, reduced motion, aria-live); unaudited. | Enterprise procurement and inclusivity | P2 | Audit report with remediation list | Partially implemented (verified, unaudited); Proposed for AA |
| NFR-012 | Schema evolution: API and event schemas versioned; breaking changes only with a major version and migration notes. | Integration stability | P1 | Contract tests against published schemas in CI | Proposed (Phase 2) |

## 23. AI-Specific Requirements

| ID | Requirement | Rationale | Priority | Acceptance criteria | Status |
|---|---|---|---|---|---|
| AIR-001 | The LLM must not produce authoritative compliance figures. All figures in decisions come from the deterministic engines; the LLM narrates provided values only. | PP-1 | P0 | Output validator rejects LLM-introduced numerals in figure fields; code review gate | Implemented in MVP design (verified for frontend path); Proposed enforcement (Phase 2) |
| AIR-002 | All LLM outputs destined for decisions are structured and schema-validated; free prose appears only in designated narrative fields. | PP-4 | P0 | Invalid structured outputs are retried or abstained, never rendered | Proposed (Phase 2) |
| AIR-003 | Citation obligation: every regulatory claim in a narrative maps to a retrieved passage; uncited claims are dropped or the response abstains. | G-5 | P0 | Citation-completeness eval meets target; spot-check audits | Current MVP provides citations (described backend); validation Proposed (Phase 5) |
| AIR-004 | Abstention: when retrieval sufficiency or evidence completeness falls below thresholds, the system returns an explicit insufficient-evidence outcome rather than an answer. | PP-5 | P0 | Abstention eval set: out-of-corpus and under-evidenced cases abstain at target rate | Proposed (Phase 5) |
| AIR-005 | No single global confidence percentage. Confidence is reported per dimension: mapping confidence, evidence completeness, retrieval sufficiency, policy certainty, calculation validity. | Honest uncertainty | P0 | Decision schema contains the five dimensions; no aggregate score is displayed. The MVP's hardcoded confidence figure is removed | Proposed (Phase 2); MVP currently violates (Demo only) |
| AIR-006 | Retrieved documents and tool outputs are untrusted data: instruction-like content in retrieved text is never executed; prompts isolate evidence from instructions. | Injection containment | P0 | Injection test corpus (including document-embedded instructions) does not alter tool selection or outputs | Proposed (Phase 2 onward); see SECURITY.md |
| AIR-007 | Model, prompt-template, and parameter versions are recorded on every decision and Q&A trace; model upgrades require passing the evaluation gates before rollout. | Replay and drift control | P0 | Every trace carries model and prompt versions; upgrade checklist enforced | Proposed (Phase 2) |
| AIR-008 | No raw chain-of-thought is exposed to users. Explainability is delivered through tool traces, evidence, calculation traces, and decision rationale fields. | Controlled explainability | P0 | UI and API expose traces, not model reasoning dumps | Proposed (aligned with FR-020) |
| AIR-009 | RAG is not claimed to eliminate hallucination anywhere in product or documentation; claims are limited to grounding, citation, and measured unsupported-claim rates. | Honesty | P0 | Documentation and UI copy audit | Ongoing (this suite complies) |
| AIR-010 | AI-suggested policy or mapping changes are proposals with provenance; activation requires qualified human approval. | PP-7, FR-014 | P0 | No activation path exists without a recorded approval | Proposed (Phase 4) |

## 24. Explainability Requirements

Every finalised decision must expose, in the UI and the audit package: the evidence used (documents, versions, pages, passages, retrieval scores), the calculation trace (formula version, inputs, units, factors and their sources, intermediate values), the applied policies (IDs and versions with effective dates), the tool-call trace (sequence, inputs, outputs, durations, outcomes), the reviewer trail (who, what, when, rationale), and the confidence dimensions of AIR-005. Explainability is rendered from stored structure (FR-018, FR-020), never regenerated after the fact. Status: session-level seed Implemented (verified); full requirement Proposed (Phases 2 to 4).

## 25. Human-Review Requirements

Review is mandatory, not optional, for: ambiguous entity mappings below the deterministic threshold; evidence sufficiency failures where the user asserts actuals; conflicting regulatory sources; expired or unverified verifier evidence; financial exposure above a tenant-configured materiality threshold; novel production routes; low-confidence extractions; all policy changes; declaration export. Reviewers must see the full decision context and evidence in one place, and every reviewer action becomes part of the decision trace. Reviewer override rates are a tracked metric (Section 29) because persistently high overrides indicate miscalibrated policies, not bad reviewers. Status: Proposed (Phase 4).

## 26. Compliance and Legal Guardrails

- BorderIQ output is decision support, not legal advice; the UI and exports carry this statement.
- The calculation engine is not represented as a legally certified CBAM liability engine (Out of scope until a formal validation process with qualified reviewers exists; open question OQ-08).
- No autonomous filing: submission to authorities is always a human act outside the platform (FR-030 exports drafts only).
- Regulation text is stored and cited from authoritative sources with provenance; the platform never presents paraphrase as quotation.
- Synthetic and demo data is labelled in every surface where it appears (PP-10).
- Policy activation, the point where regulation interpretation becomes system behaviour, always passes qualified human approval (AIR-010).

## 27. Data Requirements

- **Regulatory corpus**: official CBAM regulation, implementing regulations, guidance, and default-value documents from authoritative EU sources only, with checksums, versions, effective dates, and supersession links (FR-022). Current MVP corpus: three PDFs (described).
- **Enterprise data**: shipments, products, aliases, CN codes, installations, routes, energy and precursor inputs, emission records, supplier declarations, verifier reports. Current MVP: Representative demo data only (8-product catalogue, form input). Target: durable store per [DATA_MODEL.md](DATA_MODEL.md).
- **Reference data**: emission factors and carbon prices with source, version, and validity window. Carbon-price sourcing is OQ-07.
- **Data quality states**: every consumed record carries a state from missing, conflicting, stale, unverified, unsupported, suspicious, or validated; states gate eligibility (FR-013).
- **Units and precision**: quantities in tonnes, intensities in tCO2e per tonne of goods, money in EUR with explicit currency; decimal arithmetic with documented rounding rules per figure class (defined in DATA_MODEL.md; the MVP's floating-point demo engine does not meet this).
- **Retention and deletion**: per NFR-005 and SECURITY.md.

## 28. Integration Requirements

Phase-ordered (see ROADMAP): CSV and Excel import with validation (Phase 3); batch API (Phase 3); SQL and object-storage adapters (Phase 8); SAP, Oracle, Dynamics connectors (Phase 8); verifier-system and document-management integration (Phase 8); carbon-price adapter (Phase 8, pending OQ-07); event backbone for shipment, evidence, regulation, policy, and decision events with idempotency, retry, dead-letter queues, correlation IDs, replay, and schema versioning (Phase 3 minimal queue, thresholds for larger platforms documented in DEPLOYMENT.md). IoT is integration-adjacent but excluded as a direct source of final emissions figures (Section 15).

## 29. Success Criteria

Product-level acceptance for the target platform, measured on evaluation sets and pilots. Targets marked with an asterisk are initial engineering targets to be validated against pilot baselines, not promises.

- **Retrieval**: Recall@5 at or above 0.9* on the golden regulatory-question set; effective-version accuracy at or above 0.98*; citation correctness at or above 0.95*; measurable reranker lift documented.
- **Generation**: unsupported-claim rate at or below 2 percent* on audited samples; structured-output validity at or above 99.5 percent*; abstention accuracy at or above 0.9* on the abstention set.
- **Entity resolution**: deterministic-path share at or above 80 percent* of production mappings; ambiguity routed to review at 100 percent; mapping precision at or above 0.98* on the labelled set.
- **Calculation**: 100 percent golden-test pass rate, zero tolerated unit errors, byte-identical replay.
- **Policy**: effective-date correctness 100 percent on golden cases; reviewer override rate trending downward across releases.
- **Business (pilot)**: median investigation time under 15 minutes* for standard shipments; 100 percent of finalised decisions with complete audit packages; measured reduction in avoidable default-value usage against the pilot baseline.

## 30. Business Metrics

Tracked from pilot onward, against a measured baseline captured during onboarding (no invented baselines): time per shipment investigation; manual-review rate and reduction over time; percentage of shipments with complete evidence packages before filing; avoidable default-value usage detected; potential cost difference identified (reported as identified, not realised, unless confirmed); filing delays avoided; audit-package completeness; analyst hours per reporting cycle; consultant leverage (shipments per analyst) for the consulting channel.

## 31. Risks and Mitigations

| ID | Risk | Impact | Mitigation |
|---|---|---|---|
| R-1 | Regulatory misinterpretation embedded in policies | Wrong filings at scale | Human-approved policy activation (AIR-010); golden effective-date tests; change intelligence (FR-024); legal-review process (OQ-08) |
| R-2 | Retrieval returns plausible but wrong or outdated passages | Confident wrong answers | Metadata scoping and effective-date filters before similarity (FR-023); citation validation; abstention (AIR-004); retrieval evals (Section 29) |
| R-3 | Entity mis-mapping (product or CN code) | Systematically wrong calculations | Deterministic-first resolution, review queues, mapping provenance (FR-011); resolution metrics |
| R-4 | LLM output treated as authoritative figures | Compliance failure | AIR-001 and AIR-002 enforcement; output validation; UI renders figures only from engine outputs |
| R-5 | Demo data mistaken for real capability | Credibility loss with buyers and judges | PP-10 labelling everywhere; this documentation's status labels |
| R-6 | Backend claims unverified against code | Documentation drift | OQ-01: commit the backend; re-verify labelled sections |
| R-7 | Scope creep into multi-regulation platform before CBAM depth exists | Diluted product | Non-goal and phased roadmap; CSRD, EUDR, SEC listed only as far-future direction |
| R-8 | Prompt injection via uploaded or retrieved documents | Tool abuse, data leakage | AIR-006; SECURITY.md controls; least-privilege tools; approval gates |
| R-9 | Vendor or model dependency (OpenAI) | Availability and governance risk | Model-version pinning and eval gates (AIR-007); provider abstraction in LLD; per-tenant provider options as a later option |
| R-10 | Effective-dating complexity underestimated | Silent legal drift | Treat as first-class (PP-3, FR-012, FR-022); dedicated golden tests |
| R-11 | Over-engineering (graph, agents, event platforms) before need | Wasted build, fragility | PP-9; explicit adoption thresholds in DEPLOYMENT.md and AGENTS.md; fail-closed optionality |

## 32. Assumptions

- A-1: The described Flask backend exists and materially matches Section 17.2; discrepancies will be corrected when code is committed (pairs with R-6).
- A-2: Official CBAM documents remain publicly obtainable from authoritative EU sources.
- A-3: Pilot customers can export shipment data as CSV or Excel even where ERP integration is not yet built.
- A-4: Verified emissions records can be obtained from existing carbon-accounting or verifier processes; BorderIQ consumes, not produces, primary measurements.
- A-5: An OpenAI-hosted model is acceptable for pilots under enterprise API terms; stricter residency needs are deferred to OQ-10.
- A-6: The hackathon frontend remains the reference client and will be committed to this repository.

## 33. Dependencies

- OpenAI APIs (LLM and embeddings) or an equivalent provider behind the provider abstraction.
- ChromaDB (current MVP, described); candidate replacement per OQ-03.
- LangChain-compatible document tooling for ingestion (current MVP, described).
- Authoritative EU publication sources for the regulatory corpus.
- For optional phases: a graph database if OQ-04 approves; a managed relational database and object storage for Phase 3.

## 34. Constraints

- Hackathon-stage team and budget; the modular-monolith default (PP-9) is also a resourcing constraint.
- No access to real customer shipment data yet; synthetic data must carry the Representative demo data label until pilots.
- The documentation suite must remain truthful under the labelling convention even when that reads less impressive than competitor marketing.

## 35. Release Phases

Summarised here; authoritative detail with exit criteria in [ROADMAP.md](ROADMAP.md).

| Phase | Name | PRD anchors |
|---|---|---|
| 0 | Current MVP stabilisation | FR-001 to FR-007, R-6 closure |
| 1 | Deterministic shipment-intelligence foundation | FR-015 core, FR-011 basic |
| 2 | Typed tool registry and structured decision object | FR-018, FR-021, FR-020 trace, AIR-001 to AIR-008 enforcement |
| 3 | Persistent enterprise data and bulk ingestion | FR-010, FR-027, NFR-002, NFR-003, NFR-008 |
| 4 | Policy-as-code and human review | FR-012 (with policy), FR-013, FR-014, FR-019, FR-020 full, Section 25 |
| 5 | Hybrid regulatory intelligence and change detection | FR-022, FR-023, FR-024, AIR-003, AIR-004 |
| 6 | Knowledge graph | FR-025 |
| 7 | Decision twin and scenarios | FR-016, FR-026 |
| 8 | Enterprise integrations | FR-028, FR-030 |
| 9 | Multi-tenant enterprise platform | FR-029, NFR-004, NFR-007 |
| 10 | Multi-jurisdiction platform | Beyond-CBAM scope, gated by OQ-11 |

## 36. Open Questions

| ID | Question | Blocking |
|---|---|---|
| OQ-01 | Backend verification: commit the Flask backend so Section 17.2 claims (chunking strategy, models, endpoints, tests, env) can be verified and corrected | Documentation accuracy; Phase 0 exit |
| OQ-02 | Target backend framework: remain on Flask initially or migrate to FastAPI at Phase 2 | LLD service design |
| OQ-03 | First enterprise vector store: retain ChromaDB or adopt Postgres plus pgvector when the relational store lands in Phase 3 | DATA_MODEL, DEPLOYMENT |
| OQ-04 | Is Neo4j an approved optional dependency for Phase 6, or should the graph projection target a different store | KNOWLEDGE_GRAPH |
| OQ-05 | First product line and pilot production routes to model in depth (steel assumed; which routes) | Phase 1 scope |
| OQ-06 | Primary buyer: non-EU exporter, EU importing declarant, or consulting provider | Packaging, tenancy priorities |
| OQ-07 | Carbon-price data: user-supplied per investigation, tenant-configured, or provided via a market-data adapter | FR-015 inputs, Phase 8 |
| OQ-08 | Legal-review process that approves policy activations (who qualifies, what record is kept) | Phase 4 governance |
| OQ-09 | Data-residency and retention requirements of target customers | SECURITY, DEPLOYMENT, NFR-005 |
| OQ-10 | Model-provider constraints for regulated customers (in-VPC or open-weights requirements) | Provider abstraction depth |
| OQ-11 | Which regulations beyond CBAM are genuinely in scope for Phase 10 (CSRD, EUDR, SEC climate are candidates, not commitments) | Long-term architecture |
| OQ-12 | Reviewer materiality threshold defaults (exposure level that forces review) | FR-019, Section 25 |

## 37. Glossary

| Term | Definition |
|---|---|
| CBAM | EU Carbon Border Adjustment Mechanism: carbon pricing applied to imports of covered carbon-intensive goods |
| Embedded emissions | Greenhouse-gas emissions attributed to the production of a good, per CBAM methodology |
| Default value | Commission-published emissions intensity used when actual emissions are not sufficiently demonstrated |
| Actual emissions | Installation-specific emissions determined under a permitted methodology with supporting evidence |
| CN code | Combined Nomenclature code classifying goods for EU customs purposes |
| Production route | The manufacturing pathway of a product (for example blast furnace versus electric arc furnace for steel), which affects emissions and applicable methodology |
| Precursor | An input material whose embedded emissions must be included in the final good's embedded emissions |
| Compliance investigation | The end-to-end evaluation of one shipment or consignment producing a structured decision |
| Structured decision | The machine-validated decision object defined in DECISION_ENGINE.md (FR-018) |
| Decision trace | The recorded inputs, versions, retrievals, tool calls, and reviewer actions behind a decision (FR-020) |
| Evidence package | The set of documents and records supporting a decision's methodology and figures |
| Audit package | The exportable bundle of a decision plus its trace and evidence references |
| Policy evaluation | Deterministic evaluation of versioned, effective-dated policy rules against an investigation |
| Deterministic calculation engine | The versioned, unit-explicit, decimal-safe computation service producing all authoritative figures |
| Tool orchestration | Governed selection and execution of registry tools, with the LLM unable to fabricate outputs |
| Regulatory intelligence | Ingestion, versioning, scoped retrieval, and change analysis of authoritative regulation |
| Effective dating | Binding rules and documents to the periods in which they apply |
| Abstention | An explicit insufficient-evidence outcome instead of a generated answer |
| Human review | The mandatory reviewer workflow for material or uncertain cases |
| Current MVP | The hackathon-stage system per Section 17 labelling |
| Target architecture | The proposed platform defined by this suite, not yet built |
| Representative demo data | Synthetic or hardcoded values used for demonstration, always labelled |
