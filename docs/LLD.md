# BorderIQ Low-Level Design

| Field | Value |
|---|---|
| Title | BorderIQ Low-Level Design (LLD) |
| Version | 1.0 |
| Status | Draft for review |
| Owner | Jahnavi Ralhan |
| Last updated | 2026-08-01 |
| Scope | HOW the current MVP works and HOW the target platform is designed: components, contracts, data flow, failure behaviour, and the migration path between the two. Requirements and rationale live in the [PRD](PRD.md); diagrams live in [ARCHITECTURE](ARCHITECTURE.md). |
| Related documents | [PRD](PRD.md), [ARCHITECTURE](ARCHITECTURE.md), [API_SPEC](API_SPEC.md), [DATA_MODEL](DATA_MODEL.md), [DECISION_ENGINE](DECISION_ENGINE.md), [TOOL_CALLING](TOOL_CALLING.md), [KNOWLEDGE_GRAPH](KNOWLEDGE_GRAPH.md), [SECURITY](SECURITY.md), [DEPLOYMENT](DEPLOYMENT.md), [ROADMAP](ROADMAP.md), [AGENTS](AGENTS.md) |

This document uses the labelling convention defined at the top of the [PRD](PRD.md): Current MVP (verified), Current MVP (described), Representative demo data, Proposed, Out of scope, Open design question. Component profiles carry a Status field using these labels.

## 1. Document Control

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0 | 2026-08-01 | Jahnavi Ralhan | Initial LLD written against PRD v3.0 |

## 2. Scope

In scope: the current MVP implementation as verifiable today; the target logical architecture through ROADMAP Phase 9; component contracts, persistence design, orchestration, error handling, evaluation, testing, and migration. Out of scope: requirement rationale (PRD), rendered diagrams (ARCHITECTURE), field-level entity definitions (DATA_MODEL), endpoint schemas (API_SPEC), the decision lifecycle state machine (DECISION_ENGINE), and per-tool contracts (TOOL_CALLING). Where those documents own a topic, this one links rather than restates.

## 3. Design Principles

The PRD's product principles PP-1 through PP-10 bind this design. Their engineering translation:

1. **Deterministic core, probabilistic edge.** Everything that produces an authoritative figure or an eligibility outcome is deterministic, versioned, and unit-tested (PP-1, AIR-001). The LLM sits at the edges: interpretation, classification, extraction, narration, and tool selection.
2. **Modular monolith first.** One deployable application with strict internal module boundaries (packages with explicit public interfaces), extracted into services only at the thresholds documented in Section 36 and [DEPLOYMENT.md](DEPLOYMENT.md) (PP-9).
3. **Schemas at every boundary.** Typed request, response, tool, event, and decision schemas, validated at the boundary, versioned under NFR-012. Prose is never an interface.
4. **Scope before similarity.** Exact lookups for exact identifiers (products, CN codes); metadata filters (jurisdiction, effective date, sector) before any vector search (FR-023, PP-3).
5. **Fail closed, shape preserved.** Optional subsystems degrade to empty, well-typed responses; the MVP's demo-mode pattern is the template (NFR-009, PP-6).
6. **Every mutation traced.** Correlation IDs from HTTP edge to tool execution to decision record (FR-020, NFR-008).
7. **Versions are data.** Formulas, policies, prompts, models, document versions, and schema versions are recorded per decision so replay is possible (PP-8, AIR-007).

## 4. Current Implementation Architecture

### 4.1 Verified: frontend

A single-page application served as one Flask template: `templates/index.html`, `static/css/styles.css`, `static/js/app.js` (vanilla JavaScript, Bootstrap 5 via CDN). Eight surfaces switched client-side. Verified behaviour:

- API client calls exactly `POST /api/ask`, `GET /api/status`, `GET /api/knowledge-base` with a 20 second timeout per call.
- On any failure, surfaces render labelled demo content and the global badge switches to demo mode; response shapes are preserved (the origin of design principle 5).
- Shipment Analyzer runs a client-side demo rules engine over a hardcoded 8-product catalogue (Representative demo data). A code comment marks the intended replacement with a backend call.
- Session audit log: in-memory array of question and analysis entries with inputs, evidence references, and chunk IDs; volatile.
- All dynamic strings are HTML-escaped before insertion; there is no authentication, storage, or build step.

### 4.2 Described: backend (Current MVP, described; unverified pending OQ-01)

Python and Flask serving JSON. Reported pipeline: official CBAM PDFs loaded via LangChain-compatible PDF tooling, chunked with metadata (source file, page number, document type, chunk ID), embedded with an OpenAI embedding model, persisted in a local ChromaDB collection; `POST /api/ask` embeds the question, retrieves top-k chunks, and prompts an OpenAI chat model to answer only from the retrieved context, returning the answer plus source metadata. `GET /api/status` reports corpus and connectivity counts. `GET /api/knowledge-base` lists indexed documents. A Python product catalogue and rules engine mirroring the frontend demo calculation have been described; whether an analysis endpoint is exposed is unverified.

Known unknowns tracked under OQ-01: exact chunking strategy (page-wise versus recursive character splitting has been described inconsistently), exact model identifiers, server-side analysis endpoint existence, tests, dependency pins, environment variables.

### 4.3 Current data flow (both layers)

