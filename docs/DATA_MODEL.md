# BorderIQ Data Model

| Field | Value |
|---|---|
| Title | BorderIQ Data Model |
| Version | 1.0 |
| Status | Draft for review |
| Owner | Jahnavi Ralhan |
| Last updated | 2026-08-01 |
| Scope | Current MVP data shapes and the target entity catalogue: fields, validation, relationships, tenancy, retention, and the cross-cutting rules for units, decimals, effective dating, versioning, and schema evolution. Lifecycle semantics live in [DECISION_ENGINE](DECISION_ENGINE.md); storage topology in [LLD](LLD.md) Section 21. |
| Related documents | [PRD](PRD.md), [LLD](LLD.md), [ARCHITECTURE](ARCHITECTURE.md), [API_SPEC](API_SPEC.md), [DECISION_ENGINE](DECISION_ENGINE.md), [TOOL_CALLING](TOOL_CALLING.md), [KNOWLEDGE_GRAPH](KNOWLEDGE_GRAPH.md), [SECURITY](SECURITY.md), [DEPLOYMENT](DEPLOYMENT.md), [ROADMAP](ROADMAP.md) |

Labelling follows the [PRD](PRD.md) convention. Everything in Section 4 onward is Target architecture (Proposed) unless stated otherwise.

## 1. Document Control

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0 | 2026-08-01 | Jahnavi Ralhan | Initial data model against PRD v3.0 and LLD v1.0 |

## 2. Current MVP Data Model

The current MVP persists no enterprise data. The complete current model is three API payload shapes plus two in-memory client structures.

### 2.1 API payloads (Current MVP: frontend verified, backend described)

Ask response:

```json
{
  "answer": "string",
  "sources": [
    {
      "source_file": "Default values transitional period.pdf",
      "page_number": 7,
      "document_type": "Default Values",
      "chunk_id": "chunk_17"
    }
  ]
}
```

Status response:

```json
{
  "documents_indexed": 3,
  "chunks_created": 159,
  "vector_db": "Connected",
  "llm": "Connected",
  "embedding_model": "text-embedding-3-small"
}
```

Knowledge-base response: an array (or `{"documents": [...]}`) of:

```json
{
  "source_file": "implementation regulation.pdf",
  "document_type": "Implementing Regulation",
  "pages_processed": 68,
  "status": "Indexed",
  "last_indexed": "Demo run"
}
```

### 2.2 Client-side structures (Current MVP verified; Representative demo data)

Demo product catalogue entry (8 rows, hardcoded):

```json
{
  "id": "pig-iron",
  "name": "Pig iron",
  "cn": "7201",
  "category": "Iron & steel",
  "def": 2.07,
  "page": 7
}
```

Session audit entry: `{time, kind, title, rows: [[label, value]], summary}`, in-memory only, lost on refresh.

### 2.3 Described backend chunk metadata (unverified, OQ-01)

Per retrieved chunk: source file, page number, document type, chunk ID. Whether sector or effective-date metadata exists server-side is unverified; the target vector metadata model (Section 6) supersedes it either way.

Known defects the target model corrects: floating-point numerics (`def: 2.07`), no units, no tenancy, no versioning, no provenance, page-count and chunk-count conflation in demo data.

## 3. Modelling Conventions

These conventions apply to every target entity and are not repeated per entity.

**Base fields (every transactional table):** `id` (UUID, primary key), `tenant_id` (UUID, required, indexed; see tenancy below), `created_at` and `updated_at` (timestamptz, UTC), `row_version` (integer, optimistic locking per LLD Section 26).

**Tenancy classes:** `tenant` (row belongs to one tenant; `tenant_id` required), `reference` (global read-only reference data such as CN codes and regulatory documents; `tenant_id` null), `hybrid` (reference data with optional tenant-specific overrides via a separate override table, used for aliases and policies).

**Retention classes (NFR-005):** `OPS` (life of the tenant plus 90 days), `AUDIT` (7 years default, tenant-configurable, jurisdiction confirmation pending OQ-09), `EPHEMERAL` (TTL-bound caches and jobs). Deletion follows [SECURITY.md](SECURITY.md).

**Quantity type** (JSON representation; stored as `NUMERIC` plus a unit column):

```json
{ "value": "120.000", "unit": "t" }
```

`value` is always a decimal string, never a float. Permitted units are a closed enum (Section 10).

**Money type:** `{ "value": "10800.00", "currency": "EUR" }`.

**Data-quality state** (PRD Section 27), on every consumed evidence-bearing record: one of `validated`, `missing`, `conflicting`, `stale`, `unverified`, `unsupported`, `suspicious`.

**Effective dating pattern:** paired `effective_from` (date, required) and `effective_to` (date, null = open), with the invariant that versions of the same logical object never overlap (Section 11).

**Identifiers in JSON examples** are shortened for readability (`"uuid-..."`).

## 4. Target Entity Catalogue

Grouped by domain. Each entry states purpose, ownership (the module allowed to write it, per LLD component IDs), tenancy, retention, fields beyond the base fields, validation, relationships, indexes, and an example. Exemplar entities (marked) carry full field tables; supporting entities use the condensed form with identical rigour but less repetition.

### 4.1 Identity and Tenancy

#### Tenant

Purpose: the isolation and configuration root for one customer organisation. Ownership: Admin service. Tenancy: none (it is the tenant). Retention: AUDIT.
Fields: `name` (text, required), `status` (enum: active, suspended, closed), `settings` (jsonb: materiality threshold per OQ-12, retention overrides, provider options per OQ-10), `residency_region` (text, nullable pending OQ-09).
Validation: unique `name`; status transitions only via Admin service. Relationships: parent of everything tenant-scoped. Indexes: unique(name).

