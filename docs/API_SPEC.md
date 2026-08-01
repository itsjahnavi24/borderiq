# BorderIQ API Specification

| Field | Value |
|---|---|
| Title | BorderIQ API Specification |
| Version | 1.0 |
| Status | Draft for review |
| Owner | Jahnavi Ralhan |
| Last updated | 2026-08-01 |
| Scope | HTTP contracts: the current MVP API exactly as the verified client uses it, and the target v1 API for the platform. Entity field semantics live in [DATA_MODEL](DATA_MODEL.md); tool contracts in [TOOL_CALLING](TOOL_CALLING.md); lifecycle semantics in [DECISION_ENGINE](DECISION_ENGINE.md). |
| Related documents | [PRD](PRD.md), [LLD](LLD.md), [ARCHITECTURE](ARCHITECTURE.md), [DATA_MODEL](DATA_MODEL.md), [DECISION_ENGINE](DECISION_ENGINE.md), [TOOL_CALLING](TOOL_CALLING.md), [SECURITY](SECURITY.md), [DEPLOYMENT](DEPLOYMENT.md), [ROADMAP](ROADMAP.md) |

Labelling per the [PRD](PRD.md) convention. Section 2 is Current MVP; Sections 4 onward are Target architecture (Proposed) with the introducing ROADMAP phase stated per group.

## 1. Document Control

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0 | 2026-08-01 | Jahnavi Ralhan | Initial API specification against PRD v3.0 and the verified MVP client |

## 2. Current MVP API

Three unauthenticated JSON routes under `/api`. Shapes below are exactly what the verified frontend sends and renders; the backend implementation is Current MVP (described, OQ-01). No other route exists; in particular, no shipment-analysis endpoint is called by the current client (the analyzer runs client-side, PRD Section 17.1).

### POST /api/ask

Purpose: citation-backed regulatory Q&A (FR-001). Status: Implemented (frontend verified; backend described).

Request:

```json
{ "question": "What are default values for Pig Iron?" }
```

Response 200:

```json
{
  "answer": "string",
  "sources": [
    { "source_file": "Default values transitional period.pdf",
      "page_number": 7,
      "document_type": "Default Values",
      "chunk_id": "chunk_17" }
  ]
}
```

Client behaviour on failure or timeout (20 s): labelled demo fallback (FR-007).

### GET /api/status

Purpose: knowledge-base and service health (FR-002). Status: Implemented (frontend verified; backend described).

Response 200:

```json
{ "documents_indexed": 3, "chunks_created": 159,
  "vector_db": "Connected", "llm": "Connected",
  "embedding_model": "text-embedding-3-small" }
```

### GET /api/knowledge-base

Purpose: list indexed documents (FR-003). Status: Implemented (frontend verified; backend described). The client accepts either a bare array or `{"documents": [...]}`.

Response 200:

```json
[ { "source_file": "implementation regulation.pdf",
    "document_type": "Implementing Regulation",
    "pages_processed": 68, "status": "Indexed", "last_indexed": "Demo run" } ]
```

Migration note: these three routes remain available during Phases 0 to 2 and are superseded by `/v1/ask`, `/v1/status`, and `/v1/regulatory/documents`; the legacy paths are retired one minor version after the frontend switches (NFR-012).

## 3. Target API Conventions

All target endpoints share these conventions; they are not repeated per endpoint.

- **Base path and versioning**: `/v1`. Breaking changes only in `/v2` with migration notes (NFR-012). Additive changes within v1.
- **Authentication and authorization**: bearer token from the identity provider (C-21); role checks per endpoint (roles: analyst, reviewer, auditor, admin; [SECURITY.md](SECURITY.md)). Tenant context derives from the token, never from request bodies.
- **Correlation**: every response carries `X-Request-Id`; clients may supply one and it propagates through traces (NFR-008).
- **Idempotency**: mutating endpoints marked Idempotency: key accept an `Idempotency-Key` header (max 128 chars); replays return the recorded response with `Idempotency-Replayed: true` (LLD Section 25).
- **Pagination**: list endpoints accept `limit` (default 50, max 200) and `cursor`; responses return `next_cursor` or null.
- **Timestamps and numerics**: UTC ISO 8601; all quantities and money as DATA_MODEL types (decimal strings, explicit unit or currency).
- **Content type**: `application/json`, except file upload and package download endpoints.