Question path: browser -> `POST /api/ask` -> retrieval over ChromaDB -> LLM narration with citations -> browser renders answer and evidence cards -> session audit entry. Analysis path (MVP): browser-only; product select resolves demo catalogue entry -> client-side arithmetic -> rendered result -> session audit entry. No durable write exists anywhere in the current system.

## 5. Target Logical Architecture

One application (modular monolith) with the following internal modules, plus external stores. Rendered diagrams: [ARCHITECTURE.md](ARCHITECTURE.md) sections 3 and 4.

```text
Client (web SPA)
   |
API layer (auth, validation, correlation IDs, rate limits)
   |
Application services
   |-- Investigation Orchestrator ----- Tool Registry and Executor
   |        |                                |
   |        |                                +-- Retrieval Service (hybrid, scoped)
   |        |                                +-- Entity Resolution Service
   |        |                                +-- Calculation Engine
   |        |                                +-- Policy Engine
   |        |                                +-- Risk Engine
   |        |                                +-- Scenario Engine
   |        +-- Decision Assembler
   |-- Review Service
   |-- Ingestion Workers (regulatory corpus; enterprise imports)
   |-- Change Intelligence (Phase 5)
   |
Stores: relational DB | object storage | vector index | cache | queue | optional graph
Cross-cutting: identity, audit trace store, observability, secrets
```

Module boundaries are package boundaries with typed interfaces; every arrow above is an in-process call in Phases 1 to 4 and a candidate service boundary later (Section 36).

## 6. Component Inventory

| # | Component | Kind | Status | Detailed in |
|---|---|---|---|---|
| C-01 | Web frontend (SPA) | Client | Current MVP (verified); evolves | Section 7 |
| C-02 | API layer | Boundary | Current MVP (described, 3 routes); target Proposed | Section 8, [API_SPEC](API_SPEC.md) |
| C-03 | Investigation Orchestrator | Service | Proposed (Phase 2) | Section 10, [DECISION_ENGINE](DECISION_ENGINE.md) |
| C-04 | Tool Registry and Executor | Service | Proposed (Phase 2) | Section 11, [TOOL_CALLING](TOOL_CALLING.md) |
| C-05 | Calculation Engine | Deterministic service | Demo only today; Proposed (Phase 1) | Section 12 |
| C-06 | Policy Engine | Deterministic service | Proposed (Phase 4) | Section 13 |
| C-07 | Risk Engine | Deterministic service | Demo heuristic today; Proposed (Phase 2) | Section 14 |
| C-08 | Scenario Engine | Deterministic service | Proposed (Phase 7) | Section 15 |
| C-09 | Decision Assembler | Service | Proposed (Phase 2) | Section 16 |
| C-10 | Retrieval Service | Service | Current MVP (described, dense top-k); target hybrid Proposed (Phase 5) | Section 17 |
| C-11 | Regulatory Ingestion Pipeline | Workers | Current MVP (described, minimal); target Proposed (Phase 5) | Sections 18, 19 |
| C-12 | Entity Resolution Service | Service | Demo lookup today; Proposed (Phases 1, 3) | Section 20 |
| C-13 | Relational database | Store | Absent today; Proposed (Phase 3) | Section 21, [DATA_MODEL](DATA_MODEL.md) |
| C-14 | Object storage | Store | Absent today; Proposed (Phase 3) | Section 21 |
| C-15 | Vector index | Store | Current MVP (described, local ChromaDB); target per OQ-03 | Sections 17, 21 |
| C-16 | Knowledge graph | Optional store | Proposed (Phase 6), fail closed | Section 22, [KNOWLEDGE_GRAPH](KNOWLEDGE_GRAPH.md) |
| C-17 | Queue and event backbone | Infrastructure | Absent today; Proposed (Phase 3 minimal) | Section 23 |
| C-18 | Cache | Infrastructure | Absent today; Proposed (Phase 3) | Section 24 |
| C-19 | Review Service | Service | Absent today; Proposed (Phase 4) | Section 10.3 |
| C-20 | Audit and trace store | Store | Session-only today (verified); Proposed durable (Phase 2) | Section 30 |
| C-21 | Identity provider integration | Boundary | Absent today; Proposed (Phase 3) | Section 31, [SECURITY](SECURITY.md) |
| C-22 | Observability stack | Cross-cutting | Absent today; Proposed (Phase 3) | Section 32 |
| C-23 | Change Intelligence | Service | Proposed (Phase 5) | Section 18.3 |

Component profiles below use a standard block: Responsibility, Inputs, Outputs, Dependencies, State, Failure modes, Scaling, Observability, Security boundary, Status.

## 7. Frontend Design

| Attribute | Value |
|---|---|
| Responsibility | Render all product surfaces; call the API; never compute authoritative figures once the backend analysis path exists |
| Inputs | User interaction; JSON API responses |
| Outputs | API requests; rendered decisions, evidence, traces |
| Dependencies | C-02 API layer; CDN assets (to be vendored, Section 31) |
| State | Client session state only; no browser persistence of enterprise data |
| Failure modes | Backend unreachable (handled: labelled demo mode); malformed response (handled: shape checks before render) |
| Scaling | Static assets; scales with any web tier |
| Observability | Client errors reported to the API layer (Proposed, Phase 3) |
| Security boundary | Untrusted client; all validation re-done server-side |
| Status | Current MVP (verified); target changes below |