```json
{ "id": "uuid-t1", "name": "Acme Steel Exports", "status": "active",
  "settings": { "materiality_threshold": { "value": "50000.00", "currency": "EUR" } } }
```

#### User

Purpose: an authenticated person with roles. Ownership: Identity integration (C-21). Tenancy: tenant. Retention: AUDIT (identity references persist in traces).
Fields: `external_idp_id` (text, required, unique per tenant), `display_name` (text), `email` (text, validated format), `roles` (text[], from the RBAC catalogue in [SECURITY.md](SECURITY.md)), `status` (enum: active, disabled).
Validation: at least one role; reviewer role assignment audited. Relationships: referenced by ReviewTask, AuditPackage, ToolExecution actor fields. Indexes: unique(tenant_id, external_idp_id).

```json
{ "id": "uuid-u1", "tenant_id": "uuid-t1", "external_idp_id": "okta|118...",
  "display_name": "Priya N", "email": "priya@acme.example", "roles": ["analyst", "reviewer"], "status": "active" }
```

#### Organisation

Purpose: a legal entity within a tenant (declarant, exporter, group company); lets one tenant model several reporting entities. Ownership: Admin service. Tenancy: tenant. Retention: AUDIT.
Fields: `legal_name` (text, required), `role` (enum: declarant, exporter, importer, group), `country` (ISO 3166-1 alpha-2), `eori` (text, nullable).
Validation: country code valid; `eori` format checked when present. Relationships: Shipment.declarant_org_id; Supplier belongs to an Organisation context. Indexes: (tenant_id, legal_name).

```json
{ "id": "uuid-o1", "tenant_id": "uuid-t1", "legal_name": "Acme Steel GmbH",
  "role": "declarant", "country": "DE", "eori": "DE1234567890123" }
```

### 4.2 Supply Chain

#### Supplier

Purpose: an external producing or trading party providing goods or declarations. Ownership: Import service and Admin service. Tenancy: tenant. Retention: AUDIT.
Fields: `name` (text, required), `country` (ISO), `contact` (jsonb), `data_quality_state` (enum, default unverified).
Validation: name normalised for matching. Relationships: OPERATES Installation; issues SupplierDeclaration. Indexes: (tenant_id, name).

```json
{ "id": "uuid-s1", "tenant_id": "uuid-t1", "name": "Nordvik Metals AS",
  "country": "NO", "data_quality_state": "validated" }
```

#### Installation

Purpose: a physical production site whose emissions records apply to goods produced there. Ownership: Import and Admin services. Tenancy: tenant. Retention: AUDIT.
Fields: `supplier_id` (fk, required), `name` (text, required), `country` (ISO), `installation_code` (text, nullable; official identifier where one exists), `geo` (jsonb, nullable).
Validation: unique (tenant_id, supplier_id, name). Relationships: USES ProductionRoute; PRODUCES Product; EmissionRecord MEASURED_AT Installation. Indexes: (tenant_id, supplier_id).

```json
{ "id": "uuid-i1", "tenant_id": "uuid-t1", "supplier_id": "uuid-s1",
  "name": "Nordvik Works 2", "country": "NO", "installation_code": "NO-INST-0042" }
```

#### ProductionRoute

Purpose: the manufacturing pathway (for example blast furnace basic oxygen furnace, electric arc furnace, direct reduction) that determines applicable methodology and typical intensity. Ownership: Admin service (reference-managed), tenant extensions allowed. Tenancy: hybrid. Retention: AUDIT.
Fields: `code` (text, required, stable; e.g. `BF-BOF`), `name` (text), `category` (enum matching CBAM sectors), `description` (text).
Validation: unique code within scope. Relationships: Installation USES; Product PRODUCED_VIA; Methodology applicability in PolicyVersion conditions. Indexes: unique(code) on reference rows.

```json
{ "id": "uuid-r1", "tenant_id": null, "code": "EAF", "name": "Electric arc furnace",
  "category": "iron_steel" }
```

#### Product (exemplar, full table)

Purpose: the canonical good, the anchor of entity resolution (FR-011); replaces the MVP demo catalogue row. Ownership: Entity Resolution service (C-12) and Admin service. Tenancy: hybrid (global canonical products; tenant products allowed). Retention: AUDIT.

| Field | Type | Required | Notes |
|---|---|---|---|
| canonical_key | text | yes | Stable slug, unique in scope (e.g. `pig-iron`) |
| name | text | yes | Display name |
| category | enum | yes | CBAM sector: iron_steel, cement, aluminium, fertilisers, hydrogen, electricity |
| cn_code_id | fk CNCode | yes | Primary CN mapping |
| default_route_id | fk ProductionRoute | no | Typical route, informational |
| description | text | no | |
| data_quality_state | enum | yes | Convention state |

Validation: unique (scope, canonical_key); category and CN code consistency checked against reference data. Relationships: MAPS_TO CNCode; alias fan-in via ProductAlias; ShipmentLine.product_id; EmissionFactor and default values keyed by (product, route, version). Indexes: unique(canonical_key) global; (tenant_id, canonical_key) tenant rows.

```json
{ "id": "uuid-p1", "tenant_id": null, "canonical_key": "pig-iron", "name": "Pig iron",
  "category": "iron_steel", "cn_code_id": "uuid-cn7201", "data_quality_state": "validated" }
```

#### ProductAlias

Purpose: a known external name for a Product, making resolution deterministic over time (LLD Section 20). Ownership: Entity Resolution service; rows created by approved mappings. Tenancy: tenant (aliases are tenant vocabulary). Retention: AUDIT.
Fields: `product_id` (fk, required), `alias` (text, required, normalised), `source_system` (text: erp, supplier, import), `method` (enum: exact, alias, fuzzy_approved, human), `confidence` (numeric 0 to 1, only for fuzzy_approved), `provenance` (jsonb: who or what created it, evidence).
Validation: unique (tenant_id, alias, source_system); an alias maps to exactly one product per tenant; conflicts create a review item instead of a second row. Relationships: Product. Indexes: (tenant_id, alias).

