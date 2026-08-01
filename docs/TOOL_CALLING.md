# BorderIQ Tool Calling

| Field | Value |
|---|---|
| Title | BorderIQ Tool Calling |
| Version | 1.0 |
| Status | Draft for review |
| Owner | Jahnavi Ralhan |
| Last updated | 2026-08-01 |
| Scope | The governed tool layer: registry architecture, per-tool contracts for all proposed tools, selection policy, validation, tracing, security, and testing. Orchestration mechanics live in [LLD](LLD.md) Sections 10 to 11; the lifecycle that consumes these tools in [DECISION_ENGINE](DECISION_ENGINE.md); execution persistence in [DATA_MODEL](DATA_MODEL.md) (ToolExecution). |
| Related documents | [PRD](PRD.md), [LLD](LLD.md), [ARCHITECTURE](ARCHITECTURE.md), [DATA_MODEL](DATA_MODEL.md), [DECISION_ENGINE](DECISION_ENGINE.md), [API_SPEC](API_SPEC.md), [SECURITY](SECURITY.md), [AGENTS](AGENTS.md), [ROADMAP](ROADMAP.md) |

Status: Target architecture (Proposed, Phase 2 onward). The current MVP has no tool layer; the LLM is used only for retrieval-grounded narration (Current MVP, described). Requirement anchors: FR-021, AIR-001, AIR-002, AIR-006, AIR-010.

## 1. Document Control

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0 | 2026-08-01 | Jahnavi Ralhan | Initial tool-calling specification against PRD v3.0, LLD v1.0, DECISION_ENGINE v1.0 |

## 2. Why Tool Calling

The investigation lifecycle needs capabilities the model does not have and must not simulate: enterprise data access, deterministic calculation, policy evaluation, scoped retrieval, and controlled side effects. Tool calling gives the model a governed vocabulary of actions: it may select which registered capability to invoke and with what arguments; the platform executes, validates, and records. This yields three properties a bare model cannot provide: every capability has a contract, every invocation has a trace, and every side effect has a permission and, where needed, an approval.

## 3. Why the LLM Is Not Allowed to Calculate

Language models produce plausible tokens, not verified arithmetic; in a compliance filing, a plausible-but-wrong figure is a failure with financial and regulatory consequences (PRD Sections 5 and 9). Deterministic engines give the properties audits require and models cannot: exact reproducibility (same inputs, same outputs, byte-identical on replay, NFR-001), unit safety (rejection instead of coercion), versioned formulas with golden tests, and complete input lineage. Therefore AIR-001: the model never performs arithmetic that reaches a decision; calculation tools accept typed inputs and return engine results, and the numeric guard (LLD Section 29) rejects any narrative that introduces numerals not present in the decision object. The model's legitimate numeric role is limited to extracting candidate values from documents, which then enter the system as unverified data subject to validation, never as results.

## 4. Tool-Registry Architecture

The registry is data, not code discovery: a versioned catalogue where each entry defines the complete contract of one tool. Registry entry fields (FR-021):

| Field | Meaning |
|---|---|
| name | Stable snake_case identifier, never reused |
| purpose | One-sentence responsibility |
| owner | Owning module (LLD component ID) |
| version | Semver; breaking schema changes bump major |
| input_schema / output_schema | JSON Schema (draft 2020-12), closed objects (`additionalProperties: false`) |
| classification | deterministic or probabilistic |
| class | read_only, deterministic_compute, retrieval, write, high_impact |
| timeout_ms | Hard execution ceiling |
| retry_class | none, retry_dependency (LLD Section 28) |
| idempotency | idempotent, at_most_once, requires_key |
| permissions | Required role or actor capability |
| approval | none, or the approval gate that must be satisfied |
| audit_fields | Extra fields captured into ToolExecution beyond the standard set |

The executor (LLD Section 11) is the only path to execution: it validates input against the schema, checks permission and approval, executes with the timeout, validates output against the schema, writes the ToolExecution trace, and returns only validated output. There is no bypass path; internal orchestrator calls go through the same executor so the trace is complete regardless of actor.

## 5. Discovery and Versioning