Target changes, in order: (1) replace the client-side demo rules engine with the investigation API and delete the local catalogue (Phase 1 to 2); (2) remove the hardcoded confidence figure and render the five confidence dimensions from the decision object (AIR-005, Phase 2); (3) add the Review Workspace surface (Phase 4); (4) render decision traces and audit-package export (Phases 2 to 4); (5) vendor third-party assets and add auth/session handling (Phase 3). The demo-mode degradation pattern is retained permanently as the reference implementation of NFR-009.

## 8. API Layer

| Attribute | Value |
|---|---|
| Responsibility | HTTP boundary: authentication, request validation, correlation-ID issuance, rate limiting, routing to application services, response shaping |
| Inputs | HTTPS requests |
| Outputs | Validated typed requests to services; JSON responses per [API_SPEC](API_SPEC.md) |
| Dependencies | C-21 identity; application services |
| State | Stateless |
| Failure modes | Validation failure (400 with field errors); auth failure (401/403); downstream timeout (504 with correlation ID); overload (429) |
| Scaling | Horizontal behind a load balancer |
| Observability | Access logs, latency and error-rate metrics per route, correlation IDs |
| Security boundary | The trust boundary between untrusted clients and the application; see [SECURITY.md](SECURITY.md) |
| Status | Current MVP (described): 3 unauthenticated routes. Target: Proposed, Phase 2 onward |

Current routes and their exact shapes are documented in [API_SPEC.md](API_SPEC.md) alongside the target v1 API. Framework: Flask today; the recommendation is to adopt FastAPI at Phase 2 for native request/response models and OpenAPI generation, with Flask retained if migration cost outweighs benefit at that point. Decision deferred: OQ-02. All target design in this document is framework-neutral.

## 9. Application Services

Application services are thin coordinators: they own transactions and call modules; they contain no business math and no prompts. Service list: Investigation service (wraps the orchestrator per request or batch item), Import service (CSV/Excel/batch intake, row validation, job creation), Review service (Section 10.3), Knowledge service (document registry queries for the Knowledge Centre), Admin service (tenant configuration, mappings, policies; Phase 4 onward). Each service method: validates a typed request, opens a unit of work, invokes domain modules, writes trace records, emits events, returns a typed response. Status: Proposed (Phase 2 onward), except the trivial current handlers (described).

## 10. Orchestration Layer

### 10.1 Investigation Orchestrator (C-03)

| Attribute | Value |
|---|---|
| Responsibility | Drive one compliance investigation through the 16-stage lifecycle defined in [DECISION_ENGINE.md](DECISION_ENGINE.md): intake to audit packaging. Owns stage ordering, short-circuiting, and abstention |
| Inputs | Validated investigation request (shipment reference or inline payload, requested methodology if any, tenant and user context) |
| Outputs | Structured decision object (FR-018); trace records per stage; events |
| Dependencies | C-04 tools, C-05 to C-09 engines, C-10 retrieval, C-12 resolution, C-20 trace store |
| State | Per-investigation state machine persisted at each transition (resumable) |
| Failure modes | Stage failure with retryable error (retry per Section 28); stage failure terminal (decision status blocked or insufficient evidence, never a silent partial); LLM planner unavailable (fall back to the deterministic default plan) |
| Scaling | Horizontal workers consuming an investigation queue (Phase 3) |
| Observability | Stage duration and outcome metrics; full trace per FR-020 |
| Security boundary | Executes with tenant-scoped credentials; may only reach tools the caller's role permits |
| Status | Proposed (Phase 2) |

Design rule: the default execution plan is a deterministic, hard-coded stage sequence. The LLM-assisted planner (the Investigation Planner agent in [AGENTS.md](AGENTS.md)) may only reorder or skip optional stages within an allowlist and is itself optional; with it disabled, investigations still run. This keeps the orchestrator a workflow, not an agent (PP-9, technical decision rule 7).

### 10.2 Regulatory Q&A flow

The Compliance Assistant path is a two-stage subset: scoped retrieval (C-10) then narration with citation validation (Section 29). It shares tracing and abstention with investigations but never invokes calculation or policy engines. Status: Current MVP (described) for the unscoped version; scoped version Proposed (Phase 5).

### 10.3 Review Service (C-19)

| Attribute | Value |
|---|---|
| Responsibility | Create review tasks from routing rules (Section 25 of the PRD); present decision context; record approve, reject, request-information actions; finalise or re-route decisions |
| Inputs | Decisions with status review required; reviewer actions |
| Outputs | Finalised decision states; reviewer trail entries; events |
| Dependencies | C-13 relational store, C-20 trace store, C-21 identity |
| State | Review tasks (durable) |
| Failure modes | Reviewer conflict (optimistic locking, second writer gets a conflict error); orphaned tasks (escalation timers) |
| Scaling | Trivial (human-rate workload) |
| Observability | Queue depth, time-to-review, override-rate metrics |
| Security boundary | Reviewer role required; approvals recorded with identity |
| Status | Proposed (Phase 4) |

## 11. Tool Registry (C-04)