```json
{ "id": "uuid-a1", "tenant_id": "uuid-t1", "product_id": "uuid-p1",
  "alias": "PIG IRON GR-A", "source_system": "erp", "method": "human",
  "provenance": { "approved_by": "uuid-u1", "review_task": "uuid-rt9" } }
```

#### CNCode

Purpose: Combined Nomenclature reference entry. Ownership: reference ingestion. Tenancy: reference. Retention: AUDIT (versioned reference).
Fields: `code` (text, required; digits with optional spaces normalised out), `description` (text), `cbam_covered` (boolean), `effective_from` and `effective_to` (dates; CN codes change over time).
Validation: code format; non-overlapping effective windows per code. Relationships: Product MAPS_TO; Article APPLIES_TO (graph projection). Indexes: (code, effective_from).

```json
{ "id": "uuid-cn7201", "tenant_id": null, "code": "7201", "description": "Pig iron and spiegeleisen",
  "cbam_covered": true, "effective_from": "2023-10-01", "effective_to": null }
```

### 4.3 Shipments

#### Shipment (exemplar, full table)

Purpose: one consignment under investigation; the root of J2 in the PRD. Ownership: Import service. Tenancy: tenant. Retention: AUDIT.

| Field | Type | Required | Notes |
|---|---|---|---|
| declarant_org_id | fk Organisation | yes | Reporting entity |
| external_ref | text | yes | Client reference, unique per tenant |
| reporting_period | text | yes | Period key, e.g. `2026-Q2`; drives effective dating |
| destination_country | ISO code | yes | EU member state |
| supplier_id | fk Supplier | no | Primary supplier where known |
| shipped_on | date | no | |
| source | enum | yes | form, csv, api, erp, event |
| input_snapshot_ref | text | yes | Object-storage key of the immutable raw input (lineage) |
| status | enum | yes | received, investigating, decided, superseded |

Validation: unique (tenant_id, external_ref); reporting_period parseable; snapshot written before any mutation. Relationships: has ShipmentLines; SOURCED_FROM Installation via lines; investigations reference shipment_id. Indexes: (tenant_id, reporting_period), (tenant_id, external_ref) unique.

```json
{ "id": "uuid-sh1", "tenant_id": "uuid-t1", "declarant_org_id": "uuid-o1",
  "external_ref": "ACME-2026-0417", "reporting_period": "2026-Q2",
  "destination_country": "NL", "source": "csv",
  "input_snapshot_ref": "t1/imports/2026/07/imp-91/row-417.json", "status": "investigating" }
```

#### ShipmentLine

Purpose: one product position within a shipment; the unit of calculation. Ownership: Import service. Tenancy: tenant. Retention: AUDIT.
Fields: `shipment_id` (fk, required), `line_no` (int, required), `product_id` (fk Product, required after resolution), `raw_product_text` (text, as received), `cn_code_id` (fk, resolved), `installation_id` (fk, nullable), `route_id` (fk ProductionRoute, nullable), `quantity` (Quantity, required, unit t), `resolution_ref` (fk to the resolution record).
Validation: quantity value > 0; unresolved lines cannot enter calculation (they hold status via data-quality state). Relationships: Shipment, Product, Installation, Batch. Indexes: (shipment_id, line_no) unique.

```json
{ "id": "uuid-sl1", "tenant_id": "uuid-t1", "shipment_id": "uuid-sh1", "line_no": 1,
  "raw_product_text": "PIG IRON GR-A", "product_id": "uuid-p1", "cn_code_id": "uuid-cn7201",
  "installation_id": "uuid-i1", "route_id": "uuid-r1",
  "quantity": { "value": "120.000", "unit": "t" } }
```

#### Batch

Purpose: an optional production batch linking a shipment line to specific emission records and inputs. Ownership: Import service. Tenancy: tenant. Retention: AUDIT.
Fields: `shipment_line_id` (fk, required), `batch_ref` (text), `produced_from` and `produced_to` (dates), `installation_id` (fk).
Validation: date order. Relationships: EnergyInput, PrecursorInput, EmissionRecord scoping. Indexes: (tenant_id, batch_ref).

```json
{ "id": "uuid-b1", "tenant_id": "uuid-t1", "shipment_line_id": "uuid-sl1",
  "batch_ref": "NW2-2026-118", "installation_id": "uuid-i1",
  "produced_from": "2026-05-02", "produced_to": "2026-05-09" }
```

#### EnergyInput

Purpose: energy consumed producing a batch or line (drives indirect emissions). Ownership: Import service. Tenancy: tenant. Retention: AUDIT.
Fields: `batch_id` (fk, required), `energy_source` (enum: grid_electricity, ppa_electricity, natural_gas, coal, coke, other), `amount` (Quantity: MWh or GJ), `emission_factor_id` (fk EmissionFactor, resolved), `data_quality_state` (enum).
Validation: amount > 0; unit consistent with source class. Relationships: Batch, EmissionFactor.

```json
{ "id": "uuid-e1", "tenant_id": "uuid-t1", "batch_id": "uuid-b1",
  "energy_source": "grid_electricity", "amount": { "value": "84.500", "unit": "MWh" },
  "emission_factor_id": "uuid-f7", "data_quality_state": "validated" }
```

#### PrecursorInput

