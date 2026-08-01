# BorderIQ Knowledge Graph

| Field | Value |
|---|---|
| Title | BorderIQ Knowledge Graph |
| Version | 1.0 |
| Status | Draft for review |
| Owner | Jahnavi Ralhan |
| Last updated | 2026-08-01 |
| Scope | The optional graph projection: why and when graph traversal earns its place, the node and relationship catalogue, example queries, and the fail-closed degradation contract. Entity truth lives in [DATA_MODEL](DATA_MODEL.md); the graph never replaces it. |
| Related documents | [PRD](PRD.md), [LLD](LLD.md), [ARCHITECTURE](ARCHITECTURE.md), [DATA_MODEL](DATA_MODEL.md), [DECISION_ENGINE](DECISION_ENGINE.md), [API_SPEC](API_SPEC.md), [SECURITY](SECURITY.md), [DEPLOYMENT](DEPLOYMENT.md), [ROADMAP](ROADMAP.md) |

Status: Target architecture (Proposed, Phase 6; FR-025). Nothing graph-related exists in the current MVP, and the platform is required to be fully functional without it (NFR-009). Requirement anchors: FR-024, FR-025, NFR-009.

## 1. Document Control

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0 | 2026-08-01 | Jahnavi Ralhan | Initial knowledge-graph design against PRD v3.0, LLD v1.0, DATA_MODEL v1.0 |

## 2. Position: an Optional Projection, Never the Source of Truth

The graph is a rebuildable projection of relational truth (LLD Section 22, technical risk TR-8). PostgreSQL owns every entity; the projection builder derives nodes and edges from those rows. The graph is never written in a request path, never stores anything that exists nowhere else, and can be dropped and rebuilt at any time without data loss. If it is absent, misconfigured, or down, every feature that uses it degrades to a shape-preserving empty response and core investigation is untouched.

This posture is deliberate: a graph database is a second operational system with its own failure modes, backup story, and access controls. It must pay rent in query capability, not exist for architectural fashion (PP-9).

## 3. Why a Graph: the Query Shapes That Earn It

Three query families are genuinely awkward in SQL and natural in a graph:

1. **Multi-hop regulatory impact.** "This annex changed; which suppliers ship affected goods to us next quarter?" is a five-hop traversal (fragment to CN code to product to shipment line to shipment to supplier) whose hop count varies by question. In SQL this is a different hand-written join chain per question shape; in a graph it is one variable-length pattern. This family powers change intelligence (FR-024).
2. **Contextual similarity.** "Which installations similar to this one (same route, same category, overlapping suppliers) have validated actuals?" supports evidence strategy: if peers demonstrate actuals, the gap is process, not feasibility.
3. **Path explanations.** "Show the chain from this decision back to the regulation version and the verifier report" is a path query whose result is itself the explanation, aligned with the lineage model (DATA_MODEL Section 8).

**When the graph is the wrong tool, and what is used instead:** entity lookup by identifier (SQL primary keys and alias tables); one-hop joins such as shipment-to-lines (SQL); text similarity over regulation content (the vector index); aggregation and reporting (SQL); anything transactional (SQL, always). A traversal of depth one or two with a fixed shape is a join, and joins stay in PostgreSQL. The honest threshold: the graph earns its place when questions have variable-depth, heterogeneous-edge patterns that would otherwise multiply bespoke SQL.

## 4. Division of Storage

| Concern | Store | Not the graph because |
|---|---|---|
| Entity truth, transactions, tenancy | PostgreSQL | ACID, retention, audit immutability |
| Regulation text and retrieval | Vector plus lexical indexes | Similarity search is not traversal |
| Raw documents, evidence files, audit packages | Object storage | Blobs do not belong in any database |
| Relationship traversal, impact propagation, similarity, path explanation | Graph projection | The one job it is for |

The graph stores identifiers, traversal-relevant properties, and edges. It never stores document text, calculation payloads, evidence files, or anything requiring retention guarantees of its own.

## 5. Projection Architecture

The projection builder is a batch worker (rebuild) plus an incremental updater (event-driven, consuming the LLD Section 23 event stream: shipment.created, regulation.published, policy.approved, decision.completed and peers). Rules:

- Every node carries `source_id` (the PostgreSQL primary key) and `projected_at`.
- Rebuild is idempotent and complete: drop the tenant subgraph, stream from PostgreSQL, recreate. Target: rebuild is routine, not an incident procedure.
- Drift detection (TR-8): nightly counts and checksums per node and edge type compared against relational queries; divergence beyond tolerance triggers a rebuild and an alert.
- Writes happen only in the builder identity; application request paths hold read-only graph credentials ([SECURITY.md](SECURITY.md)).