Authoritative contract in [TOOL_CALLING.md](TOOL_CALLING.md). LLD summary: a registry of versioned tool definitions (name, owner, version, JSON-Schema input and output, timeout, idempotency class, permission, approval requirement, deterministic-or-probabilistic flag) plus an executor that (1) validates inputs against the schema before execution, (2) executes with per-tool timeout and retry class, (3) validates outputs against the schema, (4) writes a ToolExecution trace, (5) enforces permissions and approval gates before side-effectful tools. The LLM selects tools by name from the registry and receives only validated outputs; it cannot register, mutate, or simulate tools (AIR-001, FR-021). Failure modes: schema-invalid input (rejected, traced), timeout (retry class or abstain), output invalid (treated as tool failure, never passed to the model as truth). Status: Proposed (Phase 2).

## 12. Calculation Engine (C-05)

| Attribute | Value |
|---|---|
| Responsibility | All authoritative arithmetic: direct, indirect, and precursor embedded emissions; default and actual scenarios; exposure and difference estimation (FR-015) |
| Inputs | Typed calculation requests: quantities, intensities, factors, prices, each with value, unit, source reference, and validity metadata |
| Outputs | CalculationRun records: results with units, intermediate values, formula version, input lineage |
| Dependencies | None at runtime beyond its own versioned formula and factor tables (pure by design); factor retrieval happens before invocation via tools |
| State | Stateless functions; CalculationRun persisted by the caller |
| Failure modes | Unit mismatch (reject, never coerce); missing input (reject with named field); factor validity window violation (reject) |
| Scaling | CPU-trivial; scales with the application |
| Observability | Run counts, rejection reasons, golden-test status in CI |
| Security boundary | No I/O, no network; cannot be reached except through typed requests |
| Status | Demo only today (frontend, floating point); Proposed (Phase 1 core, Phase 2 hardened) |

Implementation rules: Python `decimal.Decimal` end to end with per-figure-class rounding rules defined in [DATA_MODEL.md](DATA_MODEL.md); units as an explicit enum with checked conversions (no implicit coercion); every formula is a pure function registered with a semantic version; golden fixtures per formula version; property tests for boundary conditions (zero quantity, intensity above default, missing precursors). The MVP demo formulas (defaultTotal, actualTotal, avoided, savings, ratio) become the Phase 1 seed fixtures after re-derivation in Decimal.

## 13. Policy Engine (C-06)

| Attribute | Value |
|---|---|
| Responsibility | Deterministic evaluation of versioned, effective-dated policy rules: methodology eligibility, evidence requirements, approval thresholds (FR-012, FR-013, FR-014) |
| Inputs | Investigation context: resolved entities, reporting period, evidence states, calculation outputs where thresholds apply |
| Outputs | Policy outcomes: eligible or ineligible with rule citations, required-evidence lists, routing directives |
| Dependencies | Policy store (C-13); no LLM |
| State | Policies as data: PolicyDefinition and PolicyVersion entities (DATA_MODEL) |
| Failure modes | No applicable policy version for the period (decision blocked with reason, never a fallback guess); conflicting policies (conflict outcome, routed to review) |
| Scaling | In-memory evaluation of loaded policy sets; trivial |
| Observability | Policy hit counts, conflict rate, override rate |
| Security boundary | Policy activation requires the approval workflow (AIR-010); the engine only reads activated versions |
| Status | Proposed (Phase 4) |

Representation: declarative rules (condition tree over typed context fields) stored as data with jurisdiction, product and route applicability, effective dates, and provenance links to the regulatory fragments they derive from. Rules are authored by humans; AI-drafted rules enter as proposals with DERIVED_FROM provenance and are inert until approved. No general-purpose rule DSL execution of untrusted input.

## 14. Risk Engine (C-07)

Deterministic scoring over declared inputs: exposure magnitude, evidence completeness, mapping confidence, data anomalies, novelty flags. Output: risk class plus the contributing factors (explainable by construction). Thresholds are versioned configuration, not code. The MVP's ratio-band heuristic (0.6 and 0.9 cut points, unsourced) is acknowledged demo behaviour and is replaced, not ported. Failure mode: missing factor inputs degrade to the most conservative class with a stated reason. Status: Proposed (Phase 2 basic, Phase 4 policy-linked).

## 15. Scenario Engine (C-08)

Executes controlled what-if runs against a decision twin version (FR-016, FR-026): it clones the twin, applies a declared assumption delta (price, route, energy mix, evidence state, regulation version), re-runs calculation and policy engines, and emits a Scenario record referencing the base decision. Invariants: scenarios never mutate official decisions; every scenario stores its assumption set; scenario results are labelled as hypothetical in every surface. Status: Proposed (Phase 7).

## 16. Decision Assembler (C-09)

Builds the structured decision object (FR-018) from stage outputs: status, methodology, eligibility, figures (from CalculationRuns only), assumptions, missing evidence, risk, supporting sources, applied policy versions, required actions, reviewer state, the five confidence dimensions (AIR-005), and provenance. Validates the assembled object against the decision schema; on validation failure the investigation fails closed (no partial decision is ever rendered). Narration is generated after assembly, from the object, with citation checks (Section 29); the narrative is stored as a field of the decision, never the other way round. Status: Proposed (Phase 2). Full object schema: [DECISION_ENGINE.md](DECISION_ENGINE.md).

## 17. Retrieval and Regulatory-Intelligence Layer (C-10)