Purpose: an input material whose embedded emissions roll into the final good (PRD glossary). Ownership: Import service. Tenancy: tenant. Retention: AUDIT.
Fields: `batch_id` (fk, required), `precursor_product_id` (fk Product, required), `quantity` (Quantity, t), `embedded_intensity` (Quantity, tCO2e per t, nullable when derived), `source` (enum: supplier_declaration, default_value, calculated), `evidence_ref` (fk SupplierDeclaration or null), `data_quality_state`.
Validation: if source = supplier_declaration then evidence_ref required. Relationships: Batch, Product, SupplierDeclaration.

```json
{ "id": "uuid-pi1", "tenant_id": "uuid-t1", "batch_id": "uuid-b1",
  "precursor_product_id": "uuid-p9", "quantity": { "value": "18.000", "unit": "t" },
  "source": "supplier_declaration", "evidence_ref": "uuid-sd1", "data_quality_state": "validated" }
```

### 4.4 Emissions Evidence

#### EmissionRecord (exemplar, full table)

Purpose: a verified emissions measurement for an installation and period; the evidential basis for actual-emissions methodology (A-4: BorderIQ consumes, never produces, primary measurements). Ownership: Import service; state transitions by Evidence validation. Tenancy: tenant. Retention: AUDIT.

| Field | Type | Required | Notes |
|---|---|---|---|
| installation_id | fk Installation | yes | |
| route_id | fk ProductionRoute | yes | |
| product_id | fk Product | yes | |
| period_from / period_to | date | yes | Measurement window |
| direct_intensity | Quantity | yes | tCO2e per t of goods |
| indirect_intensity | Quantity | no | tCO2e per t |
| method | text | yes | Measurement methodology label |
| verification_report_id | fk VerificationReport | no | Required for validated state |
| data_quality_state | enum | yes | validated requires an in-window verification report |
| source_ref | text | yes | Object-storage key of the source document or extract |

Validation: period order; state = validated only when a VerificationReport covers the window and is itself valid; stale computed when period_to older than the tenant staleness policy. Relationships: MEASURED_AT Installation; VERIFIED_BY Verifier via report; consumed by CalculationRun. Indexes: (tenant_id, installation_id, product_id, period_from).

```json
{ "id": "uuid-er1", "tenant_id": "uuid-t1", "installation_id": "uuid-i1",
  "route_id": "uuid-r1", "product_id": "uuid-p1",
  "period_from": "2026-01-01", "period_to": "2026-06-30",
  "direct_intensity": { "value": "1.310", "unit": "tCO2e_per_t" },
  "indirect_intensity": { "value": "0.110", "unit": "tCO2e_per_t" },
  "method": "installation monitoring plan", "verification_report_id": "uuid-vr1",
  "data_quality_state": "validated", "source_ref": "t1/evidence/er1.pdf" }
```

#### EmissionFactor

Purpose: a reference factor (including official default values) with source lineage and validity. Ownership: Reference ingestion (default values parsed from RegulatoryFragments); tenant factors via Admin. Tenancy: hybrid. Retention: AUDIT.
Fields: `kind` (enum: default_value, grid_factor, fuel_factor, custom), `product_id` (fk, nullable), `route_id` (fk, nullable), `region` (text, nullable), `value` (Quantity), `basis` (text: what the factor multiplies), `source_fragment_id` (fk RegulatoryFragment, required for default_value), `effective_from` / `effective_to`.
Validation: default_value rows must carry fragment provenance; non-overlapping windows per (kind, product, route, region). Relationships: RegulatoryFragment; CalculationRun inputs. Indexes: (kind, product_id, route_id, effective_from).

```json
{ "id": "uuid-f1", "tenant_id": null, "kind": "default_value", "product_id": "uuid-p1",
  "route_id": null, "value": { "value": "2.070", "unit": "tCO2e_per_t" },
  "basis": "per tonne of goods", "source_fragment_id": "uuid-fr77",
  "effective_from": "2026-01-01", "effective_to": null }
```

#### SupplierDeclaration

Purpose: supplier-provided emissions or precursor data with its own validity and verification state. Ownership: Import service. Tenancy: tenant. Retention: AUDIT.
Fields: `supplier_id` (fk), `installation_id` (fk, nullable), `covers` (enum: product_intensity, precursor_intensity), `product_id` (fk), `declared_intensity` (Quantity), `issued_on` (date), `valid_until` (date), `document_ref` (object-storage key), `data_quality_state`.
Validation: expired when valid_until past (state stale); unverified until validation. Relationships: PrecursorInput.evidence_ref; evidence checks in FR-013.

```json
{ "id": "uuid-sd1", "tenant_id": "uuid-t1", "supplier_id": "uuid-s1",
  "covers": "precursor_intensity", "product_id": "uuid-p9",
  "declared_intensity": { "value": "1.940", "unit": "tCO2e_per_t" },
  "issued_on": "2026-03-14", "valid_until": "2027-03-13",
  "document_ref": "t1/evidence/sd1.pdf", "data_quality_state": "unverified" }
```

#### VerificationReport

Purpose: an accredited verifier's report validating emission records or declarations. Ownership: Import service. Tenancy: tenant. Retention: AUDIT.
Fields: `verifier_name` (text), `verifier_accreditation` (text, nullable), `covers_installation_id` (fk), `period_from` / `period_to` (dates), `issued_on` (date), `valid_until` (date, nullable), `document_ref` (key), `data_quality_state`.
Validation: window order; expiry drives stale state and FR-013 outcomes. Relationships: EmissionRecord.verification_report_id.

```json
{ "id": "uuid-vr1", "tenant_id": "uuid-t1", "verifier_name": "Nordic Verify AB",
  "covers_installation_id": "uuid-i1", "period_from": "2026-01-01", "period_to": "2026-06-30",
  "issued_on": "2026-07-10", "document_ref": "t1/evidence/vr1.pdf",
  "data_quality_state": "validated" }
```