**Error envelope** (all non-2xx):

```json
{ "error": { "code": "validation_failed", "message": "human-readable summary",
             "request_id": "req_9f31",
             "fields": [ { "path": "lines[0].quantity.value", "issue": "must be a positive decimal string" } ] } }
```

Error codes: `validation_failed` 400, `unauthenticated` 401, `forbidden` 403, `not_found` 404, `conflict` 409 (optimistic lock or duplicate), `idempotency_conflict` 409 (same key, different payload), `rate_limited` 429, `dependency_unavailable` 503, `timeout` 504, `internal` 500. Domain outcomes (insufficient evidence, unsupported case, abstention) are 200-level responses with status fields, never errors (LLD Section 27).

## 4. Shipments and Imports

Introduced Phase 3 (FR-010, FR-027). Roles: analyst.

| Endpoint | Purpose | Idempotency |
|---|---|---|
| POST /v1/shipments | Create one shipment with lines | key |
| GET /v1/shipments/{id} | Fetch shipment with lines and resolution state | n/a |
| GET /v1/shipments | List; filters: reporting_period, status, external_ref | n/a |
| POST /v1/imports | Start a CSV or Excel import job (multipart or object-storage reference) | key |
| GET /v1/imports/{job_id} | Job status and per-row report | n/a |

POST /v1/shipments request (shape mirrors DATA_MODEL Shipment and ShipmentLine; raw identifiers permitted, resolution happens server-side):

```json
{ "external_ref": "ACME-2026-0417", "declarant_org_id": "uuid-o1",
  "reporting_period": "2026-Q2", "destination_country": "NL",
  "lines": [ { "line_no": 1, "raw_product_text": "PIG IRON GR-A",
               "quantity": { "value": "120.000", "unit": "t" },
               "installation_ref": "NO-INST-0042" } ] }
```

Response 201: the created Shipment with `status: "received"` and per-line resolution outcomes (resolved, ambiguous with review_task_id, unknown). Errors: `validation_failed` with per-field paths; `conflict` on duplicate external_ref.

GET /v1/imports/{job_id} response 200 (FR-010 acceptance shape):

```json
{ "job_id": "imp_91", "status": "completed",
  "rows_total": 1000, "rows_accepted": 987, "rows_rejected": 9, "rows_review": 4,
  "report": [ { "row": 412, "outcome": "rejected",
                "reasons": [ { "path": "quantity.unit", "issue": "unsupported unit 'lbs'" } ] },
              { "row": 417, "outcome": "review", "review_task_id": "uuid-rt9" } ] }
```

## 5. Investigations

Introduced Phase 2 (server-side analysis replacing the client demo engine; FR-015, FR-018). Roles: analyst.

| Endpoint | Purpose | Idempotency |
|---|---|---|
| POST /v1/investigations | Run one investigation for a shipment line (sync for standard cases, async above thresholds) | key |
| GET /v1/investigations/{id} | Status and, when complete, the decision reference | n/a |
| POST /v1/investigations/{id}/rerun | Re-run after evidence or regulation change; supersedes per DECISION_ENGINE Section 7 | key |

POST /v1/investigations request:

```json
{ "shipment_line_id": "uuid-sl1",
  "requested_methodology": "actuals",
  "assumptions": { "carbon_price": { "value": "80.00", "currency": "EUR", "source": "user_supplied" } } }
```

Response 202 (async) or 200 (sync):

