# BorderIQ Security

| Field | Value |
|---|---|
| Title | BorderIQ Security |
| Version | 1.0 |
| Status | Draft for review |
| Owner | Jahnavi Ralhan |
| Last updated | 2026-08-01 |
| Scope | Threat model, security controls, and the gap between the verified current MVP and production requirements. Trust-boundary diagram in [ARCHITECTURE](ARCHITECTURE.md) Section 16; tool-layer containment in [TOOL_CALLING](TOOL_CALLING.md); retention in [DATA_MODEL](DATA_MODEL.md) Section 13. |
| Related documents | [PRD](PRD.md), [LLD](LLD.md), [ARCHITECTURE](ARCHITECTURE.md), [DATA_MODEL](DATA_MODEL.md), [DECISION_ENGINE](DECISION_ENGINE.md), [TOOL_CALLING](TOOL_CALLING.md), [API_SPEC](API_SPEC.md), [DEPLOYMENT](DEPLOYMENT.md), [ROADMAP](ROADMAP.md) |

Posture statement: BorderIQ handles commercially sensitive supply-chain data and produces figures with financial and regulatory consequences. The security design follows OWASP application-security and OWASP LLM guidance as engineering references. No certification (ISO 27001, SOC 2, or otherwise) is claimed anywhere in this suite; certifications are future work listed in Section 17. Requirement anchors: NFR-005, NFR-006, NFR-007, AIR-006, AIR-010.

## 1. Document Control

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0 | 2026-08-01 | Jahnavi Ralhan | Initial security design; current-MVP section grounded in verified code |

## 2. Security Principles

1. **Fail closed** (PP-6): optional and high-risk functionality degrades to safe, shape-preserving behaviour; permissions are deny-by-default.
2. **Untrusted by position, not by content**: everything arriving from outside a trust boundary (client input, uploaded files, retrieved document text, tool outputs from probabilistic sources) is data to validate, never instructions to follow (AIR-006).
3. **Least privilege everywhere**: users get roles, services get scoped identities, tools get per-tool permissions, the graph gets read-only request credentials.
4. **Humans gate impact** (PP-7, AIR-010): policy activation, declaration export, and material decisions require recorded human approval; this is a security control, not only a product feature.
5. **Everything attributable**: security-relevant actions land in an append-only audit log with actor identity (Section 13).

## 3. Assets

| Asset | Sensitivity | Primary threats |
|---|---|---|
| Supplier and shipment data | Commercially sensitive; reveals sourcing, volumes, costs | Exfiltration, cross-tenant leakage |
| Emissions evidence and verifier documents | Sensitive; audit-bearing | Tampering, poisoning, loss |
| Compliance decisions and traces | Sensitive; legally consequential | Tampering, unauthorised disclosure, replay divergence |
| Policy versions | High integrity requirement | Unauthorised activation, silent drift |
| Regulatory corpus | Public content, high integrity requirement | Poisoning, version confusion |
| Credentials and API keys | Critical | Theft, leakage through repos or logs |
| Model prompts and tool registry | Integrity-sensitive configuration | Tampering to enable injection or tool abuse |

## 4. Actors and Trust Model

Legitimate actors: analyst, reviewer, auditor, tenant admin, platform operator, service identities (orchestrator, ingestion builder, projection builder), and the external model provider. Threat actors considered: anonymous internet attackers; authenticated cross-tenant attackers (a hostile or compromised customer); malicious document suppliers (poisoned uploads or manipulated supplier declarations); a compromised model provider response; and negligent insiders. Nation-state adversaries and physical attacks are out of scope for this stage and inherited from the cloud provider's controls.

## 5. Trust Boundaries

Four zones per [ARCHITECTURE.md](ARCHITECTURE.md) Section 16: Zone 0 untrusted (browsers, uploads), Zone 1 application, Zone 2 data stores, Zone 3 external providers (model API, EU sources). Every crossing has a named control: TLS plus authentication into Zone 1; validation and scanning between Zone 0 artefacts and Zone 2 storage; minimum-necessary context and no secrets outbound to Zone 3; checksummed allowlisted ingestion inbound from EU sources; retrieved text re-entering Zone 1 marked untrusted (Section 11).