### 4.5 Regulatory Corpus

#### RegulatoryDocument

Purpose: the logical regulation or guidance document, independent of version. Ownership: Regulatory ingestion (C-11). Tenancy: reference. Retention: AUDIT.
Fields: `family` (enum: regulation, implementing_regulation, guidance, default_values), `title` (text), `jurisdiction` (text: EU), `authority` (text: source allowlist entry).
Relationships: has RegulationVersions. Indexes: (family, title).

```json
{ "id": "uuid-rd1", "tenant_id": null, "family": "default_values",
  "title": "Default values for the transitional period", "jurisdiction": "EU",
  "authority": "European Commission" }
```

#### RegulationVersion (exemplar, full table)

Purpose: one immutable published version of a document, the unit of effective dating and supersession (FR-022, PP-3). Ownership: Regulatory ingestion. Tenancy: reference. Retention: AUDIT (never deleted; supersession preserves history).

| Field | Type | Required | Notes |
|---|---|---|---|
| document_id | fk RegulatoryDocument | yes | |
| version_label | text | yes | Human label, e.g. `2026-01` |
| checksum | text (sha256) | yes | Of the raw file; ingestion idempotency key |
| raw_ref | text | yes | Object-storage key of the immutable file |
| published_on | date | yes | |
| effective_from / effective_to | date | yes / no | Window this version governs |
| supersedes_version_id | fk self | no | SUPERSEDES link |
| ingestion_status | enum | yes | ingested, parsed, indexed, verified |

Validation: unique checksum; non-overlapping effective windows within a document; supersession acyclic. Relationships: CONTAINS RegulatoryFragments; SUPERSEDES prior version. Indexes: unique(checksum), (document_id, effective_from).

```json
{ "id": "uuid-rv2", "tenant_id": null, "document_id": "uuid-rd1", "version_label": "2026-01",
  "checksum": "sha256:9f3a...", "raw_ref": "ref/regs/rd1/v2026-01.pdf",
  "published_on": "2025-12-15", "effective_from": "2026-01-01", "effective_to": null,
  "supersedes_version_id": "uuid-rv1", "ingestion_status": "verified" }
```

#### RegulatoryFragment

Purpose: the retrievable unit: article, annex, table row group, section, or page fallback (LLD Section 19). Ownership: Regulatory ingestion. Tenancy: reference. Retention: AUDIT.
Fields: `version_id` (fk RegulationVersion, required), `fragment_type` (enum: article, annex, table, section, page), `path` (text: structural path, e.g. `annex-1/table-1`), `page_from` / `page_to` (int), `text` (text), `table_payload` (jsonb, nullable: typed rows for tables), `meta` (jsonb: the vector metadata of Section 6).
Validation: page span within document; table_payload schema-checked per table class. Relationships: cited by RetrievalTrace and ComplianceDecision.supporting_sources; source of EmissionFactor default values. Indexes: (version_id, path); full-text index on text (lexical retrieval).

```json
{ "id": "uuid-fr77", "tenant_id": null, "version_id": "uuid-rv2", "fragment_type": "table",
  "path": "annex-1/table-1/iron-steel", "page_from": 7, "page_to": 7,
  "table_payload": { "rows": [ { "product": "pig-iron", "value": "2.070", "unit": "tCO2e_per_t" } ] },
  "meta": { "category": "iron_steel", "emissions_type": "total", "methodology": "default" } }
```

### 4.6 Policy

#### PolicyDefinition

Purpose: the logical policy (what it governs), independent of version. Ownership: Policy engine admin (C-06) via approval workflow. Tenancy: hybrid (global baseline, tenant-specific policies allowed). Retention: AUDIT.
Fields: `key` (text, unique in scope), `title` (text), `governs` (enum: methodology_eligibility, evidence_requirement, approval_threshold, routing).
Relationships: has PolicyVersions.

```json
{ "id": "uuid-pd1", "tenant_id": null, "key": "actuals-eligibility-iron-steel",
  "title": "Actual emissions eligibility for iron and steel", "governs": "methodology_eligibility" }
```

#### PolicyVersion (exemplar, full table)

Purpose: one approved, effective-dated rule set; the only thing the policy engine evaluates (FR-014, AIR-010). Ownership: written by proposal flow; activated only by approval. Tenancy: follows definition. Retention: AUDIT (immutable once activated).

| Field | Type | Required | Notes |
|---|---|---|---|
| definition_id | fk PolicyDefinition | yes | |
| version_no | int | yes | Monotonic per definition |
| rule | jsonb | yes | Declarative condition tree over typed context fields (LLD Section 13) |
| applicability | jsonb | yes | jurisdiction, categories, CN codes, routes |
| evidence_requirements | jsonb | no | Required evidence classes and validity |
| effective_from / effective_to | date | yes / no | |
| derived_from_fragment_ids | uuid[] | no | DERIVED_FROM provenance |
| proposed_by | enum | yes | human, ai_suggested |
| approved_by_user_id | fk User | yes for active | AIR-010 |
| status | enum | yes | proposed, approved, active, retired, rejected |

Validation: active requires approved_by; windows non-overlapping per definition and applicability; rule schema-validated. Relationships: evaluated into PolicyEvaluation entries inside decisions; DERIVED_FROM fragments. Indexes: (definition_id, version_no) unique, (status, effective_from).

```json
{ "id": "uuid-pv3", "tenant_id": null, "definition_id": "uuid-pd1", "version_no": 3,
  "rule": { "all": [ { "field": "emission_record.state", "eq": "validated" },
                      { "field": "verification.in_window", "eq": true } ] },
  "applicability": { "jurisdiction": "EU", "categories": ["iron_steel"] },
  "effective_from": "2026-01-01", "proposed_by": "ai_suggested",
  "approved_by_user_id": "uuid-u9", "status": "active",
  "derived_from_fragment_ids": ["uuid-fr41"] }
```