```json
{ "investigation_id": "uuid-inv1", "status": "completed", "decision_id": "uuid-d1" }
```

Phase 1 interim: before the full lifecycle exists, this endpoint runs stages 8 and 10 only (calculation plus basic risk) and returns engine results without policy evaluation; the response carries `"lifecycle": "phase1_reduced"` so clients cannot mistake it for a full decision. This is the endpoint the frontend switches to when deleting its client-side engine (LLD Section 39 step 2).

## 6. Decisions, Traces, Scenarios, Audit Packages

Decisions and traces Phase 2; audit packages Phase 4; scenarios Phase 7. Roles: analyst (read), reviewer or auditor for packages.

| Endpoint | Purpose | Idempotency |
|---|---|---|
| GET /v1/decisions/{id} | The structured decision object (DATA_MODEL Section 4.8) | n/a |
| GET /v1/decisions | List; filters: status, shipment_id, reporting_period | n/a |
| GET /v1/decisions/{id}/trace | Full decision trace: tool executions, retrieval traces, calculation runs, versions | n/a |
| GET /v1/decisions/{id}/replay | Execute replay and return the diff report (ARCHITECTURE Section 11) | n/a |
| POST /v1/decisions/{id}/audit-package | Materialise and return the AuditPackage reference (FR-020) | key |
| GET /v1/audit-packages/{id}/download | Signed download of the package archive | n/a |
| POST /v1/decisions/{id}/scenarios | Run a scenario against the decision twin (FR-016) | key |
| GET /v1/decisions/{id}/scenarios | List scenarios with assumptions and comparisons | n/a |

GET /v1/decisions/{id} response 200: the ComplianceDecision JSON exactly as modelled in DATA_MODEL Section 4.8, with figures resolved for display alongside their CalculationRun references:

```json
{ "id": "uuid-d1", "status": "finalised", "methodology": "actuals",
  "figures": { "calculation_run": "uuid-cr1",
                "resolved": { "avoided": { "value": "78.000", "unit": "tCO2e" },
                               "difference": { "value": "6240.00", "currency": "EUR" } } },
  "confidence": { "mapping": { "level": "high", "basis": "alias, human-approved" } },
  "supporting_sources": [ { "fragment_id": "uuid-fr77", "version": "uuid-rv2", "pages": [7] } ],
  "narrative": "..." }
```

POST /v1/decisions/{id}/scenarios request:

```json
{ "label": "Carbon price 95",
  "assumptions": { "carbon_price": { "value": "95.00", "currency": "EUR" } } }
```

Response 200: Scenario object with base-versus-scenario comparison, labelled hypothetical. Error: `conflict` with code detail `base_not_finalised` when the base decision is not finalised.

## 7. Evidence

Introduced Phase 3 (upload and storage) and Phase 4 (validation semantics, FR-013). Roles: analyst.

| Endpoint | Purpose | Idempotency |
|---|---|---|
| POST /v1/evidence | Register evidence metadata and obtain a signed upload URL | key |
| GET /v1/evidence/{id} | Metadata, data-quality state, linkage | n/a |
| GET /v1/evidence/{id}/validation | Validation findings per FR-013 checks | n/a |

POST /v1/evidence request:

```json
{ "kind": "verification_report", "verifier_name": "Nordic Verify AB",
  "covers_installation_id": "uuid-i1",
  "period_from": "2026-01-01", "period_to": "2026-06-30",
  "file": { "name": "vr1.pdf", "content_type": "application/pdf", "bytes": 482113 } }
```

Response 201:

```json
{ "id": "uuid-vr1", "upload_url": "https://storage.example/signed/...",
  "data_quality_state": "unverified" }
```

Uploaded files pass scanning and type validation before becoming referenceable (SECURITY.md); until then `data_quality_state` stays `unverified` and FR-013 treats the evidence as absent. Kinds: `supplier_declaration`, `verification_report`, `emission_record_source`.

