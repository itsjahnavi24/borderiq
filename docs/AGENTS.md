# BorderIQ Agents

| Field | Value |
|---|---|
| Title | BorderIQ Agents |
| Version | 1.0 |
| Status | Draft for review |
| Owner | Jahnavi Ralhan |
| Last updated | 2026-08-01 |
| Scope | The small set of justified LLM agents, their contracts, and the explicit rule for why almost everything else is a deterministic workflow, tool, or service instead. Tool contracts live in [TOOL_CALLING](TOOL_CALLING.md); the lifecycle agents operate inside lives in [DECISION_ENGINE](DECISION_ENGINE.md). |
| Related documents | [PRD](PRD.md), [LLD](LLD.md), [DECISION_ENGINE](DECISION_ENGINE.md), [TOOL_CALLING](TOOL_CALLING.md), [SECURITY](SECURITY.md), [ROADMAP](ROADMAP.md) |

Status: everything in this document is Target architecture (Proposed). No agent exists in the current MVP, and by design the platform is complete without any of them: every agent below is an optional enhancement whose absence degrades quality of convenience, never correctness. Requirement anchors: PP-9, AIR-001 to AIR-008, FR-021, FR-024.

## 1. Document Control

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0 | 2026-08-01 | Jahnavi Ralhan | Initial agent specification against PRD v3.0 and TOOL_CALLING v1.0 |

## 2. When a Workflow Beats an Agent

Definition used here: an agent is an LLM-driven loop that plans, selects tools, observes results, and iterates toward an objective. A workflow is a fixed, code-owned sequence that may call the LLM for bounded sub-tasks.

The test for agenthood, applied before anything earns the name: (1) the task genuinely requires runtime planning because the useful step sequence varies with content in ways code cannot cheaply enumerate; (2) every action the loop can take is a registered tool with the LLM-selectable containment of [TOOL_CALLING.md](TOOL_CALLING.md) Section 7; (3) the loop has typed inputs, a schema-validated output, hard stopping conditions, and a deterministic fallback; (4) failure of the agent leaves the platform correct, merely less convenient.

Anything failing the test ships as a workflow, tool, or service (technical decision rule 7). Consequences of this rule, stated plainly:

- The **investigation lifecycle is a workflow**, not an agent: the 16 stages of DECISION_ENGINE Section 3 run in a deterministic default order, always.
- **Calculators, the product resolver, the policy engine, retrieval, evidence validators, and risk scoring are tools and services**, never agents: their correctness is the product.
- The LLM's non-agent roles (narration inside stage 13, extraction of candidate values, classification) are bounded workflow steps governed by AIR-001, AIR-002, and AIR-008, not loops.

Three agents pass the test, plus one optional fourth. There is no framework ambition here: each agent is a small governed loop over the same tool executor everything else uses.

## 3. Shared Runtime Constraints (All Agents)

Every agent: executes tools only through the executor (no bypass; TOOL_CALLING Section 4); sees only its allowlisted, read-only and retrieval tool subset (write and high-impact classes are structurally unreachable, TOOL_CALLING Section 7); runs under per-invocation token, tool-call, and wall-clock budgets that cap rather than overspend (NFR-010); is traced end to end (ToolExecution rows plus an agent-run record with plan, steps, and stop reason); emits schema-validated output or a typed failure, never partial prose (AIR-002); and has a deterministic fallback owned by the calling workflow. Prompt-injection containment per SECURITY Section 11 applies verbatim: retrieved and document text an agent reads is data, and an injected instruction can at worst waste read calls.

## 4. Agent A-1: Investigation Planner

| Field | Value |
|---|---|
| Objective | Given a validated investigation context, propose an execution plan that skips inapplicable optional stages and orders retrieval scopes well, reducing latency and cost against the default plan |
| Implementation status | Proposed (Phase 2, behind a flag; default off) |
| Input | Investigation context summary: resolved entities, requested methodology, evidence-state overview, reporting period |
| Allowed tools | fetch_shipment, fetch_supplier, fetch_installation, validate_evidence_completeness, resolve_policy_version (read-only context gathering only) |
| Prohibited actions | Skipping or reordering mandatory stages (the allowlist of skippable stages is code-owned: currently only redundant reference retrievals and duplicate evidence checks); selecting any tool outside its list; altering any figure, eligibility, or routing outcome; extending its own allowlist |
| Output schema | `{ "plan": [ { "stage": int, "action": "run" or "skip", "reason": string } ], "retrieval_scopes": [ scope objects per TOOL_CALLING retrieve_regulatory_evidence input ], "confidence_note": string }`, schema-validated; a plan violating the mandatory-stage invariant is rejected wholesale |
| Stopping conditions | Plan emitted; or 6 tool calls; or 30 seconds; or budget cap |
| Memory | None beyond the invocation; no cross-investigation state |
| Approval boundaries | None needed: the plan can only remove permitted redundancy; the orchestrator validates every plan against the invariant before adopting it |
| Failure behaviour | Any failure, rejection, or timeout: orchestrator runs the deterministic default plan (LLD Section 10.1); the investigation proceeds identically minus the optimisation |
| Evaluation | Plan-validity rate; latency and cost delta versus default plan on the golden investigation set; zero tolerated invariant violations (release gate) |

## 5. Agent A-2: Regulatory Change Analyst