| Attribute | Value |
|---|---|
| Responsibility | Return the correct, currently effective regulatory passages for a scoped query, with scores and provenance; abstain when evidence is insufficient (FR-023) |
| Inputs | Scoped query: text plus filters (jurisdiction, regulation family, effective date or reporting period, CN code, category, route, methodology, emissions type, document version) |
| Outputs | Ranked passages with fragment IDs, document versions, pages, scores; RetrievalTrace records; sufficiency verdict |
| Dependencies | C-15 vector index, lexical index, document registry (C-13), optional reranker |
| State | Indexes derived from the registry; rebuildable |
| Failure modes | Reranker down (skip, flag in trace, NFR-009); empty scoped result (abstain, AIR-004); version filter conflict (error, never silent widening of scope) |
| Scaling | Read-heavy; index sharding or a managed index at volume (OQ-03) |
| Observability | Recall/precision eval dashboards (Section 33); per-query latency, filter hit rates |
| Security boundary | Retrieval is tenant- and corpus-scoped; retrieved text is untrusted data downstream (AIR-006) |
| Status | Current MVP (described): unscoped dense top-k over one collection. Target: Proposed (Phase 5) |

Target pipeline per query: (1) mandatory scope resolution (at minimum jurisdiction and effective date; investigation queries add CN code, route, methodology); (2) metadata pre-filter on the fragment store; (3) dense retrieval and lexical BM25 retrieval in parallel over the filtered set; (4) score fusion; (5) optional cross-encoder reranking; (6) citation validation (fragment exists, version effective, page matches); (7) sufficiency check against thresholds; below threshold, return abstention rather than the best bad passage.

## 18. Ingestion Pipeline (C-11)

### 18.1 Regulatory corpus ingestion (FR-022)

Stages: source acquisition from the authoritative allowlist only; immutable raw storage in object storage with checksum and provenance record; document registry entry (RegulatoryDocument, RegulationVersion with effective dates and supersession links); structural parsing (Section 19); fragment metadata enrichment; embedding and index write; verification pass (fragment counts, spot citations). Idempotent per checksum: re-ingesting an identical file is a no-op; a changed file creates a new version, never overwrites. Status: Current MVP (described) implements a minimal loader-chunk-embed path; target Proposed (Phase 5).

### 18.2 Enterprise data ingestion (FR-010, FR-027)

CSV and Excel imports run as jobs: schema validation, per-row canonicalisation, entity resolution (Section 20), per-row accept or reject with reasons, durable persistence, and an import report. Batch API follows the same job path. Idempotency by import fingerprint plus client idempotency key (Section 25). Status: Proposed (Phase 3).

### 18.3 Change Intelligence (C-23, FR-024)

On new regulation version: structural diff against the superseded version; change classification per article, annex, and table; impact mapping via fragment metadata (and graph traversal when available, [KNOWLEDGE_GRAPH.md](KNOWLEDGE_GRAPH.md)) to CN codes, routes, methodologies, active policies, and open decisions; creation of a human review task with the impact report. Policy updates activate only through the Phase 4 approval workflow; affected finalised decisions are flagged, and open investigations re-run. No autonomous legal updates. Status: Proposed (Phase 5).

## 19. Structural Document Parsing

Regulatory documents are parsed into a fragment tree: document -> version -> (article | annex | table | section) -> fragment. Each fragment stores: fragment ID, parent path, page span, text, table payload where applicable (default-value tables become typed rows, not prose), and enrichment metadata (regulation family, jurisdiction, CN codes referenced, methodology tags, emissions type, effective window). Page-level chunking (the MVP approach as described) remains the fallback for documents that defeat structural parsing; fragments always retain page numbers so citations stay page-addressable either way. Parser outputs are validated by fragment-count and table-shape checks before indexing. Status: Proposed (Phase 5).

## 20. Entity-Resolution Layer (C-12)

| Attribute | Value |
|---|---|
| Responsibility | Resolve external identifiers (ERP product names, supplier names, CN codes, installation IDs) to canonical entities with provenance and confidence (FR-011) |
| Inputs | Raw identifier plus context (tenant, source system, category hints) |
| Outputs | Canonical ID with method (exact, alias, fuzzy), confidence, provenance; or a review-queue item |
| Dependencies | Canonical and alias tables (C-13); review service for ambiguity |
| State | Alias tables and mapping history (durable, tenant-scoped) |
| Failure modes | Ambiguity (never auto-resolve; queue); conflicting aliases (conflict state, queue); unknown entity (unsupported-case outcome) |
| Scaling | Indexed lookups; fuzzy stage bounded by candidate pre-filtering |
| Observability | Deterministic-path share, ambiguity rate, review turnaround (PRD Section 29 metrics) |
| Security boundary | Tenant-scoped aliases; cross-tenant mapping reuse is opt-in reference data only |
| Status | Demo lookup today (8-row catalogue); Proposed (Phase 1 exact and alias, Phase 3 full with review queue) |

Resolution order is fixed: exact canonical match, then alias table, then constrained fuzzy match (normalised string similarity within the same category and tenant) producing a confidence score, then review queue. Vector similarity is not used for exact identifier lookup (technical decision rule 5). Every accepted mapping writes provenance (who or what mapped it, method, evidence) and becomes a deterministic alias for the future, so the system's deterministic-path share rises with use. Data-quality states (missing, conflicting, stale, unverified, unsupported, suspicious, validated) are assigned here and gate downstream eligibility (FR-013).