The model is offered, per interaction, only the subset of tools permitted for the current actor, stage, and tenant (the allowed tool list in ARCHITECTURE Section 7); it never sees the full registry. Tool definitions are versioned; an orchestrator run pins the registry snapshot version in provenance so replay resolves the same contracts. Deprecation: a tool version enters deprecated with a sunset date; both versions run in parallel during the window; names are never reused for different semantics. `GET /v1/tools` ([API_SPEC](API_SPEC.md)) exposes the catalogue for operators and integration tests.

## 6. Typed Schemas

All schemas are closed JSON Schema objects with explicit types, required lists, enums for closed sets, and the shared type definitions from [DATA_MODEL](DATA_MODEL.md) Section 3 (`Quantity`, `Money`, decimal strings, unit enum). Pydantic models generate and enforce the same schemas in code, so the registry document and the runtime validator cannot drift. Shared definitions (referenced as `$defs` in the exemplars below):

```json
{
  "$defs": {
    "Quantity": {
      "type": "object", "additionalProperties": false,
      "required": ["value", "unit"],
      "properties": {
        "value": { "type": "string", "pattern": "^-?\\d+(\\.\\d+)?$" },
        "unit": { "enum": ["t", "kg", "tCO2e", "tCO2e_per_t", "MWh", "GJ"] }
      }
    },
    "Money": {
      "type": "object", "additionalProperties": false,
      "required": ["value", "currency"],
      "properties": {
        "value": { "type": "string", "pattern": "^-?\\d+(\\.\\d+)?$" },
        "currency": { "enum": ["EUR"] }
      }
    },
    "Ref": { "type": "string", "format": "uuid" }
  }
}
```

## 7. Tool-Selection Policy

Who may select tools: the orchestrator (deterministic default plan, LLD Section 10.1), the optional Investigation Planner agent within its allowlist ([AGENTS.md](AGENTS.md)), and users via the administrative execute endpoint for permitted read-only tools. Selection rules: the model receives tool descriptions and the current context, and may propose a call; the executor decides. A proposal outside the offered subset is rejected and traced (never silently dropped, so injection attempts are visible). The model cannot chain side effects: write and high_impact tools are never in an LLM-selectable subset; they are invoked only by the orchestrator on rule triggers or by humans.

## 8. Tool-Output Validation

Outputs are schema-validated before anything consumes them; an invalid output is a tool failure (`output_invalid`), never partially accepted and never shown to the model as data (LLD Section 11). Semantic validation follows structural: referenced IDs must exist in tenant scope, quantities must carry permitted units, factor references must be effective for the investigation period. Probabilistic tools (the two classification tools below) additionally return a declared basis field, and their outputs are marked probabilistic in the trace so downstream confidence dimensions account for them (DECISION_ENGINE Section 6).

## 9. Tool Execution Trace

Every execution writes a ToolExecution row (DATA_MODEL Section 4.7): tool name and version, actor, validated input, validated output or failure, outcome, duration, idempotency key, approval reference where applicable, correlation and investigation IDs. Traces are immutable, retained under AUDIT class, and are the raw material of decision replay (FR-020) and of the tool failure-rate metric (NFR-008).

## 10. Retry and Timeout Policy

Timeouts are per tool (catalogue below); no unbounded call exists. Retry follows LLD Section 28: only `retry_dependency` failures retry, with backoff and bounded attempts; validation failures and domain outcomes never retry. Class defaults: read_only and retrieval 10 s timeout with dependency retry; deterministic_compute 5 s, no retry (pure functions either run or reject); write 15 s, retry only with an idempotency key; high_impact 30 s, no automatic retry (a failed high-impact call is surfaced, not repeated silently).

## 11. Idempotency

Deterministic compute tools are naturally idempotent. Read and retrieval tools are idempotent by definition. Write tools require a caller idempotency key (`requires_key`); replays return the recorded result (LLD Section 25). High-impact tools are at_most_once: the executor records intent before execution and refuses a second attempt with the same key, surfacing the first outcome.

## 12. Human Approval

