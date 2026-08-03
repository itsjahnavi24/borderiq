# BorderIQ Roadmap

| Field | Value |
|---|---|
| Title | BorderIQ Roadmap |
| Version | 1.0 |
| Status | Draft for review |
| Owner | Jahnavi Ralhan |
| Last updated | 2026-08-01 |
| Scope | Delivery phases 0 to 10 with objectives, capabilities, dependencies, risks, acceptance criteria, and exclusions. Requirements are anchored to the [PRD](PRD.md) (Section 35 maps phases to requirement IDs); design detail to the [LLD](LLD.md) and its migration plan (Section 39). |
| Related documents | [PRD](PRD.md), [LLD](LLD.md), [ARCHITECTURE](ARCHITECTURE.md), [DATA_MODEL](DATA_MODEL.md), [DECISION_ENGINE](DECISION_ENGINE.md), [TOOL_CALLING](TOOL_CALLING.md), [API_SPEC](API_SPEC.md), [KNOWLEDGE_GRAPH](KNOWLEDGE_GRAPH.md), [SECURITY](SECURITY.md), [DEPLOYMENT](DEPLOYMENT.md), [AGENTS](AGENTS.md) |

Phases are sequential by dependency, not by calendar; no dates are promised. A phase exits only when its acceptance criteria pass, and later phases never begin on capabilities a prior phase left unaccepted. Status today: Phase 0 is in progress with most stabilisation items already landed (verified 2026-08-01); everything from Phase 1 onward is Proposed.

## 1. Document Control

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0 | 2026-08-01 | Jahnavi Ralhan | Initial roadmap against PRD v3.0; Phase 0 status grounded in verified code |

## 2. Phase 0: Current MVP Stabilisation

**Objective**: a truthful, reproducible, tested baseline in one repository.

**Capabilities**: full-corpus ingestion with live status reporting (FR-002, FR-003); grounded Q&A (FR-001); server-backed shipment analysis over the verified 8-product catalogue (FR-004 upgraded from client-side demo); labelled degradation (FR-007); calculator test suite.

**Status detail (verified)**: already landed via the 2026-08-01 fix set: 20-page ingestion cap removed, hardcoded status endpoints replaced with live counts, stable chunk IDs, pinned dependencies, lazy client construction, error-detail gating, gitignore and secrets hygiene with key rotation, catalogue extended to 8 products verified against the source PDF, pytest calculator suite (9 tests). Remaining: push code and docs to the repository as one tree; add CI running the test suite on every push; add first RAG-path tests (retrieval smoke against a fixture index); make model identifiers environment-configurable.

**Dependencies**: none.

**Risks**: R-6 (now closed by verification; residual risk is repo drift if code and docs are updated separately, mitigated by CI living in the same repo).

**Acceptance criteria**: repository contains code plus docs; CI green on push (pytest plus lint); `python -m rag.rag_pipeline && python app.py` works from a clean clone per [DEPLOYMENT.md](DEPLOYMENT.md) Section 3; `/api/status` reports live counts; all eight catalogue products round-trip through `/api/analyze-shipment`; README states the local-only security posture.

**Excluded**: any new capability; framework migration (OQ-02); auth.

## 3. Phase 1: Deterministic Shipment-Intelligence Foundation

**Objective**: authoritative calculation moves fully server-side with engineering-grade numerics.

**Capabilities**: Calculation Engine per LLD Section 12: Decimal arithmetic, closed unit enum, versioned pure formulas, golden fixtures, lineage on every run (FR-015 core); basic deterministic entity resolution, exact and alias paths (FR-011 basic); the interim investigation endpoint with the explicit `phase1_reduced` response marker (API_SPEC Section 5); frontend deletes its client-side engine and hardcoded confidence figure, rendering only engine outputs.

**Dependencies**: Phase 0 exit.

**Risks**: TR-4 (Decimal retrofit versus demo parity; mitigated by re-deriving current formulas as the first golden fixtures before the frontend switch).