## 6. Identity and Access

- **Authentication**: OIDC against the customer identity provider (SSO) for enterprise tenants; platform-issued identities only for early pilots. Tokens are short-lived; refresh handled by the IdP. Introduced Phase 3 (nothing durable exists before then to protect).
- **RBAC baseline**: roles analyst (investigate, ask, import), reviewer (analyst plus review actions and audit-package generation, export approval), auditor (read decisions, traces, packages; no mutation), tenant admin (settings, users, mappings, webhooks), platform operator (infrastructure, no tenant data by default, break-glass audited). Role catalogue is the permission vocabulary used by [TOOL_CALLING.md](TOOL_CALLING.md) and [API_SPEC.md](API_SPEC.md).
- **ABAC extensions** (Phase 9): attribute conditions on top of roles where tenants need them, for example restricting an analyst to one Organisation's shipments. Roles stay the base; attributes narrow, never widen.
- **Service identities**: separate credentials per worker class; the projection builder can write the graph, request paths cannot; ingestion can write object storage raw prefixes, nothing else can.

## 7. Tenant Isolation

NFR-007 verbatim: no query path may return another tenant's data, enforced at every layer and verified by automated tests. Mechanisms: `tenant_id` mandatory on transactional rows with repository-layer enforcement (tenant-less access refuses, LLD Section 21); tenant-scoped vector namespaces when tenant documents are ever indexed; tenant prefixes in object storage with per-prefix access policies; graph tenant predicates by construction ([KNOWLEDGE_GRAPH.md](KNOWLEDGE_GRAPH.md) Section 10); per-tenant quotas (Section 12); per-tenant webhook secrets. The isolation test suite (API, SQL, retrieval, storage, graph) is a CI release gate.

## 8. Data Protection

- **Encryption in transit**: TLS 1.2 or higher everywhere, including service-to-store links.
- **Encryption at rest**: enabled on every store (managed database, object storage, indexes, graph, backups); cloud-managed keys initially, customer-managed keys as a Phase 9 option where contracts require.
- **Secrets**: a secrets manager from the first deployed environment; nothing secret in code, images, or repository history. Local development may use `.env`, which is gitignored, with `.env.example` committed. Rotation is routine and documented; any secret that leaves a developer machine (in an archive, a chat, a log) is treated as burned and rotated immediately, a rule applied in practice during this project when the MVP key travelled inside a zip.
- **Retention and deletion**: per DATA_MODEL Section 13 (AUDIT 7 years default, OQ-09 pending); tenant deletion removes tenant-scoped data across all stores after the retention window; deletion jobs produce completion evidence.
- **Data residency**: region pinning deferred to Phase 9 and OQ-09; the design keeps all stores region-scopable.

## 9. Model-Provider Controls

- API-based access under enterprise terms; no training on submitted data is the required contractual baseline (verify per provider; OQ-10 for stricter requirements such as in-VPC inference).
- Minimum-necessary context: prompts carry retrieved fragments and typed investigation context, never credentials, never full customer datasets, never other tenants' content.
- Model and prompt versions pinned and recorded per trace (AIR-007); provider changes pass evaluation gates before rollout.
- Provider responses are untrusted structured data: schema-validated (AIR-002), numerically guarded (AIR-001), citation-checked (AIR-003).
- Provider outage is a degradation case, not a security event (LLD Section 38); provider compromise is treated in Section 11 as manipulated output and is contained by the same validation layers.

## 10. Content Integrity: Corpus, Supplier Data, Uploads