## 21. Data-Persistence Layer

Owned entity-by-entity in [DATA_MODEL.md](DATA_MODEL.md); LLD rules:

- **Relational database (C-13)** is the system of record for all transactional entities (shipments, decisions, policies, mappings, reviews, traces metadata). Recommendation: PostgreSQL from Phase 3. Migrations are versioned and forward-only in deployment (Section 35).
- **Object storage (C-14)** holds immutable artefacts: raw regulatory files, uploaded evidence documents, exported audit packages; addressed by checksum, tenant-prefixed.
- **Vector index (C-15)**: local ChromaDB today (described). At Phase 3, either retain ChromaDB server-side or consolidate on Postgres plus pgvector to reduce operational surface; recommendation is pgvector at MVP-scale corpus sizes, revisited if corpus or query volume outgrows it. Decision: OQ-03. The retrieval service isolates this choice behind one interface.
- **Cache (C-18)** and **queue (C-17)**: Section 23 and 24.
- **Graph (C-16)**: optional projection, never the system of record ([KNOWLEDGE_GRAPH.md](KNOWLEDGE_GRAPH.md)).

Tenancy: `tenant_id` on every transactional row, tenant-scoped vector namespaces and storage prefixes from the first multi-tenant phase; enforced in queries by construction (repository layer refuses tenant-less access) and verified by NFR-007 tests.

## 22. Knowledge-Graph Integration (C-16)

Summary only; full design in [KNOWLEDGE_GRAPH.md](KNOWLEDGE_GRAPH.md). The graph is a projection built from relational truth for relationship traversal (regulatory impact propagation, similarity, multi-hop applicability). It is optional and fail-closed: a `graph_enabled` check gates every graph feature; when disabled or unreachable, callers receive the same response shapes with empty graph sections and core investigation is unaffected (NFR-009). Graph writes happen only in the projection builder, never in request paths. Store choice pending OQ-04. Status: Proposed (Phase 6).

## 23. Event Architecture (C-17)

Phase 3 introduces the minimal reliable backbone: a Postgres-backed job and outbox pattern (transactional outbox written with the owning transaction; a dispatcher publishes to a lightweight queue such as Redis Streams or an equivalent managed queue). Event set as in the PRD Section 28 (shipment.created, shipment.updated, supplier.declaration.received, verification.report.received, emission.record.updated, regulation.published, regulation.superseded, policy.approved, decision.completed, decision.review_required). Every event: schema-versioned envelope, correlation ID, idempotency key, retry with backoff, dead-letter queue after bounded attempts, replay from the outbox. A larger event platform (Kafka-class) is adopted only past the thresholds documented in [DEPLOYMENT.md](DEPLOYMENT.md) (sustained multi-producer fan-out or cross-service streaming; technical decision rule 2). Status: Proposed (Phase 3).

## 24. Cache Strategy

Cache what is stable and expensive, never what gates correctness: (1) scoped-retrieval results keyed by (corpus version, scope filter hash, query hash), invalidated by corpus version change; (2) status and registry reads with short TTLs; (3) reranker outputs alongside retrieval entries. Never cached: policy evaluations, calculations, decisions (cheap, and correctness-critical), or anything crossing tenants without the tenant in the key. Cache failures degrade to recompute (fail open for reads is acceptable here because the source of truth is authoritative; this is the one deliberate exception to fail-closed, and it never applies to permissions). Status: Proposed (Phase 3).

## 25. Idempotency

Client-supplied idempotency keys on all mutating APIs (imports, investigation creation, review actions, tool execution API), stored with request fingerprint and response; replays return the stored response. Tool executions carry an idempotency class in the registry (idempotent, at-most-once, requires-key). Ingestion is checksum-idempotent (Section 18.1). Event consumers are idempotent by (event ID, consumer) dedup records. Status: Proposed (Phase 2 onward).

## 26. Concurrency

Investigations are single-writer per investigation ID (queue partitioning by ID). Decision finalisation and review actions use optimistic locking (version column); conflicting writers receive a typed conflict error. Batch imports parallelise per row after file-level validation, bounded by worker pool size. Policy activation is serialised per policy family to prevent interleaved effective-date windows. No shared mutable in-process state across requests beyond caches. Status: Proposed (Phase 3).

## 27. Error Handling

Uniform error envelope at the API (code, message, correlation ID, field errors where applicable; see [API_SPEC.md](API_SPEC.md)). Internal taxonomy: ValidationError (caller-fixable), DependencyError (retryable), DomainError (business outcome such as insufficient evidence, expressed as decision status, not HTTP failure), IntegrityError (bug; alert). Rule: business impossibility is a decision outcome, not an exception; exceptions are reserved for defects and dependency failures. All errors carry correlation IDs into logs and traces. Status: pattern Proposed (Phase 2); the MVP's labelled demo fallback is the current, verified, client-side ancestor.

## 28. Retry and Circuit Breaking