**Acceptance criteria**: golden tests pass with byte-identical reproduction across runs; unit mismatches reject rather than coerce; the MVP reference case (500 t pig iron, 1.80 versus 2.07, EUR 80) reproduces 135 tCO2e and EUR 10,800 from the new engine; frontend contains no calculation logic and no confidence percentage; every response carries the reduced-lifecycle marker.

**Excluded**: policy evaluation, evidence validation, durable storage, the full decision object.

## 4. Phase 2: Typed Tool Registry and Structured Decision Object

**Objective**: the governed execution spine: every capability a contract, every invocation a trace, every decision a validated object.

**Capabilities**: tool registry and executor with the initial read-only, retrieval, and compute tool set (FR-021); structured decision object with schema validation and the five confidence dimensions (FR-018, AIR-005); durable decision traces and replay of calculation and policy stages (FR-020 trace level, NFR-001); structured-output enforcement, numeric guard, citation post-check (AIR-001, AIR-002, AIR-003, AIR-008); risk engine basic (FR-017 basic); evaluation harness skeleton (LLD Section 33); decisions and tools APIs (API_SPEC Sections 5, 6, 10).

**Dependencies**: Phase 1 engine (tools wrap it); OQ-02 decided at phase planning (Flask versus FastAPI); OQ-13 decided (decision schema versioning).

**Risks**: TR-5 (registry over-abstraction; mitigated by shipping only tools the Phase 2 flows invoke); TR-7 (framework churn).

**Acceptance criteria**: schema-invalid tool inputs and outputs are rejected and traced; write and high-impact tool classes are absent from every LLM-selectable subset (injection corpus test passes); a stored decision replays byte-identically; decision objects failing schema validation never render; the evaluation skeleton runs in CI.

**Excluded**: human-review workflow (routing rules compute but everything above auto-approve blocks conservatively until Phase 4); probabilistic tools (OQ-17 recommendation: deferred); bulk ingestion.

## 5. Phase 3: Persistent Enterprise Data and Bulk Ingestion

**Objective**: BorderIQ becomes a system of record: durable data, real deployments, identity.

**Capabilities**: PostgreSQL with the DATA_MODEL transactional entities (FR-027); object storage; CSV and Excel import with per-row outcomes plus batch API (FR-010); full entity resolution with review-queue persistence (FR-011 full); evidence upload with scanning (API_SPEC Section 7 upload level); outbox and queue with the first events and webhooks (LLD Section 23, API_SPEC Section 13); authentication and RBAC (SECURITY Section 6); observability, budgets, caches (NFR-008, NFR-010); dev, staging, prod environments with CI/CD (DEPLOYMENT Sections 4, 9); health probes; vendored frontend assets; OQ-03 executed (vector store decision).

**Dependencies**: Phase 2 (decisions must be durable objects before storing them is meaningful).

**Risks**: scope width (this is the largest phase; mitigated by the LLD Section 39 ordering inside it: storage, then auth, then imports, then events); NFR-002 and NFR-003 latency and throughput targets unvalidated until load tests here.

**Acceptance criteria**: a 1,000-row import completes with per-row outcomes and a completion event (FR-010 criteria); data survives restart and restore (backup drill per DEPLOYMENT Section 11); anonymous access to every mutating route is denied; the isolation suite exists (single-tenant but tests are written); NFR-002 and NFR-003 load reports produced; deployment is rolling through CI.

**Excluded**: policy engine, review UI, multi-tenancy beyond the single-tenant schema fields.

## 6. Phase 4: Policy-as-Code and Human Review

**Objective**: eligibility and approval become governed, versioned, and human-gated.

**Capabilities**: policy store and engine with effective dating (FR-012, FR-014); evidence sufficiency evaluation over data-quality states (FR-013); review service, queue, and workspace with the full decision-table routing (FR-019, DECISION_ENGINE Sections 7, 8); full audit packages and export (FR-020 full); tenant materiality settings (OQ-12 defaults decided); AIR-010 approval workflow including policy activation; retention enforcement begins (NFR-005); probabilistic tools land if OQ-17 confirms.