### 4.7 Execution and Traces

#### CalculationRun (exemplar, full table)

Purpose: one deterministic engine execution with complete lineage; the only legitimate source of authoritative figures (FR-015, AIR-001). Ownership: Calculation engine caller (orchestrator). Tenancy: tenant. Retention: AUDIT (immutable).

| Field | Type | Required | Notes |
|---|---|---|---|
| investigation_id | uuid | yes | Correlates to the decision |
| formula_key | text | yes | e.g. `embedded_emissions.actual` |
| formula_version | text (semver) | yes | Pinned engine formula version |
| inputs | jsonb | yes | Typed values with units and source refs (factor IDs, record IDs) |
| intermediates | jsonb | no | Named intermediate values |
| results | jsonb | yes | Typed results with units and rounding class applied |
| engine_version | text | yes | Calculation engine build version |
| status | enum | yes | ok, rejected_unit, rejected_input, rejected_validity |

Validation: results present only when status ok; every input carries a source reference; Decimal-string numerics only. Relationships: referenced by ComplianceDecision.figures and Scenario. Indexes: (tenant_id, investigation_id).

```json
{ "id": "uuid-cr1", "tenant_id": "uuid-t1", "investigation_id": "uuid-inv1",
  "formula_key": "embedded_emissions.compare", "formula_version": "1.2.0",
  "inputs": { "quantity": { "value": "120.000", "unit": "t", "source": "uuid-sl1" },
              "actual_intensity": { "value": "1.420", "unit": "tCO2e_per_t", "source": "uuid-er1" },
              "default_intensity": { "value": "2.070", "unit": "tCO2e_per_t", "source": "uuid-f1" },
              "carbon_price": { "value": "80.00", "currency": "EUR", "source": "user_supplied" } },
  "results": { "avoided": { "value": "78.000", "unit": "tCO2e" },
               "difference": { "value": "6240.00", "currency": "EUR" } },
  "engine_version": "calc-0.4.1", "status": "ok" }
```

#### ToolExecution

Purpose: one governed tool call with validated inputs and outputs (FR-021). Ownership: Tool executor (C-04). Tenancy: tenant. Retention: AUDIT (immutable).
Fields: `investigation_id` (uuid, nullable for Q&A), `tool_name` (text), `tool_version` (text), `actor` (enum: orchestrator, llm_planner, user), `input` (jsonb, schema-validated), `output` (jsonb, schema-validated, null on failure), `outcome` (enum: ok, rejected_input, rejected_permission, timeout, failed, output_invalid), `duration_ms` (int), `idempotency_key` (text, nullable), `approval_ref` (fk ReviewTask, required for high-impact tools).
Validation: output present only when outcome ok. Indexes: (tenant_id, investigation_id), (tool_name, created_at).

```json
{ "id": "uuid-tx1", "tenant_id": "uuid-t1", "investigation_id": "uuid-inv1",
  "tool_name": "retrieve_default_emission_value", "tool_version": "1.0.0",
  "actor": "orchestrator", "input": { "product_id": "uuid-p1", "period": "2026-Q2" },
  "output": { "factor_id": "uuid-f1" }, "outcome": "ok", "duration_ms": 18 }
```

#### RetrievalTrace

Purpose: one scoped retrieval with its filters, candidates, and sufficiency verdict (FR-023). Ownership: Retrieval service (C-10). Tenancy: tenant. Retention: AUDIT.
Fields: `investigation_id` (uuid, nullable), `query_text` (text), `scope` (jsonb: the metadata filters applied), `corpus_version` (text), `candidates` (jsonb: fragment_id, dense_score, lexical_score, fused, rerank_score), `sufficiency` (enum: sufficient, insufficient), `abstained` (boolean).
Validation: abstained implies sufficiency insufficient. Relationships: cited fragments feed decisions. Indexes: (tenant_id, investigation_id).

```json
{ "id": "uuid-rt1", "tenant_id": "uuid-t1", "investigation_id": "uuid-inv1",
  "query_text": "default value pig iron", "corpus_version": "2026-07-01",
  "scope": { "jurisdiction": "EU", "effective_on": "2026-06-30", "category": "iron_steel" },
  "candidates": [ { "fragment_id": "uuid-fr77", "fused": 0.91, "rerank_score": 0.88 } ],
  "sufficiency": "sufficient", "abstained": false }
```

### 4.8 Decisions and Review

#### Scenario

Purpose: a what-if run against a decision twin version; never mutates the official decision (FR-016, FR-026). Ownership: Scenario engine (C-08). Tenancy: tenant. Retention: OPS.
Fields: `base_decision_id` (fk ComplianceDecision, required), `twin_version` (text), `assumptions` (jsonb: declared deltas), `calculation_run_id` (fk), `policy_outcome` (jsonb), `label` (text).
Validation: assumptions non-empty; results always rendered with a hypothetical label. Indexes: (tenant_id, base_decision_id).

```json
{ "id": "uuid-sc1", "tenant_id": "uuid-t1", "base_decision_id": "uuid-d1",
  "twin_version": "twin-uuid-d1-v1", "label": "Carbon price 95",
  "assumptions": { "carbon_price": { "value": "95.00", "currency": "EUR" } },
  "calculation_run_id": "uuid-cr9" }
```

#### ComplianceDecision (exemplar, full table)

Purpose: the structured decision object, the platform's primary output (FR-018). Field semantics and the state machine are owned by [DECISION_ENGINE.md](DECISION_ENGINE.md); this is the persistence shape. Ownership: Decision assembler (C-09); reviewer state by Review service. Tenancy: tenant. Retention: AUDIT (immutable except reviewer_state and status transitions, all audited).