- **Regulatory corpus**: acquisition only from the authoritative allowlist; immutable raw storage with checksums and provenance; versioned, supersession-linked, never edited in place (FR-022). A poisoned or spoofed regulation source is countered by the allowlist plus checksum provenance; a compromised authoritative source itself is detected by cross-checking published checksums where available and by the human review step in change intelligence (FR-024).
- **Supplier data**: enters with data-quality state `unverified` and gains standing only through validation (FR-013); a malicious supplier declaration can therefore inflate nothing on its own, because eligibility requires verification evidence.
- **Uploads**: content-type validation, size limits, antivirus scanning, and archive-bomb protection before any file becomes referenceable; files are stored under tenant prefixes and served back only through signed, expiring URLs; uploaded documents never enter the regulatory corpus (they are evidence, a separate class with separate retrieval scope, if indexed at all).

## 11. LLM Threat Model (OWASP LLM Alignment)

| Risk (OWASP LLM) | BorderIQ exposure | Controls |
|---|---|---|
| Prompt injection, direct | User asks crafted questions | Instruction and data separation in prompts; output schema validation (AIR-002); no user text ever becomes tool policy |
| Prompt injection, indirect | Instructions embedded in PDFs or retrieved fragments | Evidence delivered as delimited untrusted data (AIR-006); injected tool proposals hit the offered-subset rejection and are traced ([TOOL_CALLING.md](TOOL_CALLING.md) Section 14); injection corpus is a release gate |
| Insecure output handling | Model output reaching users or systems | Schema validation before consumption; numeric guard (AIR-001); citation check (AIR-003); no model output is executed, evaluated, or templated into queries |
| Training-data or document poisoning | Corpus manipulation | Section 10 corpus controls; no fine-tuning on customer or corpus data in scope |
| Excessive agency and tool abuse | Model triggering side effects | Write and high-impact tools outside every LLM-selectable subset; executor-enforced permissions and approval gates; deterministic default plan runs with the planner disabled |
| Sensitive information disclosure | Cross-tenant or secret leakage via responses | Tenant-scoped retrieval; secrets never in prompts; trace redaction (TOOL_CALLING Section 13); tenant isolation suite |
| Denial of service and cost abuse | Prompt flooding, token exhaustion | Rate limits (Section 12); per-investigation token and retrieval budgets that cap rather than overspend (NFR-010) |
| Supply chain | Model, library, and index dependencies | Pinned dependencies (now in place in the MVP); provider version pinning with eval gates; dependency scanning in CI (Phase 3) |
| Overreliance | Users treating narration as authority | Figures render only from engine outputs; confidence dimensions instead of a reassuring single score (AIR-005); decision-support disclaimer (PRD Section 26) |

## 12. Application Threat Controls

- **SSRF**: structurally closed; no generic URL-fetch capability exists anywhere, including as a tool (TOOL_CALLING Section 13); ingestion fetches only allowlisted sources; webhook deliveries go only to admin-registered endpoints, with private-address ranges refused at registration.
- **Code execution**: no model output or user input is executed; policy rules are declarative data evaluated by a closed interpreter (LLD Section 13); no dynamic imports of tenant content.
- **Injection (SQL, template)**: parameterised queries only; graph queries are parameterised templates (KNOWLEDGE_GRAPH Section 10); user strings HTML-escaped at render (already implemented and verified in the MVP frontend).
- **Data exfiltration**: egress from application workers limited to declared destinations (stores, provider API, webhooks); audit-package downloads via expiring signed URLs bound to the requesting identity; large exports logged and rate-limited.
- **Cross-tenant retrieval**: Section 7; additionally, retrieval requests carry tenant scope injected by the executor, never caller-supplied (TOOL_CALLING Section 13).
- **Denial of service**: per-identity and per-tenant rate limits at the API layer (429 with retry semantics, API_SPEC Section 3); import size ceilings; traversal depth limits and timeouts on graph queries; queue backpressure with visible status rather than unbounded acceptance.

## 13. Security Audit Logging