**Dependencies**: Phase 3 (policies, tasks, and packages need durable storage and identity); OQ-08 answered (who approves policy) and OQ-15 answered (reference-date rule).

**Risks**: R-1 (misinterpretation embedded in policies; mitigated by golden period tests and the approval gate); reviewer-capacity assumptions untested until a pilot.

**Acceptance criteria**: no policy activates without a recorded qualified approval; golden effective-date cases pass 100 percent; a decision matching any routing rule cannot finalise without reviewer action; an exported audit package passes the manifest completeness check and replays; expired verifier evidence demonstrably blocks actuals eligibility with the stated reason.

**Excluded**: regulatory change automation (diffs arrive Phase 5); scenario engine.

## 7. Phase 5: Hybrid Regulatory Intelligence and Change Detection

**Objective**: retrieval graduates from similarity search to effective-dated regulatory intelligence.

**Capabilities**: document registry with checksums, versions, supersession (FR-022); structural parsing with page fallback (LLD Section 19); hybrid scoped retrieval with reranking, citation validation, sufficiency, abstention (FR-023, AIR-003, AIR-004); scoped `/v1/ask` and retrieval APIs replacing the legacy `/api` routes; change intelligence with impact mapping and review-task creation (FR-024); full retrieval and generation evaluation suites as release gates (PRD Section 29 targets); OQ-14 decided (reranker).

**Dependencies**: Phase 4 (change intelligence needs the review workflow and policy store to have somewhere to land); Phase 3 storage.

**Risks**: TR-1 (parsing brittleness; page fallback is the floor), TR-3 (hybrid complexity; ship scoped dense first, add lexical and reranking behind flags with measured lift), R-2 (retrieval wrongness; the eval suites are the control).

**Acceptance criteria**: every indexed fragment traces to a checksummed source version; superseded versions never retrieved for current periods (golden tests); abstention triggers on the out-of-corpus set at target rate; retrieval eval meets the PRD Section 29 starred targets or the targets are re-baselined with documented rationale; a changed default-value table produces an impact report and a review task with no policy changing untouched by approval.

**Excluded**: graph-based impact traversal (relational joins suffice and remain the permanent floor); multi-jurisdiction corpora.

## 8. Phase 6: Knowledge Graph and Relationship Intelligence

**Objective**: relationship traversal where it earns rent: impact propagation, peer evidence, path explanation.

**Capabilities**: projection builder with rebuild and drift detection; the 13-node, 14-edge catalogue; the four query families; graph-informed change impact augmenting the Phase 5 relational floor; capability-flagged UI sections (FR-025, [KNOWLEDGE_GRAPH.md](KNOWLEDGE_GRAPH.md) throughout); OQ-04 and OQ-19 decided.

**Dependencies**: Phases 3 to 5 (there is nothing to project before durable entities, policies, and the document registry exist).

**Risks**: TR-8 (projection drift; nightly checks plus rebuild-as-routine).

**Acceptance criteria**: full investigation suite passes with the graph disabled (NFR-009 gate); projection counts reconcile with SQL after rebuild; the four golden traversal queries return fixture-correct results; the superseded-policy-exposure query feeds review-task creation end to end.

**Excluded**: graph as system of record for anything; graph-driven policy evaluation (deterministic engine remains sole authority).

## 9. Phase 7: Decision Twin and Scenario Engine

**Objective**: controlled what-if analysis on immutable snapshots of decided cases.

**Capabilities**: decision twin versioning (FR-026); scenario engine over assumption deltas with the standard comparison set: default versus actual, price, route, energy mix, evidence state, regulation version (FR-016); scenario APIs (API_SPEC Section 6); hypothetical labelling throughout.

**Dependencies**: Phase 4 (finalised decisions to snapshot); Phase 5 (regulation-version scenarios need the registry).

**Risks**: scenario results mistaken for decisions (mitigated by labelling invariants and the never-mutate rule, DECISION_ENGINE Section 13).