| Field | Type | Required | Notes |
|---|---|---|---|
| investigation_id | uuid | yes | Unique |
| shipment_id / shipment_line_id | fk | yes | Subject |
| status | enum | yes | draft, auto_approved, review_required, blocked, insufficient_evidence, unsupported_case, finalised, superseded |
| methodology | enum | yes when decided | actuals, defaults, mixed |
| eligibility | jsonb | yes | Outcome per methodology with policy citations |
| figures | jsonb | yes | References into CalculationRun results only |
| assumptions | jsonb | no | Declared assumptions (e.g. price source) |
| missing_evidence | jsonb | no | Evidence classes and reasons |
| risk | jsonb | yes | Class plus contributing factors |
| supporting_sources | jsonb | yes | fragment_id, version, pages per claim |
| applied_policies | jsonb | yes | PolicyVersion IDs |
| required_actions | jsonb | no | Next steps |
| reviewer_state | jsonb | no | Task ref, actions, actor, timestamps |
| confidence | jsonb | yes | Five dimensions, each level plus basis (AIR-005) |
| narrative | text | no | Rendered after assembly; citation-checked |
| provenance | jsonb | yes | Model, prompt, engine, corpus, schema versions |

Validation: schema-validated before persistence (LLD Section 16); figures may only reference CalculationRun IDs; every supporting source resolves to a fragment. Indexes: (tenant_id, shipment_id), (tenant_id, status), unique(investigation_id).

```json
{ "id": "uuid-d1", "tenant_id": "uuid-t1", "investigation_id": "uuid-inv1",
  "shipment_line_id": "uuid-sl1", "status": "auto_approved", "methodology": "actuals",
  "eligibility": { "actuals": { "eligible": true, "policy": "uuid-pv3" } },
  "figures": { "calculation_run": "uuid-cr1" },
  "risk": { "class": "medium", "factors": ["exposure_moderate", "evidence_validated"] },
  "supporting_sources": [ { "fragment_id": "uuid-fr77", "version": "uuid-rv2", "pages": [7] } ],
  "applied_policies": ["uuid-pv3"],
  "confidence": { "mapping": { "level": "high", "basis": "alias, human-approved" },
                  "evidence_completeness": { "level": "high", "basis": "validated record in window" },
                  "retrieval_sufficiency": { "level": "high", "basis": "rerank 0.88, threshold 0.7" },
                  "policy_certainty": { "level": "high", "basis": "single active version" },
                  "calculation_validity": { "level": "high", "basis": "golden formula 1.2.0" } },
  "provenance": { "model": "provider-model-2026-05", "prompt": "narrate-v4",
                  "engine": "calc-0.4.1", "corpus": "2026-07-01", "decision_schema": "1.0" } }
```

#### ReviewTask

Purpose: a human-review work item bound to a decision (FR-019). Ownership: Review service (C-19). Tenancy: tenant. Retention: AUDIT.
Fields: `decision_id` (fk, required), `reason` (enum from PRD Section 25 routing list), `context_ref` (jsonb pointers), `assigned_role` (text), `status` (enum: open, in_review, approved, rejected, info_requested, escalated), `actor_user_id` (fk User, on action), `rationale` (text, required on reject), `acted_at` (timestamptz).
Validation: terminal actions require actor and, for reject, rationale. Indexes: (tenant_id, status), (decision_id).

```json
{ "id": "uuid-rt9", "tenant_id": "uuid-t1", "decision_id": "uuid-d2",
  "reason": "ambiguous_mapping", "assigned_role": "reviewer", "status": "approved",
  "actor_user_id": "uuid-u1", "acted_at": "2026-07-30T09:14:00Z" }
```

#### AuditPackage

Purpose: the exportable closure of one decision: trace, evidence references, versions, reviewer trail (FR-020; lineage diagram in [ARCHITECTURE.md](ARCHITECTURE.md) Section 17). Ownership: Audit service on export. Tenancy: tenant. Retention: AUDIT.
Fields: `decision_id` (fk, required), `manifest` (jsonb: every included record ID and object key with checksums), `package_ref` (object-storage key of the export), `exported_by` (fk User), `format_version` (text).
Validation: manifest completeness check against the trace before export; export requires finalised decision and a human trigger (FR-030 analogue). Indexes: (tenant_id, decision_id).

```json
{ "id": "uuid-ap1", "tenant_id": "uuid-t1", "decision_id": "uuid-d1",
  "package_ref": "t1/audit/d1/pkg-2026-07-31.zip", "exported_by": "uuid-u1",
  "format_version": "1.0",
  "manifest": { "decision": "uuid-d1", "calculation_runs": ["uuid-cr1"],
                 "retrieval_traces": ["uuid-rt1"], "fragments": ["uuid-fr77"],
                 "evidence": ["t1/evidence/er1.pdf"], "checksum": "sha256:aa19..." } }
```

## 5. Relational Model Recommendation

PostgreSQL from Phase 3 (LLD Section 21; store selection OQ-03 for the vector side). Rules: every entity above is one table using the base-field convention; foreign keys enforced with `ON DELETE RESTRICT` for audit-class rows (nothing audit-bearing cascades away); jsonb for the declared flexible payloads only (rule trees, manifests, scopes), never as a substitute for modelled columns; partial indexes on hot status values (`status = 'review_required'`); `NUMERIC` for all quantities and money with unit and currency columns alongside; immutability of audit-class rows enforced by revoking UPDATE except on the explicitly mutable columns (status, reviewer_state) and by trigger-based audit of those transitions. Migrations are forward-only per LLD Section 35.

## 6. Vector Metadata Model