| Field | Value |
|---|---|
| Objective | Given a structural diff between regulation versions (FR-024), classify the changes, assess impact in plain terms, and draft proposed policy amendments as inert proposals for human review |
| Implementation status | Proposed (Phase 5) |
| Input | Diff report (API_SPEC Section 8 shape), prior impact reports for the document family |
| Allowed tools | retrieve_regulatory_evidence, resolve_policy_version, resolve_cn_code, retrieve_default_emission_value (grounding and cross-reference only) |
| Prohibited actions | Activating, editing, or retiring any policy (activation is not a tool and does not exist on any callable surface, TOOL_CALLING Section 12); creating review tasks directly (the change-intelligence workflow creates the task and attaches this agent's output); asserting legal conclusions (drafts are labelled proposals with provenance, AIR-010) |
| Output schema | `{ "change_classification": [ { "path": string, "kind": enum, "summary": string } ], "impact_assessment": { "categories": [], "cn_codes": [], "affected_policies": [], "affected_open_decisions": int, "narrative": string }, "policy_proposals": [ { "definition_key": string, "proposed_rule_delta": object, "derived_from_fragments": [], "rationale": string } ] }` |
| Stopping conditions | Output emitted; or 15 tool calls; or 5 minutes; or budget cap |
| Memory | Read access to prior impact reports for the same document family; writes nothing outside its output |
| Approval boundaries | Everything: no proposal has any effect until a qualified reviewer approves it through the Phase 4 workflow (OQ-08); the agent's output is advisory input to a human task |
| Failure behaviour | Change-intelligence workflow falls back to the mechanical diff plus relational impact mapping (the Phase 5 floor); the review task is created regardless, with a note that analyst drafting was unavailable |
| Evaluation | Classification agreement with reviewer outcomes; proposal acceptance and edit-distance rates; unsupported-claim rate in narratives (must cite fragments); reviewer time saved versus mechanical-diff-only baseline |

## 6. Agent A-3: Audit Package Narrator

| Field | Value |
|---|---|
| Objective | Given a finalised decision and its complete trace, produce the human-readable narrative sections of the audit package: what was decided, on what evidence, under which rules, with what review history |
| Implementation status | Proposed (Phase 4) |
| Input | The decision object and trace references (never live systems: replay discipline, DECISION_ENGINE Section 11) |
| Allowed tools | retrieve_regulatory_evidence (only to quote already-cited fragments at greater length; scope pinned to the decision's supporting_sources) |
| Prohibited actions | Introducing any numeral not present in the decision object (the AIR-001 numeric guard runs on its output); citing anything outside the decision's supporting_sources; characterising reviewer actions beyond the recorded rationale; softening or reweighing risk and confidence fields |
| Output schema | `{ "sections": { "decision_summary": string, "methodology_and_eligibility": string, "evidence_basis": string, "calculation_explanation": string, "review_history": string }, "citations_used": [ fragment refs ] }`; every regulatory claim sentence must map to a citation (AIR-003 post-check) |
| Stopping conditions | Sections emitted; or 8 tool calls; or 2 minutes; or budget cap |
| Memory | None; each package narrated fresh from its trace (reproducibility of tone is not required; reproducibility of facts is guaranteed by the guards) |
| Approval boundaries | The narrated package is part of the export that requires the FR-030 human trigger; the narrative itself needs no separate approval because it cannot introduce facts |
| Failure behaviour | Audit package exports with structured sections only (tables from the trace) and narrative marked unavailable; export is never blocked by narration (LLD Section 38 pattern) |
| Evaluation | Citation completeness; numeric-guard rejection rate (target zero at steady state); auditor usefulness rating in pilots; structured-only fallback exercised in degradation tests |

## 7. Agent A-4 (Optional): Review Triage

| Field | Value |
|---|---|
| Objective | Order the review queue by materiality, deadline pressure, and case similarity, and draft a two-paragraph case summary per task so reviewers start oriented |
| Implementation status | Proposed (Phase 4, optional; ship only if pilot queue depth justifies it, else defer) |
| Input | Open review tasks with decision references |
| Allowed tools | fetch_shipment, validate_evidence_completeness (context only) |
| Prohibited actions | Approving, rejecting, or requesting information (reviewer-role actions are API-gated to humans, API_SPEC Section 11); hiding or dropping tasks (ordering only; every task remains visible); writing to any record |
| Output schema | `{ "ordering": [ { "review_task_id": ref, "priority_basis": string } ], "summaries": { review_task_id: string } }` |
| Stopping conditions | Output emitted; or 10 tool calls; or 60 seconds; or budget cap |
| Memory | None |
| Approval boundaries | None applicable: output is presentation-layer assistance |
| Failure behaviour | Queue renders in default order (age, then materiality from the decision fields); summaries absent |
| Evaluation | Reviewer-reported usefulness; time-to-first-action delta; zero incidents of hidden tasks (invariant test) |

## 8. What Deliberately Does Not Exist

For the record, agents considered and rejected under the Section 2 test: a "calculation agent" (calculation is a deterministic tool; an agent adds nondeterminism to the one place it is forbidden); a "compliance decision agent" (the decision is the workflow's output; an agent deciding eligibility would violate FR-014); an "auto-remediation agent" acting on review outcomes (side effects belong to humans and rule-triggered workflows, AIR-010); a "browsing agent" for regulation discovery (ingestion is allowlisted acquisition, SECURITY Section 10; free browsing reopens the SSRF and poisoning classes deliberately closed). Any future agent proposal must pass the same test and be added here before implementation.

## 9. Evaluation Summary

Per-agent metrics above roll into the LLD Section 33 harness. Two global gates: no agent may ever cause an invariant violation in release tests (mandatory stages, numeric guard, citation scope, task visibility), and every agent's disabled-state fallback is exercised in the NFR-009 degradation suite, because an optional component that has never run disabled is not optional.

## 10. Open Questions

Owned in the [PRD](PRD.md): OQ-08 (approver qualification, bounds A-2's review loop), OQ-12 (materiality inputs to A-4 ordering). This document introduces none.