**Acceptance criteria**: scenarios never alter base decisions (property test); every scenario records its assumption set; twin versions are immutable; the default-versus-actual scenario reproduces the Phase 1 savings framing from stored twins.

**Excluded**: portfolio-level optimisation; prescriptive recommendations beyond the comparison.

## 10. Phase 8: Enterprise Integrations

**Objective**: meet enterprise data where it lives and hand results back.

**Capabilities**: adapter interface with field mappings and failure behaviour documented per adapter (FR-028); SQL and object-storage adapters first, then SAP, Oracle, Dynamics connectors; verifier-system and document-management integration; carbon-price adapter (OQ-07 decided); declaration draft export with the high-impact gate (FR-030); full event set live.

**Dependencies**: Phases 3 to 5; export additionally depends on Phase 4 approval workflow.

**Risks**: connector maintenance burden (mitigated by the stable adapter interface criterion: adding an adapter requires no core-schema change); partner-system access for testing.

**Acceptance criteria**: FR-028 criterion holds for each shipped adapter; export requires finalised decisions, a human trigger, and an approval reference, and is refused otherwise (TOOL_CALLING exemplar contract enforced); a full pilot cycle runs ERP-export to declaration-draft without manual spreadsheets.

**Excluded**: IoT ingestion as emissions source (permanent non-goal); autonomous filing (permanent non-goal).

## 11. Phase 9: Multi-Tenant Enterprise Platform

**Objective**: many customers, one platform, provable isolation.

**Capabilities**: tenancy enforced at every layer with the NFR-007 test suite as release gate (FR-029); tenant administration APIs; per-tenant quotas, policies, aliases, webhooks; ABAC extensions; availability engineering to NFR-004; customer-managed keys and residency options as contracts require (OQ-09, OQ-10); Kubernetes and Kafka thresholds re-evaluated against DEPLOYMENT Sections 6 and 7.

**Dependencies**: everything prior; a second real tenant to prove it.

**Risks**: isolation regressions (the canary and test-suite controls of SECURITY Sections 7 and 13); operational load of per-tenant configuration.

**Acceptance criteria**: zero findings from the cross-tenant suite across API, SQL, retrieval, storage, graph; tenant deletion completes with evidence within retention rules; uptime measured against NFR-004; onboarding a tenant requires configuration, not code.

**Excluded**: per-tenant code forks (never); jurisdiction expansion.

## 12. Phase 10: Multi-Jurisdiction Compliance Platform

**Objective**: the CBAM-shaped machinery generalised to further regimes, only where the model genuinely transfers.

**Capabilities**: jurisdiction as a first-class scope across registry, retrieval, policies, and calculations; candidate regimes evaluated per OQ-11 (CSRD, EUDR, SEC climate are candidates, not commitments); per-regime calculation and policy packs with their own golden suites.

**Dependencies**: Phase 9; OQ-11 answered with a market-driven selection; regime-specific legal review capacity (extends OQ-08).

**Risks**: R-7 (dilution before CBAM depth; the phase gate is the mitigation: Phase 10 opens only after pilot success metrics from PRD Section 30 are met on CBAM).

**Acceptance criteria**: a second regime runs end to end on shared machinery with a regime pack, no forked core; CBAM regression suites stay green throughout; scope decision documented with the same rigour as this suite.

**Excluded**: any regime lacking the deterministic-calculation plus versioned-policy shape; legal advice (permanent non-goal).

## 13. Cross-Phase Rules

Non-goals from PRD Section 15 bind every phase. The evaluation gates that exist at each phase run on every release from then on; suites are never retired by later phases. Open questions block only the phases that name them; each phase's planning session must close its named OQs or explicitly re-scope. Requirement traceability: PRD Section 35 is the authoritative phase-to-requirement map; where this document and that table disagree, the PRD wins and this document gets corrected.

## 14. Open Questions

All owned in the [PRD](PRD.md) Section 36 and referenced per phase above; this document introduces none.