Retry only DependencyErrors, with exponential backoff and jitter, bounded attempts, and per-dependency budgets (LLM, embeddings, reranker, graph, queues). Circuit breakers per external dependency: open after failure-rate threshold, half-open probes, and a defined degraded behaviour per dependency (LLM narration unavailable: decisions still assemble with narrative marked pending; reranker: skipped; graph: empty sections; vector index: investigation blocked with a retryable status, since retrieval is core). Timeouts are set per tool in the registry and per dependency in configuration; no unbounded call exists. Status: Proposed (Phase 2 onward).

## 29. Structured Outputs

All LLM interactions that feed decisions use schema-constrained outputs (provider structured-output or function-calling mode) validated with typed models; invalid outputs are retried once with the validation error appended, then treated as failure or abstention (AIR-002). Narration passes a citation post-check: every regulatory claim sentence must reference a retrieved fragment ID present in the trace; uncited claims are removed, and if removal guts the narrative, the response abstains (AIR-003). Numeric guard: figure fields in narration are injected from the decision object by the renderer, not generated; a validator rejects narratives introducing numerals not present in the decision (AIR-001). Prompt templates are versioned artefacts recorded per trace (AIR-007). Raw model reasoning is never exposed (AIR-008). Status: Proposed (Phase 2).

## 30. Audit Logging (C-20)

Two layers. (1) Decision traces (FR-020): per investigation, an append-only record set covering request ID, tenant, user, shipment, timestamps, input snapshot, document versions, retrieved passages and scores, reranker output, tool calls with inputs and outputs, policy and engine versions, model and prompt versions, structured decision, reviewer actions, final status. Stored relationally with large payloads in object storage by reference; retention per NFR-005. (2) Security audit log: authentication, authorisation denials, policy activations, exports, admin actions ([SECURITY.md](SECURITY.md)). Replay reads only the trace, never live systems, and must reproduce figures byte-identically (NFR-001). Status: session-only seed verified in the MVP frontend; durable design Proposed (Phase 2 trace, Phase 4 full package).

## 31. Security Controls

Owned by [SECURITY.md](SECURITY.md); the LLD-level invariants: authentication and RBAC at the API layer from the first durable-data phase (no anonymous mutation once storage exists); tenant scoping enforced in the repository layer by construction; retrieved documents and tool outputs handled as untrusted data with instruction-isolation in prompts (AIR-006); tools least-privileged with side-effectful tools behind approval gates; upload scanning and type validation before ingestion; secrets from a manager, never in code or images; frontend third-party assets vendored and integrity-pinned by Phase 3; encryption in transit everywhere and at rest for all stores. Current MVP gaps (no auth, volatile audit, CDN assets, client-side engine) are acknowledged and closed on the phases stated in SECURITY.md.

## 32. Observability

Structured JSON logs with correlation IDs end to end; RED metrics per route and per tool (rate, errors, duration); domain metrics: investigations by status, abstention rate, review queue depth and latency, override rate, deterministic-mapping share, retrieval eval scores from the offline harness, cost per investigation (token and search budgets, NFR-010); traces sampling every investigation with stage spans. Dashboards and alerts defined per environment in [DEPLOYMENT.md](DEPLOYMENT.md). Status: Proposed (Phase 3).

## 33. Evaluation

The evaluation harness is versioned with the code and gates releases (AIR-007). Suites mirror PRD Section 29: retrieval (golden question-to-fragment sets: Recall@k, Precision@k, MRR, nDCG, metadata-filter accuracy, reranker lift, effective-version accuracy, citation correctness), generation (faithfulness, answer relevance, citation completeness, unsupported-claim rate on audited samples, abstention accuracy, structured-output validity), entity resolution (exact-match rate, precision, recall, ambiguity rate, review rate on a labelled mapping set), calculation (golden tests, unit correctness, boundary coverage, byte-identical reproducibility, regression detection), and policy (conformance, effective-date correctness on golden period cases, conflict detection, override-rate trend). Model, prompt, retrieval, or corpus changes must pass the relevant suites before rollout; results are stored per version for trend analysis. Status: Proposed (Phase 2 skeleton, Phase 5 full retrieval suites).

## 34. Testing

- Unit: engines and resolvers, pure and exhaustive (calculation golden fixtures per formula version; policy golden period cases; resolver tables).
- Contract: API and event schemas against published versions in CI (NFR-012); tool inputs and outputs against registry schemas.
- Integration: orchestrator stage flows with faked dependencies; ingestion checksum idempotency; outbox delivery and consumer dedup.
- Isolation: NFR-007 cross-tenant test suite at API, retrieval, storage, and graph layers.
- Degradation: chaos-style tests disabling each optional dependency, asserting shape-preserving responses (NFR-009).
- Replay: stored decisions re-executed and diffed byte-for-byte (NFR-001).
- Security: injection corpus tests (AIR-006), authz matrix tests.
- Frontend: response-shape rendering tests and demo-mode fallback tests.

Current MVP test coverage: none verified anywhere; the described backend's tests are unverified (OQ-01). Phase 0 exit requires the seed suites for whatever code is committed.

## 35. Deployment Topology

Owned by [DEPLOYMENT.md](DEPLOYMENT.md). LLD constraints: one containerised application image (modular monolith) plus worker processes sharing the image; managed PostgreSQL; object storage; queue; secrets manager; observability stack; no Kubernetes requirement before the thresholds in DEPLOYMENT.md (a container service or small VM fleet suffices earlier; technical decision rule 3); database migrations run as a release step, backward-compatible within a minor version.