## 8. Regulatory Documents and Versions

Read endpoints Phase 5 with the document registry; ingest is admin-only. Roles: analyst (read), admin (ingest).

| Endpoint | Purpose | Idempotency |
|---|---|---|
| GET /v1/regulatory/documents | Registry listing with families and latest versions | n/a |
| GET /v1/regulatory/documents/{id}/versions | Version history with effective windows and supersession | n/a |
| GET /v1/regulatory/versions/{id} | One version: provenance, checksum, ingestion status, fragment counts | n/a |
| GET /v1/regulatory/versions/{id}/diff?against={version_id} | Structural diff report (FR-024) | n/a |
| POST /v1/regulatory/ingest | Ingest from an allowlisted source reference (admin; checksum-idempotent) | inherent |

Diff response 200 (summary shape):

```json
{ "base": "uuid-rv1", "against": "uuid-rv2",
  "changes": [ { "path": "annex-1/table-1", "kind": "table_values_changed",
                 "impact": { "categories": ["iron_steel"], "cn_codes": ["7201"],
                              "active_policies": ["uuid-pv3"], "open_decisions": 4 } } ],
  "review_task_id": "uuid-rt12" }
```

## 9. Retrieval and Q&A

Scoped retrieval and abstention Phase 5; `/v1/ask` supersedes `/api/ask`. Roles: analyst.

| Endpoint | Purpose | Idempotency |
|---|---|---|
| POST /v1/retrieval/query | Scoped fragment retrieval (mirror of the retrieve_regulatory_evidence tool contract) | n/a |
| POST /v1/ask | Grounded Q&A with mandatory scope, citations, and abstention | n/a |

POST /v1/ask request and the two response forms:

```json
{ "question": "What are default values for pig iron?",
  "scope": { "jurisdiction": "EU", "effective_on": "2026-06-30", "category": "iron_steel" } }
```

```json
{ "outcome": "answered", "answer": "...",
  "sources": [ { "fragment_id": "uuid-fr77", "version_id": "uuid-rv2",
                  "document": "Default values transitional period.pdf", "pages": [7] } ],
  "trace_id": "uuid-rt1" }
```

```json
{ "outcome": "abstained",
  "reason": "no sufficient passages for scope category=cement, effective_on=2026-06-30",
  "searched": { "corpus_version": "2026-07-01", "filters_applied": ["jurisdiction", "effective_on", "category"] },
  "trace_id": "uuid-rt2" }
```

Abstention is a 200 with `outcome: "abstained"` (AIR-004), never an error.

## 10. Tools

Phase 2, with strict permissioning (FR-021; [TOOL_CALLING.md](TOOL_CALLING.md)). Roles: analyst for the catalogue and permitted read-only execution; reviewer or admin for anything beyond.

| Endpoint | Purpose | Idempotency |
|---|---|---|
| GET /v1/tools | Registry catalogue: names, versions, classes, schemas | n/a |
| POST /v1/tools/{name}/execute | Execute one tool; permitted classes via API: read_only and retrieval only | per tool |

Execution rules: the endpoint refuses write and high_impact tools regardless of role (`forbidden` with detail `class_not_executable_via_api`); those execute only inside orchestrated flows or governed workflows. Request body must validate against the tool's input schema; failures return `validation_failed` with schema paths. Response wraps the tool output with the trace reference:

```json
{ "tool": "retrieve_default_emission_value", "version": "1.0.0",
  "outcome": "ok", "output": { "factor_id": "uuid-f1" }, "execution_id": "uuid-tx7" }
```

## 11. Review Queues and Actions

Phase 4 (FR-019). Roles: reviewer (actions), analyst (read own-context tasks).

| Endpoint | Purpose | Idempotency |
|---|---|---|
| GET /v1/reviews | Queue listing; filters: status, reason, assigned_role | n/a |
| GET /v1/reviews/{id} | Task with full decision context references | n/a |
| POST /v1/reviews/{id}/action | approve, reject, request_information | key |