Each indexed unit is one RegulatoryFragment. The index stores the embedding plus this filterable metadata (the scoping fields of FR-023):

```json
{ "fragment_id": "uuid-fr77", "version_id": "uuid-rv2", "document_family": "default_values",
  "jurisdiction": "EU", "authority": "European Commission",
  "effective_from": "2026-01-01", "effective_to": null,
  "fragment_type": "table", "path": "annex-1/table-1/iron-steel",
  "page_from": 7, "page_to": 7,
  "category": "iron_steel", "cn_codes": ["7201"],
  "production_routes": [], "methodology": "default",
  "emissions_type": "total", "corpus_version": "2026-07-01" }
```

Tenant-scoped namespaces apply only if tenant-private documents are ever indexed; the regulatory corpus itself is reference-scoped. Effective-date filtering is computed as `effective_from <= period_date AND (effective_to IS NULL OR effective_to >= period_date)` before similarity search, never after.

## 7. Graph Projection

Owned by [KNOWLEDGE_GRAPH.md](KNOWLEDGE_GRAPH.md). Summary: the graph is a rebuildable projection of the relational truth. Node types mirror the entities above (Supplier, Installation, ProductionRoute, Product, CNCode, Shipment, EmissionRecord, Verifier, RegulationVersion, Article as fragment, PolicyRule as PolicyVersion, ComplianceDecision, ReviewTask); relationships include OPERATES, USES, PRODUCES, MAPS_TO, CONTAINS, SOURCED_FROM, SUPERSEDES, APPLIES_TO, DERIVED_FROM, MEASURED_AT, VERIFIED_BY, USED, SUPPORTED_BY, GENERATED. The projection is never the system of record and is rebuilt from PostgreSQL (LLD TR-8).

## 8. Data-Lineage Model

Lineage is represented by the reference chains already modelled, not a separate store: figures reference CalculationRuns; CalculationRun inputs reference evidence rows and factors; factors reference fragments; fragments reference document versions with checksums; decisions reference policies, traces, and reviewer actions; AuditPackage.manifest is the materialised closure. The lineage invariant: every value in a finalised decision must be reachable from the decision through these references to a checksummed source. Replay (NFR-001) is a traversal of this graph with pinned versions.

## 9. Versioning Strategy

Four independent version families, each recorded in decision provenance: (1) regulatory corpus (RegulationVersion per document plus a global `corpus_version` snapshot label used by retrieval); (2) policies (PolicyVersion, monotonic per definition, effective-dated); (3) engines and formulas (semver per formula plus engine build); (4) interaction artefacts (prompt templates, model identifiers, decision schema version; OQ-13 governs whether the decision schema versions independently of the API, with the LLD recommending independent versioning plus a compatibility map). Nothing version-bearing is ever updated in place; new versions are new rows.

## 10. Units, Decimal Handling, and Rounding

Closed unit enum, initial set: `t` (tonne of goods), `kg`, `tCO2e`, `tCO2e_per_t`, `MWh`, `GJ`, plus ISO currency codes for Money. Conversions exist only where physically meaningful and are explicit functions in the calculation engine (kg to t, GJ to MWh); intensity units never convert implicitly to totals. All numerics are `NUMERIC` in the database and decimal strings in JSON (Section 3); floats are forbidden in any authoritative path.

Rounding rules by figure class (applied once, at result emission, half-even):

| Figure class | Scale |
|---|---|
| Quantities (t) | 3 decimal places |
| Intensities (tCO2e_per_t) | 3 |
| Emissions totals (tCO2e) | 3 |
| Money (EUR) | 2 |
| Scores and confidences (internal) | 4, never displayed as fake precision |

Intermediate values are not rounded; only emitted results are, and the rounding class is recorded in CalculationRun results.

## 11. Effective Dating

The pattern of Section 3 applies to RegulationVersion, RegulatoryFragment (inherited), EmissionFactor, PolicyVersion, and CNCode. Resolution rule: an investigation binds to the versions effective on its reporting-period reference date (period end unless the applicable regulation dictates otherwise; the reference-date rule is itself policy-governed and versioned). Golden period tests (LLD Section 33) pin this behaviour. Overlap is a validation error at write time, not a runtime tiebreak.

## 12. Schema Evolution

API and event schemas follow NFR-012 (versioned, breaking changes only with a major version). Database evolution: additive migrations preferred (new nullable columns, new tables); destructive changes require a deprecation phase across one minor release; jsonb payloads carry their own `schema` key where they evolve independently (rule trees, manifests). The decision schema evolves per OQ-13 with a stored `decision_schema` version in provenance so replays validate against the schema the decision was born under.

## 13. Retention Summary

| Class | Entities | Default |
|---|---|---|
| AUDIT | Tenant, User, Organisation, Supplier, Installation, ProductionRoute, Product, ProductAlias, CNCode, Shipment, ShipmentLine, Batch, EnergyInput, PrecursorInput, EmissionRecord, EmissionFactor, SupplierDeclaration, VerificationReport, RegulatoryDocument, RegulationVersion, RegulatoryFragment, PolicyDefinition, PolicyVersion, CalculationRun, ToolExecution, RetrievalTrace, ComplianceDecision, ReviewTask, AuditPackage | 7 years (NFR-005, OQ-09) |
| OPS | Scenario, import jobs, notifications | Tenant life plus 90 days |
| EPHEMERAL | Caches, queue messages, idempotency records | TTL per LLD Sections 23 to 25 |

## 14. Open Questions

Owned in the [PRD](PRD.md) Section 36: OQ-03 (vector store), OQ-09 (retention and residency), OQ-13 (decision schema versioning). New here: OQ-15: whether the reporting-period reference date is period end universally or regulation-dependent from day one (modelled as policy-governed above; needs a legal-review ruling under OQ-08).