## 36. Scaling Stages

| Stage | Trigger | Change |
|---|---|---|
| S0 | Current MVP | Single process, local stores, demo data |
| S1 | Phase 3 (durable data, pilots) | Monolith plus worker pool, Postgres, object storage, minimal queue, cache |
| S2 | Batch volume or retrieval load saturates workers | Scale workers horizontally; move vector index to a managed or dedicated deployment per OQ-03 |
| S3 | Team or domain ownership boundaries harden | Extract the first services along module seams, in order of independence: ingestion workers, retrieval service, calculation service |
| S4 | Multi-producer streaming or cross-service fan-out sustained | Adopt a Kafka-class event platform; keep the outbox pattern at producers |
| S5 | Multi-region or residency-driven isolation (OQ-09) | Regional deployments with tenant pinning |

Extraction rule: a module is extracted only when its release cadence, scaling profile, or ownership demonstrably diverges from the monolith, and its interface has been stable for a full phase.

## 37. Cost Controls

Per-investigation budgets: LLM tokens, retrieval queries, reranker calls; exceeded budgets cap the feature (skip reranker, shorten narration) and log, never silently overspend (NFR-010). Caching per Section 24. Model tiering: the cheapest adequate model per task (classification and extraction on small models; narration on the standard model), with tiers recorded per trace. Batch imports process asynchronously to smooth spend. Cost per investigation is a first-class metric (Section 32).

## 38. Graceful Degradation

Degradation matrix (all shape-preserving, NFR-009):

| Dependency down | Behaviour |
|---|---|
| LLM narration | Decision assembles; narrative field marked pending; Q&A returns explicit unavailability |
| Reranker | Retrieval proceeds without reranking; trace flags it |
| Knowledge graph | Graph sections empty; features hidden by capability flag |
| Vector index | Q&A and grounding unavailable: investigations block with retryable status (retrieval is core, not optional) |
| Queue | Synchronous fallbacks for single investigations; batch imports pause with visible status |
| Cache | Recompute (Section 24) |
| Backend entirely (MVP pattern) | Frontend demo mode, labelled |

## 39. Migration from Current MVP

Ordered, each step shippable:

1. **Commit and verify the backend** (OQ-01, Phase 0 exit): reconcile Section 4.2 against code; pin dependencies; add seed tests; vendor frontend assets.
2. **Server-side calculation** (Phase 1): implement the Calculation Engine (Decimal, units, versions, golden tests); expose the investigation analysis endpoint; switch the frontend to it and delete the client-side engine and hardcoded confidence (closes two MVP violations).
3. **Structured decision and tools** (Phase 2): decision schema, tool registry with the initial read-only and compute tools, durable decision traces, structured-output enforcement, evaluation skeleton.
4. **Durable data** (Phase 3): Postgres, object storage, imports, queue, auth, observability; ChromaDB decision per OQ-03 executed here.
5. **Policy and review** (Phase 4): policy store and engine, evidence sufficiency, review service and workspace, full audit packages.
6. **Regulatory intelligence** (Phase 5): document registry, structural parsing, hybrid scoped retrieval, abstention, change intelligence.
7. Phases 6 to 9 per [ROADMAP.md](ROADMAP.md).

No step requires rewriting a previous one; each replaces a labelled demo element with its target counterpart.

## 40. Technical Risks

| ID | Risk | Mitigation |
|---|---|---|
| TR-1 | Structural parsing of legal PDFs is brittle | Page-level fallback retained (Section 19); parser validation checks; corpus grows incrementally |
| TR-2 | Effective-date logic errors | Golden period tests as release gates (Section 33); version binding recorded per decision |
| TR-3 | Hybrid retrieval complexity delays Phase 5 | Ship scoped dense retrieval first; add lexical and reranking behind flags with measured lift |
| TR-4 | Decimal and unit retrofit breaks demo parity | Re-derive MVP formulas as Phase 1 golden fixtures before switching the frontend |
| TR-5 | Tool registry becomes a bottleneck abstraction | Start with the minimal tool set actually invoked by Phase 2 flows; grow by need |
| TR-6 | Outbox dispatcher lag under load | Monitor queue depth (Section 32); S4 threshold defined before it hurts |
| TR-7 | Framework migration churn (OQ-02) | All designs framework-neutral; decision deferred to Phase 2 planning |
| TR-8 | Graph projection drift from relational truth | Rebuildable projection, checksum comparisons, and it is never the system of record |

## 41. Open Design Questions

Inherited from PRD Section 36 and owned there: OQ-01 (backend verification), OQ-02 (Flask versus FastAPI at Phase 2; LLD recommendation: FastAPI for typed boundaries), OQ-03 (ChromaDB versus Postgres plus pgvector at Phase 3; LLD recommendation: pgvector at current corpus scale), OQ-04 (graph store approval), OQ-07 (carbon-price sourcing affects Calculation Engine inputs), OQ-09 (residency drives S5), OQ-10 (model-provider constraints drive provider abstraction depth). New LLD-level questions: OQ-13: does the Phase 2 decision schema version independently of the API version or with it (recommendation: independently, with a compatibility map); OQ-14: minimum viable reranker (cross-encoder self-hosted versus provider reranking API) for Phase 5 cost targets.