## 6. Node Catalogue

Thirteen node types, each projected from its owning entity in [DATA_MODEL.md](DATA_MODEL.md). Properties listed are traversal-relevant only.

| Node | Projected from | Key properties |
|---|---|---|
| Supplier | Supplier | source_id, name, country, data_quality_state |
| Installation | Installation | source_id, country, installation_code |
| ProductionRoute | ProductionRoute | code, category |
| Product | Product | source_id, canonical_key, category |
| CNCode | CNCode | code, cbam_covered, effective_from, effective_to |
| Shipment | Shipment | source_id, reporting_period, status |
| EmissionRecord | EmissionRecord | source_id, period_from, period_to, data_quality_state |
| Verifier | VerificationReport.verifier_name (deduplicated) | name, accreditation |
| Regulation | RegulatoryDocument plus RegulationVersion | source_id, family, version_label, effective_from, effective_to |
| Article | RegulatoryFragment | source_id, fragment_type, path, page_from |
| PolicyRule | PolicyVersion | source_id, definition_key, version_no, status, effective_from |
| ComplianceDecision | ComplianceDecision | source_id, status, methodology, risk_class |
| ReviewTask | ReviewTask | source_id, reason, status |

Shipment lines are folded into Shipment edges (Shipment PRODUCES-edge context) rather than projected as nodes; line-level detail is a SQL concern.

## 7. Relationship Catalogue

Fourteen edge types. Direction reads left to right.

| Edge | From, To | Meaning | Projected from |
|---|---|---|---|
| OPERATES | Supplier, Installation | Supplier runs the site | Installation.supplier_id |
| USES | Installation, ProductionRoute | Route in use at the site | Installation route links |
| PRODUCES | Installation, Product | Site produces the good | ShipmentLine and EmissionRecord joins |
| MAPS_TO | Product, CNCode | Canonical classification | Product.cn_code_id |
| CONTAINS | Regulation, Article | Version contains fragment | RegulatoryFragment.version_id |
| SOURCED_FROM | Shipment, Installation | Consignment origin | ShipmentLine.installation_id |
| SUPERSEDES | Regulation, Regulation | Version replacement | RegulationVersion.supersedes_version_id |
| APPLIES_TO | Article, CNCode | Fragment governs the code | Fragment metadata cn_codes |
| DERIVED_FROM | PolicyRule, Article | Rule provenance | PolicyVersion.derived_from_fragment_ids |
| MEASURED_AT | EmissionRecord, Installation | Measurement site | EmissionRecord.installation_id |
| VERIFIED_BY | EmissionRecord, Verifier | Verification authorship | VerificationReport join |
| USED | ComplianceDecision, EmissionRecord | Evidence consumed | CalculationRun input lineage |
| SUPPORTED_BY | ComplianceDecision, Article | Cited grounding | decision supporting_sources |
| GENERATED | ComplianceDecision, ReviewTask | Decision raised review | ReviewTask.decision_id |

Additional derived edges (PRODUCES via decisions, Product-to-Product precursor links) are candidates for a later version once base traversals prove out; starting minimal keeps drift detection tractable.

## 8. Example Queries

Cypher, assuming the catalogue above. These are the four query shapes the graph exists for.

**Regulatory change impact (FR-024): who is affected by a changed annex?**

```cypher
MATCH (reg:Regulation {source_id: $new_version_id})-[:CONTAINS]->(a:Article)
WHERE a.path STARTS WITH 'annex-1'
MATCH (a)-[:APPLIES_TO]->(cn:CNCode)<-[:MAPS_TO]-(p:Product)
      <-[:PRODUCES]-(i:Installation)<-[:OPERATES]-(s:Supplier)
OPTIONAL MATCH (sh:Shipment {reporting_period: $period})-[:SOURCED_FROM]->(i)
RETURN DISTINCT s.name AS supplier, i.installation_code AS installation,
       p.canonical_key AS product, collect(DISTINCT sh.source_id) AS open_shipments
```

**Audit path: explain a decision back to its sources.**

```cypher
MATCH path = (d:ComplianceDecision {source_id: $decision_id})
      -[:USED|SUPPORTED_BY|GENERATED]->(x)
OPTIONAL MATCH grounding = (d)-[:SUPPORTED_BY]->(:Article)<-[:CONTAINS]-(r:Regulation)
RETURN path, grounding, r.version_label AS regulation_version
```