POST /v1/reviews/{id}/action request:

```json
{ "action": "reject", "rationale": "verifier report window does not cover the batch period" }
```

Rules: `rationale` required for reject (`validation_failed` otherwise); actions on non-open tasks return `conflict`; every action writes the reviewer trail (DECISION_ENGINE stage 15) and transitions the decision per the state machine.

## 12. Tenant Administration and Status

Tenant settings Phase 4 minimal (materiality threshold), full administration Phase 9 (FR-029). Roles: admin.

| Endpoint | Purpose | Idempotency |
|---|---|---|
| GET /v1/tenant/settings | Current settings (materiality threshold, retention overrides, grounding mode per OQ-16) | n/a |
| PATCH /v1/tenant/settings | Update settings; every change audited | key |
| GET /v1/status | Service health: components, corpus version, queue depth (supersedes /api/status) | n/a |

GET /v1/status response 200:

```json
{ "components": { "database": "ok", "vector_index": "ok", "queue": "ok",
                   "llm_provider": "ok", "graph": "disabled" },
  "corpus_version": "2026-07-01", "documents_indexed": 3, "fragments_indexed": 1240 }
```

`graph: "disabled"` is a healthy state (NFR-009), not a failure.

## 13. Webhooks and Event Subscriptions

Phase 3 minimal (decision.completed, import job events), full event set Phase 5 (PRD Section 28 list). Roles: admin.

| Endpoint | Purpose | Idempotency |
|---|---|---|
| POST /v1/webhooks | Register an endpoint for selected event types | key |
| GET /v1/webhooks | List registrations with delivery stats | n/a |
| DELETE /v1/webhooks/{id} | Remove a registration | n/a |

Delivery contract: JSON envelope `{event_id, event_type, occurred_at, tenant_id, schema_version, data}`; HMAC-SHA256 signature header over the raw body with a per-registration secret; at-least-once delivery with exponential backoff and a bounded retry window; consumers deduplicate by `event_id`; failed registrations are disabled after sustained failure and surfaced in delivery stats. Event payloads carry references, not full objects, so webhook consumers fetch through authorised endpoints (no data bypasses authorization via webhooks).

Event types at introduction: `decision.completed`, `decision.review_required`, `import.completed`, then the remaining PRD Section 28 set as their producers land.

## 14. Endpoint Status Summary

| Group | Endpoints | Status |
|---|---|---|
| Current MVP `/api` | ask, status, knowledge-base | Implemented (frontend verified; backend described); retirement per Section 2 |
| Investigations | 3 | Proposed (Phase 2; Phase 1 reduced mode) |
| Decisions, traces | 4 | Proposed (Phase 2) |
| Tools | 2 | Proposed (Phase 2) |
| Shipments, imports | 5 | Proposed (Phase 3) |
| Evidence | 3 | Proposed (Phase 3 upload, Phase 4 validation) |
| Webhooks | 3 | Proposed (Phase 3 minimal) |
| Reviews | 3 | Proposed (Phase 4) |
| Audit packages | 2 | Proposed (Phase 4) |
| Tenant settings, status | 3 | Proposed (Phase 4 minimal, Phase 9 full) |
| Regulatory registry, diff, ingest | 5 | Proposed (Phase 5) |
| Retrieval, scoped ask | 2 | Proposed (Phase 5) |
| Scenarios | 2 | Proposed (Phase 7) |

## 15. Open Questions

Owned in the [PRD](PRD.md): OQ-02 (framework; contracts here are framework-neutral), OQ-13 (decision schema versioning relative to API versioning), OQ-16 (tenant grounding mode surfaced in tenant settings). New: OQ-18: whether POST /v1/investigations should support a batch body in addition to per-line calls, or leave batching entirely to /v1/imports. Recommendation: leave batching to imports; one investigation per call keeps idempotency and error semantics simple.