Two streams (LLD Section 30): decision traces (product-facing, FR-020) and the security audit log: authentication events, authorisation denials, role and settings changes, policy activations, approval grants, exports and downloads, break-glass operator access, webhook registration changes, secret rotations. Properties: append-only, clock-synced, actor-attributed, retained under AUDIT class, and shipped off the application host so host compromise cannot silently erase history. Alerting on: repeated authorisation denials, cross-tenant test canaries, injection-rejection spikes, replay divergence (DECISION_ENGINE Section 11), and drift-check failures.

## 14. Human Approval as a Security Control

The approval gates of AIR-010 and TOOL_CALLING Section 12 bound the blast radius of both model manipulation and account compromise: a fully hijacked analyst session still cannot activate policy, export declarations, or execute high-impact tools without a second, recorded, reviewer-role approval. Approval actions themselves are strong-authenticated (re-auth or step-up at the IdP where supported) and land in the security log.

## 15. Incident Response

Phase 3 onward: a written runbook per class (credential leak, cross-tenant exposure, corpus integrity failure, replay divergence, provider incident, data-loss event) with severity levels, an on-call owner, tenant-notification obligations and timelines (contract-driven, OQ-09 jurisdictions), and post-incident review feeding the risk register. Two incident classes are already exercised in miniature by this project: credential rotation (Section 8) and replay divergence (defined as an alerting integrity incident before the replay feature even ships).

## 16. Secure Defaults Checklist

Deny-by-default authorisation; anonymous access to nothing once storage exists; debug mode off outside local development, and error details gated on debug (implemented in the current code); demo and synthetic data labelled (PP-10); new tools enter the registry with the narrowest class that fits and no LLM-selectable exposure unless justified; new endpoints require explicit role annotations to route; graph and other optional subsystems ship disabled until configured; webhooks require explicit event selection, never wildcard by default.

## 17. Current MVP Assessment (Verified) and Production Requirements

Grounded in the code inspected on 2026-08-01, including the fix set applied the same day.

**Resolved in current code:**

| Item | Status |
|---|---|
| Error details (`str(exc)`) returned to clients | Fixed: details only in debug mode |
| Crash at import without API key (eager clients) | Fixed: lazy service and client construction; clean 503 with guidance |
| No `.gitignore`, `.env` at risk of commit | Fixed: gitignore covering secrets, index, PDFs; `.env.example` added; key rotated after archive exposure |
| Unpinned dependencies (empty requirements.txt) | Fixed: pinned |
| Hardcoded status and knowledge-base responses | Fixed: live counts from index and manifest |
| Unstable chunk IDs | Fixed: stable per-page IDs |
| No assertion tests | Partially fixed: calculator covered by pytest; RAG paths still untested |

**Open gaps, accepted for the current stage, closed at the stated phase:**

| Gap | Risk now | Closed by |
|---|---|---|
| No authentication or authorisation on any route | Anyone reaching the host can use every endpoint | Phase 3 (identity, RBAC) before any shared deployment; until then, local-only operation |
| Flask debug server (`debug=True`, dev server) | Not hardened for exposure | Production WSGI server and config split, [DEPLOYMENT.md](DEPLOYMENT.md), Phase 3 |
| Frontend assets from public CDNs | Availability and integrity dependency | Vendor and integrity-pin assets, Phase 3 |
| No rate limiting or quotas | Cost and availability abuse if exposed | Phase 3 API layer |
| No upload path hardening (no uploads exist yet) | None yet | Section 10 controls land with the evidence API, Phase 3 to 4 |
| Secrets in local `.env` only | Acceptable for local development exclusively | Secrets manager from first deployed environment |
| No security audit log | Actions unattributable | Phase 3 with identity |
| Single-tenant assumption throughout | By design at this stage | Phase 9 |

**Rule until Phase 3 lands**: the MVP runs locally or behind a private network only; it is not internet-exposable as-is, and the README will say so plainly.

## 18. Open Questions

Owned in the [PRD](PRD.md): OQ-08 (who qualifies as a policy approver), OQ-09 (residency, retention, notification obligations), OQ-10 (provider constraints, in-VPC inference). No new questions raised by this document.