**Peer evidence strategy: similar installations with validated actuals.**

```cypher
MATCH (me:Installation {source_id: $installation_id})-[:USES]->(route:ProductionRoute)
MATCH (peer:Installation)-[:USES]->(route)
WHERE peer <> me
MATCH (peer)<-[:MEASURED_AT]-(er:EmissionRecord {data_quality_state: 'validated'})
      -[:VERIFIED_BY]->(v:Verifier)
RETURN peer.installation_code AS peer_site, route.code AS route,
       count(er) AS validated_records, collect(DISTINCT v.name) AS verifiers
```

**Policy exposure to supersession: active rules derived from superseded text.**

```cypher
MATCH (pr:PolicyRule {status: 'active'})-[:DERIVED_FROM]->(a:Article)
      <-[:CONTAINS]-(old:Regulation)<-[:SUPERSEDES]-(new:Regulation)
RETURN pr.definition_key AS policy, pr.version_no AS version,
       old.version_label AS derived_from, new.version_label AS superseded_by
```

That last query is the graph earning its rent: it finds governance debt (policies citing superseded text) in four lines, and feeds the FR-024 review-task creation.

## 9. Graceful Degradation

The contract, mirroring the fail-closed pattern the MVP already proves at the frontend layer:

- A single capability check, `graph_enabled()`, gates every graph feature: true only when configuration is present and a liveness ping succeeds.
- When false, every graph-informed endpoint returns its normal response shape with empty graph sections and a capability flag; nothing errors, blocks, or retries in the request path. `GET /v1/status` reports `"graph": "disabled"` as a healthy state ([API_SPEC.md](API_SPEC.md) Section 12).
- Impact analysis (FR-024) falls back to relational fragment-metadata joins: less expressive (fixed-depth), still correct. Peer-similarity features are simply hidden by the capability flag.
- The frontend renders graph sections only when the capability flag is true; there is no loading state that waits on a disabled graph.
- Degradation tests (LLD Section 34) run the full investigation suite with the graph disabled as a release gate.

## 10. Tenancy

Tenant isolation follows [SECURITY.md](SECURITY.md) and NFR-007: every tenant-scoped node carries `tenant_id`, every query template includes the tenant predicate by construction (queries are parameterised templates owned by the service layer, never ad hoc strings from callers), and reference nodes (Regulation, Article, CNCode, ProductionRoute) are shared read-only. If the chosen store supports physical separation (per-tenant databases), Phase 9 evaluates it; until then, predicate isolation plus the NFR-007 cross-tenant test suite applies to graph queries exactly as to SQL.

## 11. Store Selection (OQ-04)

Candidates, honestly compared:

| Option | For | Against |
|---|---|---|
| Neo4j (or compatible: Memgraph) | Mature Cypher, best traversal ergonomics, the queries above run as written | Second database to operate; licensing review needed for managed versus self-hosted |
| PostgreSQL recursive CTEs (no new store) | Zero new infrastructure; fine for fixed-depth traversals | Variable-depth heterogeneous patterns become unreadable; no path ergonomics |
| Apache AGE (graph extension in PostgreSQL) | Cypher without a second database | Younger ecosystem; operational maturity to validate |

Recommendation: prototype Phase 6 against Neo4j Community or AGE behind the single `GraphPort` interface (LLD Section 22 isolates the choice), and let the FR-024 fallback (relational joins) remain the permanent floor so the graph never becomes load-bearing. Decision stays open as OQ-04.

## 12. Evaluation and Testing

- Projection correctness: node and edge counts per type reconciled against SQL ground truth after every rebuild (the drift checks of Section 5, run in CI against fixtures).
- Query correctness: golden results for the Section 8 queries on a fixture graph.
- Degradation: full-suite runs with the graph disabled (Section 9).
- Performance guardrail: traversal queries carry depth limits and timeouts registered like tool timeouts ([TOOL_CALLING.md](TOOL_CALLING.md) discipline applied to graph reads); an unbounded traversal is a defect.
- Tenancy: graph queries included in the NFR-007 isolation suite.

## 13. Open Questions

Owned in the [PRD](PRD.md): OQ-04 (store selection, Section 11). New: OQ-19: whether reference nodes (Regulation, Article, CNCode) live in one shared graph with tenant subgraphs attached, or are duplicated per tenant for physical isolation. Recommendation: shared reference plus tenant predicates until a customer's residency requirements (OQ-09) force duplication.