Tools whose approval field is non-none cannot execute without a satisfied gate: a ReviewTask in approved state referencing the intended action (approval_ref on ToolExecution). This applies to export_declaration_draft (always) and to any write tool invoked outside its rule-triggered context. Policy activation is not a tool at all; it exists only inside the Phase 4 approval workflow (AIR-010), which prevents the failure mode of an over-permissioned planner activating policy.

## 13. Security

Per [SECURITY.md](SECURITY.md), applied here: least privilege (each tool's permission maps to a role; the executor enforces it regardless of actor); tenant scoping injected by the executor, never accepted from model-proposed arguments (a tool input may name entity IDs, but tenant context comes from the authenticated request); no tool performs raw network fetches to arbitrary destinations (ingestion uses the source allowlist; there is no generic fetch_url tool, closing the SSRF class); no tool executes code or shell; secrets never appear in tool inputs, outputs, or traces (executor-level redaction as defence in depth).

## 14. Prompt-Injection Containment

Retrieved fragments, uploaded documents, and any document-derived text are untrusted data (AIR-006). Containment layers: (1) evidence is delivered to the model in delimited data blocks with instructions that it is quotable content, never directives; (2) the model's tool proposals are constrained to the offered subset, so an injected "call export tool" instruction hits a rejection that is traced and alertable; (3) write and high_impact tools are outside every LLM-selectable subset (Section 7), bounding the blast radius of a successful manipulation to wasted read calls; (4) the injection test corpus (Section 16) includes document-embedded instruction attacks and is a release gate.

## 15. Fallback Behaviour

Per-tool failure feeds the degradation matrix (LLD Section 38): retrieval abstains; reranker-dependent quality degrades with a trace flag; calculation failure is terminal for its methodology path (DECISION_ENGINE decision table rows 3 and 5); write failures surface as required actions rather than silent gaps. The orchestrator never fabricates a tool result and never lets the model do so: a missing result is a missing result, expressed in decision status or required actions.

## 16. Testing

Contract tests: every registry entry's schemas validate their own examples; executor round-trip tests per tool (valid, structurally invalid, semantically invalid, timeout, output-invalid paths). Golden tests for deterministic_compute tools mirror the calculation engine fixtures. Injection corpus tests per Section 14. Permission matrix tests: every (role, tool) pair asserted allowed or denied. Idempotency tests: duplicate keys return recorded results; at_most_once refuses. These suites are CI release gates (LLD Sections 33 to 34).

## 17. Tool Catalogue

Thirty tools in six families. Family defaults apply to every tool in the family unless a row overrides them; combined with the standard registry fields of Section 4, each row below is a complete contract summary. All tools are version 1.0.0 at introduction. Classification is deterministic unless marked probabilistic.

### 17.1 Family defaults

| Family (class) | Owner | Timeout | Retry | Idempotency | Permission | Approval |
|---|---|---|---|---|---|---|
| A. Entity and data reads (read_only) | C-12 / Import services | 10 s | dependency | idempotent | role analyst | none |
| B. Reference retrieval (read_only) | C-05 reference / C-06 | 10 s | dependency | idempotent | role analyst | none |
| C. Evidence validation (read_only) | C-06 evidence rules | 10 s | dependency | idempotent | role analyst | none |
| D. Deterministic calculation (deterministic_compute) | C-05 | 5 s | none | idempotent | role analyst | none |
| E. Regulatory retrieval and policy (retrieval / read_only) | C-10 / C-06 | 10 s | dependency | idempotent | role analyst | none |
| F. Workflow and output (write / high_impact) | C-19 / C-20 | 15 s (30 s high_impact) | key-gated | requires_key (at_most_once high_impact) | per row | per row |

### 17.2 Catalogue

| Tool | Family | Purpose | Input (summary) | Output (summary) | Error semantics |
|---|---|---|---|---|---|
| fetch_shipment | A | Load a shipment with lines by ID or external ref | shipment_ref | Shipment plus ShipmentLines | not_found |
| fetch_supplier | A | Load supplier profile and quality state | supplier_id | Supplier | not_found |
| fetch_installation | A | Load installation with routes | installation_id | Installation | not_found |
| resolve_product | A | Canonical product for a raw name (LLD Section 20 order) | raw_text, category_hint | product_id, method, confidence | ambiguous (creates review item), unknown |
| resolve_cn_code | A | CN code for a product and period | product_id, period | cn_code_id, effective window | no_effective_code |
| classify_production_route | A, probabilistic | Suggest route from process description; suggestion only, human-confirmable | description, installation_id | route_code, basis | low_confidence (routes to review) |
| retrieve_default_emission_value | B | Effective default value for product and period | product_id, period | factor_id, Quantity, source fragment | no_effective_factor |
| retrieve_emission_factor | B | Effective factor by kind, region, period | kind, region, period, product_id? | factor_id, Quantity, validity | no_effective_factor |
| retrieve_carbon_price | B | Price reference for period (source per OQ-07) | period, source_policy | Money, source, as_of | no_price_source |
| validate_supplier_declaration | C | Validity and coverage of a declaration | declaration_id, needed_coverage | state, findings | not_found |
| validate_verifier_report | C | Window and validity of a verification report | report_id, window | state, findings | not_found |
| validate_evidence_completeness | C | Evidence verdict per candidate methodology (FR-013) | shipment_line_id, methodologies | per-methodology verdicts, missing_evidence | none (verdicts are the output) |
| calculate_direct_emissions | D | Direct emissions for a line or batch | quantities, intensities, factor refs | CalculationRun ref, results | rejected_unit, rejected_input, rejected_validity |
| calculate_indirect_emissions | D | Indirect (energy) emissions | energy inputs, factor refs | CalculationRun ref | as above |
| calculate_precursor_emissions | D | Precursor embedded emissions | precursor inputs, intensities | CalculationRun ref | as above |
| calculate_embedded_emissions | D | Total embedded emissions composition | component run refs | CalculationRun ref | as above |
| calculate_default_scenario | D | Default-values scenario for the line | line ref, default factor refs | CalculationRun ref | as above |
| calculate_actual_scenario | D | Actuals scenario for the line | line ref, evidence refs | CalculationRun ref | as above |
| estimate_cbam_exposure | D | Financial exposure from emissions and price | emissions run ref, Money price | CalculationRun ref | as above |
| compare_reporting_scenarios | D | Default versus actual comparison (savings basis) | two scenario run refs | CalculationRun ref (difference, avoided) | as above |
| retrieve_regulatory_evidence | E (retrieval) | Scoped fragment retrieval with sufficiency (FR-023) | query, scope filters | fragments with scores, sufficiency, trace ref | abstained (sufficiency insufficient) |
| resolve_policy_version | E | Active policy versions for context and period (FR-012) | context, period | PolicyVersion refs | no_effective_policy |
| evaluate_reporting_eligibility | E | Policy evaluation over context (FR-014) | context, policy refs | eligibility per methodology with citations | policy_conflict (routes per DECISION_ENGINE Section 9) |
| calculate_risk_score | D | Risk class from declared factors | exposure, evidence states, confidences | class, contributing factors | none (conservative default on missing inputs) |
| detect_data_anomaly | A, probabilistic | Flag implausible inputs (out-of-range intensity, quantity outliers); flags only, never blocks alone | line ref, history window | anomaly flags, basis | none |
| run_scenario_simulation | F (write, OPS-scoped) | Execute a scenario against a decision twin (FR-016) | base_decision_id, assumptions | Scenario ref | base_not_finalised |
| create_review_task | F (write) | Open a review task for a decision or mapping | decision_id or mapping ref, reason | ReviewTask ref | duplicate_open_task |
| generate_decision_package | F (write) | Materialise the decision view with resolved figures and citations | decision_id | package ref | decision_not_assembled |
| generate_audit_package | F (write) | Materialise the audit manifest closure (FR-020) | decision_id | AuditPackage ref | trace_incomplete (integrity alert) |
| export_declaration_draft | F (high_impact) | Assemble a draft declaration for human filing (FR-030) | decision_ids, format | export ref | requires: decisions finalised, human trigger, approval gate; unmet gate returns approval_required |

Permission overrides in family F: run_scenario_simulation role analyst; create_review_task role analyst; generate_decision_package role analyst; generate_audit_package role reviewer or auditor; export_declaration_draft role reviewer, approval always required, at_most_once.

### 17.3 Exemplar contracts (full schemas)

One exemplar per class; all remaining tools follow the identical pattern with their row's fields.

**resolve_product (read_only)**

```json
{
  "name": "resolve_product",
  "version": "1.0.0",
  "owner": "C-12",
  "classification": "deterministic",
  "class": "read_only",
  "timeout_ms": 10000,
  "retry_class": "retry_dependency",
  "idempotency": "idempotent",
  "permissions": ["analyst"],
  "approval": "none",
  "input_schema": {
    "type": "object", "additionalProperties": false,
    "required": ["raw_text"],
    "properties": {
      "raw_text": { "type": "string", "minLength": 1, "maxLength": 500 },
      "category_hint": { "enum": ["iron_steel", "cement", "aluminium", "fertilisers", "hydrogen", "electricity"] },
      "source_system": { "type": "string", "maxLength": 50 }
    }
  },
  "output_schema": {
    "type": "object", "additionalProperties": false,
    "required": ["outcome"],
    "properties": {
      "outcome": { "enum": ["resolved", "ambiguous", "unknown"] },
      "product_id": { "$ref": "#/$defs/Ref" },
      "method": { "enum": ["exact", "alias", "fuzzy"] },
      "confidence": { "type": "number", "minimum": 0, "maximum": 1 },
      "review_task_id": { "$ref": "#/$defs/Ref" }
    }
  }
}
```

**calculate_embedded_emissions (deterministic_compute)**

```json
{
  "name": "calculate_embedded_emissions",
  "version": "1.0.0",
  "owner": "C-05",
  "classification": "deterministic",
  "class": "deterministic_compute",
  "timeout_ms": 5000,
  "retry_class": "none",
  "idempotency": "idempotent",
  "permissions": ["analyst"],
  "approval": "none",
  "input_schema": {
    "type": "object", "additionalProperties": false,
    "required": ["investigation_id", "component_runs", "formula_version"],
    "properties": {
      "investigation_id": { "$ref": "#/$defs/Ref" },
      "component_runs": {
        "type": "array", "minItems": 1,
        "items": { "$ref": "#/$defs/Ref" },
        "description": "CalculationRun IDs for direct, indirect, precursor components"
      },
      "formula_version": { "type": "string", "pattern": "^\\d+\\.\\d+\\.\\d+$" }
    }
  },
  "output_schema": {
    "type": "object", "additionalProperties": false,
    "required": ["status", "calculation_run_id"],
    "properties": {
      "status": { "enum": ["ok", "rejected_unit", "rejected_input", "rejected_validity"] },
      "calculation_run_id": { "$ref": "#/$defs/Ref" },
      "results": {
        "type": "object", "additionalProperties": false,
        "properties": { "embedded_total": { "$ref": "#/$defs/Quantity" } }
      },
      "rejection_reasons": { "type": "array", "items": { "type": "string" } }
    }
  }
}
```

**retrieve_regulatory_evidence (retrieval)**

```json
{
  "name": "retrieve_regulatory_evidence",
  "version": "1.0.0",
  "owner": "C-10",
  "classification": "deterministic",
  "class": "retrieval",
  "timeout_ms": 10000,
  "retry_class": "retry_dependency",
  "idempotency": "idempotent",
  "permissions": ["analyst"],
  "approval": "none",
  "input_schema": {
    "type": "object", "additionalProperties": false,
    "required": ["query", "scope"],
    "properties": {
      "query": { "type": "string", "minLength": 3, "maxLength": 1000 },
      "scope": {
        "type": "object", "additionalProperties": false,
        "required": ["jurisdiction", "effective_on"],
        "properties": {
          "jurisdiction": { "enum": ["EU"] },
          "effective_on": { "type": "string", "format": "date" },
          "category": { "type": "string" },
          "cn_codes": { "type": "array", "items": { "type": "string" } },
          "methodology": { "enum": ["default", "actual", "any"] },
          "emissions_type": { "enum": ["direct", "indirect", "precursor", "total", "any"] }
        }
      },
      "top_k": { "type": "integer", "minimum": 1, "maximum": 20, "default": 8 }
    }
  },
  "output_schema": {
    "type": "object", "additionalProperties": false,
    "required": ["sufficiency", "abstained", "retrieval_trace_id"],
    "properties": {
      "sufficiency": { "enum": ["sufficient", "insufficient"] },
      "abstained": { "type": "boolean" },
      "retrieval_trace_id": { "$ref": "#/$defs/Ref" },
      "fragments": {
        "type": "array",
        "items": {
          "type": "object", "additionalProperties": false,
          "required": ["fragment_id", "version_id", "pages", "score"],
          "properties": {
            "fragment_id": { "$ref": "#/$defs/Ref" },
            "version_id": { "$ref": "#/$defs/Ref" },
            "pages": { "type": "array", "items": { "type": "integer" } },
            "score": { "type": "number" }
          }
        }
      }
    }
  }
}
```

**create_review_task (write)**

```json
{
  "name": "create_review_task",
  "version": "1.0.0",
  "owner": "C-19",
  "classification": "deterministic",
  "class": "write",
  "timeout_ms": 15000,
  "retry_class": "retry_dependency",
  "idempotency": "requires_key",
  "permissions": ["analyst"],
  "approval": "none",
  "input_schema": {
    "type": "object", "additionalProperties": false,
    "required": ["subject_type", "subject_id", "reason", "idempotency_key"],
    "properties": {
      "subject_type": { "enum": ["decision", "mapping", "policy_proposal"] },
      "subject_id": { "$ref": "#/$defs/Ref" },
      "reason": { "enum": ["ambiguous_mapping", "evidence_gap", "conflicting_sources", "expired_evidence", "materiality", "novel_route", "low_confidence_extraction", "policy_change", "policy_conflict", "declaration_export"] },
      "context": { "type": "object" },
      "idempotency_key": { "type": "string", "maxLength": 128 }
    }
  },
  "output_schema": {
    "type": "object", "additionalProperties": false,
    "required": ["review_task_id", "status"],
    "properties": {
      "review_task_id": { "$ref": "#/$defs/Ref" },
      "status": { "enum": ["open", "duplicate_open_task"] }
    }
  }
}
```

**export_declaration_draft (high_impact)**

```json
{
  "name": "export_declaration_draft",
  "version": "1.0.0",
  "owner": "C-20",
  "classification": "deterministic",
  "class": "high_impact",
  "timeout_ms": 30000,
  "retry_class": "none",
  "idempotency": "at_most_once",
  "permissions": ["reviewer"],
  "approval": "review_task_approved",
  "audit_fields": ["approval_ref", "decision_ids", "format"],
  "input_schema": {
    "type": "object", "additionalProperties": false,
    "required": ["decision_ids", "format", "approval_ref", "idempotency_key"],
    "properties": {
      "decision_ids": { "type": "array", "minItems": 1, "items": { "$ref": "#/$defs/Ref" } },
      "format": { "enum": ["draft_package_v1"] },
      "approval_ref": { "$ref": "#/$defs/Ref" },
      "idempotency_key": { "type": "string", "maxLength": 128 }
    }
  },
  "output_schema": {
    "type": "object", "additionalProperties": false,
    "required": ["outcome"],
    "properties": {
      "outcome": { "enum": ["exported", "approval_required", "decisions_not_finalised"] },
      "export_ref": { "type": "string" }
    }
  }
}
```

## 18. Open Questions

Owned in the [PRD](PRD.md): OQ-07 (retrieve_carbon_price source), OQ-12 (materiality inputs to calculate_risk_score). New: OQ-17: whether classify_production_route and detect_data_anomaly (the two probabilistic tools) ship in Phase 2 or defer to Phase 4 when review workflow exists to absorb their suggestions. Recommendation: defer both to Phase 4; before a review queue exists, a suggestion with no absorber is noise.
